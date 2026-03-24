# NanoClaw Architecture Overview

A complete reference for how the system is structured, how agents work, and what tools are available — with comparisons to Microsoft AutoGen / Semantic Kernel for context.

---

## Table of Contents

1. [Directory Structure](#1-directory-structure)
2. [System Architecture](#2-system-architecture)
3. [Message Flow](#3-message-flow)
4. [How Agents Are Defined](#4-how-agents-are-defined)
5. [Agent Tools Reference](#5-agent-tools-reference)
6. [Container Mounts & Security](#6-container-mounts--security)
7. [Channel System](#7-channel-system)
8. [Scheduled Tasks & IPC](#8-scheduled-tasks--ipc)
9. [Database Schema](#9-database-schema)
10. [Key Configuration Values](#10-key-configuration-values)
11. [Claude Agent SDK vs Microsoft AutoGen/Semantic Kernel](#11-claude-agent-sdk-vs-microsoft-autogensemantic-kernel)

---

## 1. Directory Structure

```
memclaw/
├── src/                    # Node.js orchestrator (TypeScript)
├── container/              # Docker agent runtime
│   ├── agent-runner/       # Claude Agent SDK wrapper (runs inside container)
│   │   └── src/
│   │       ├── index.ts          # Main agent loop, query() invocation
│   │       └── ipc-mcp-stdio.ts  # Custom MCP tool server
│   ├── skills/             # Agent-accessible skill files (e.g. agent-browser)
│   ├── Dockerfile
│   └── build.sh
├── groups/                 # Per-group memory and files
│   ├── global/             # Shared memory (all agents read this)
│   │   └── CLAUDE.md
│   ├── main/               # Primary admin group (no trigger needed)
│   │   └── CLAUDE.md
│   └── {channel}_{name}/   # Per-group folders (auto-created on registration)
│       ├── CLAUDE.md
│       ├── conversations/  # Archived chat transcripts
│       └── logs/
├── data/
│   ├── ipc/                # Inter-process communication files
│   └── sessions/           # Claude session IDs per group
├── store/
│   ├── messages.db         # SQLite database
│   └── auth/               # WhatsApp credentials
├── .claude/skills/         # Claude Code skills (/setup, /add-whatsapp, etc.)
├── docs/                   # Documentation (you are here)
└── src/
    ├── index.ts            # Orchestrator: message loop, agent invocation
    ├── channels/           # Channel system (WhatsApp, Telegram, Slack, ...)
    ├── container-runner.ts # Spawns containers with correct mounts
    ├── container-runtime.ts# Runtime abstraction (Docker / Apple Container / WSL)
    ├── ipc.ts              # IPC file watcher and processor
    ├── router.ts           # Message formatting (XML) and outbound routing
    ├── config.ts           # Env config: paths, timeouts, trigger pattern
    ├── db.ts               # SQLite operations
    ├── task-scheduler.ts   # Cron/interval/once task runner
    ├── group-queue.ts      # Concurrency limiter (max 5 containers)
    ├── credential-proxy.ts # HTTP proxy that injects API keys at runtime
    └── types.ts            # TypeScript interfaces
```

---

## 2. System Architecture

NanoClaw is a **single Node.js process** that:

1. Connects to one or more messaging channels (WhatsApp, Telegram, Slack, Discord, Gmail)
2. Stores all messages in SQLite
3. Polls every 2 seconds for messages that match a trigger (`@Andy`)
4. Spawns an isolated Docker container per group to run the Claude agent
5. Streams the agent's output back to the channel

There is **one agent type**: the Claude Agent SDK running in a container. What differentiates each agent is its **group folder** (`cwd`) and **session ID** (memory).

```
Channel (WhatsApp/Telegram/Slack)
  │
  ▼ onMessage() → SQLite
  │
  ▼ 2s poll loop → trigger check (@Andy?)
  │
  ▼ GroupQueue (max 5 concurrent containers)
  │
  ▼ docker run (container/agent-runner)
      ├── stdin: JSON { prompt, sessionId, groupFolder, chatJid, isMain }
      ├── mounts: group folder, global, ipc, (project root if main)
      └── stdout: ---NANOCLAW_OUTPUT_START--- { result, newSessionId } ---NANOCLAW_OUTPUT_END---
  │
  ▼ Strip <internal> tags → channel.sendMessage()
```

---

## 3. Message Flow

### Step-by-step

1. **Channel ingests message** → `onMessage(chatJid, message)` fires
2. **Stored in SQLite** `messages` table with timestamp
3. **Poll loop** (every 2s) queries messages newer than `lastTimestamp`, grouped by chat
4. **Trigger check**: non-main groups require `@Andy` prefix; main group processes all messages
5. **Sender allowlist** checked (if configured) — drop or ignore denied senders
6. **Message formatting** — converted to XML by `router.ts`:
   ```xml
   <context timezone="America/New_York" />
   <messages>
     <message sender="John" time="09:42:15">@Andy what's the weather?</message>
     <message sender="Andy" time="09:42:20">It's sunny, 72°F</message>
   </messages>
   ```
7. **Container spawned** with JSON payload via stdin
8. **Agent runs** — Claude Agent SDK processes prompt, calls tools, produces output
9. **Output streamed back** — parsed from stdout markers, `<internal>` tags stripped
10. **Sent to channel** — `channel.sendMessage(chatJid, text)`

---

## 4. How Agents Are Defined

### The short answer

There is **no agent class**. An "agent" = `query()` call + `cwd` directory + `sessionId`.

### The `query()` call (container/agent-runner/src/index.ts)

```typescript
for await (const message of query({
  prompt: stream,
  options: {
    cwd: '/workspace/group',           // agent identity = CLAUDE.md in this dir
    resume: sessionId,                 // persistent memory across invocations
    systemPrompt: globalClaudeMd,      // global/shared instructions appended
    allowedTools: [ ... ],             // what the agent can do
    permissionMode: 'bypassPermissions',
    mcpServers: { nanoclaw: { ... } }, // custom MCP tool server
  }
}))
```

### What defines each agent's "personality"

| Parameter | What it controls | Where it comes from |
|-----------|-----------------|---------------------|
| `cwd` | Working directory; SDK auto-reads `CLAUDE.md` here | `groups/{name}/` |
| `systemPrompt` | Appended global context | `groups/global/CLAUDE.md` |
| `resume` | Session ID = conversation history | SQLite `sessions` table |
| `allowedTools` | What tools the agent can use | Hardcoded in agent-runner |
| `mcpServers` | Custom tools (send message, schedule task, etc.) | `ipc-mcp-stdio.ts` |

### Creating a new agent

No code changes needed. Just:
1. Create `groups/{channel}_{name}/CLAUDE.md` with the agent's persona/instructions
2. Register the group (via main agent's `register_group` tool or `/setup`)

The next message to that group will spawn a container with that CLAUDE.md as the system prompt.

### Group types

| Group | Persona | Special privileges |
|-------|---------|-------------------|
| `groups/main/` | Andy (admin) | No trigger needed; reads full project root; can register groups; sees all tasks |
| `groups/global/` | Chef (shared) | Not a real agent — its CLAUDE.md is injected into all other agents as context |
| `groups/{any}/` | Custom | Needs `@Andy` trigger; isolated to own folder; only sees own tasks |

---

## 5. Agent Tools Reference

### Built-in tools (always available, no setup)

These are declared in `allowedTools` and implemented natively by the Claude Agent SDK:

| Tool | Description |
|------|-------------|
| `Bash` | Run shell commands inside the container sandbox |
| `Read` | Read a file |
| `Write` | Write/create a file |
| `Edit` | Edit a file (diff-based) |
| `Glob` | Find files by pattern |
| `Grep` | Search file contents with regex |
| `WebSearch` | Search the web |
| `WebFetch` | Fetch a URL |
| `Task` | Spawn a parallel sub-task (sub-agent) |
| `TaskOutput` | Read output from a running sub-task |
| `TaskStop` | Stop a running sub-task |
| `TeamCreate` | Create a multi-agent team (agent swarm) |
| `SendMessage` | Send a message to another agent in a team |
| `TodoWrite` | Manage a to-do list |
| `ToolSearch` | Discover available tools dynamically |
| `Skill` | Invoke a Claude Code skill |
| `NotebookEdit` | Edit Jupyter notebooks |

### Custom tools via MCP (defined in ipc-mcp-stdio.ts)

These are registered as `mcp__nanoclaw__*` and communicate with the host via IPC files:

| Tool | Description | Who can use |
|------|-------------|-------------|
| `send_message` | Send a message to the group immediately (mid-task progress updates) | All agents |
| `schedule_task` | Create a cron / interval / once recurring task | All agents |
| `list_tasks` | List scheduled tasks | All (filtered to own group unless main) |
| `pause_task` | Pause a scheduled task | All (own tasks only unless main) |
| `resume_task` | Resume a paused task | All |
| `cancel_task` | Delete a scheduled task | All |
| `update_task` | Modify an existing task's prompt or schedule | All |
| `register_group` | Register a new chat/group | Main only |

#### `schedule_task` context modes

- **`group`**: task runs with existing chat history and group memory — use when the task needs to know prior conversation context
- **`isolated`**: fresh session, no history — use for self-contained tasks (include all context in the prompt)

#### `schedule_task` schedule formats

```
cron:     "0 9 * * 1-5"       → weekdays at 9am local time
interval: "3600000"            → every 1 hour (milliseconds)
once:     "2026-06-01T15:30:00" → run once at this local time (no Z suffix!)
```

### Mounted skill: agent-browser

Available to all agents via `Bash`. Provides headless Chromium/Playwright browser automation:
- Located at `/container/skills/agent-browser/`
- Used for scraping, form filling, screenshots, web automation

---

## 6. Container Mounts & Security

### Mount structure

```
/workspace/
  ├── group/     ← read-write  (this group's own folder)
  ├── global/    ← read-only   (non-main agents only — shared CLAUDE.md)
  ├── project/   ← read-only   (main agent only — full repo access)
  ├── extra/     ← optional    (additional mounts from containerConfig)
  └── ipc/       ← read-write  (messaging + task scheduling)
      ├── messages/      ← agent writes here to send messages
      ├── tasks/         ← agent writes here to schedule/manage tasks
      ├── input/         ← host writes follow-up messages here
      ├── available_groups.json  ← list of all groups (main only)
      └── current_tasks.json     ← scheduled tasks (filtered by group)
```

### Security model

- **Credentials are never mounted** into containers — the host's `credential-proxy` (HTTP, port 3001) intercepts API calls and injects real keys at runtime
- **Mount allowlist** lives at `~/.config/nanoclaw/mount-allowlist.json` (outside container, tamper-proof)
- **Sender allowlist** at `~/.config/nanoclaw/sender-allowlist.json` — `drop` mode (never stored) or `trigger` mode (stored but ignored)
- Containers run as non-root `node:1000` user
- Groups are isolated — a non-main group cannot read another group's folder

### Additional mounts (per-group config)

Groups can have extra mounts configured in the database (`containerConfig.additionalMounts`):
```json
{
  "additionalMounts": [
    { "hostPath": "~/projects/webapp", "containerPath": "webapp", "readonly": false }
  ]
}
```

---

## 7. Channel System

### Architecture

- **Factory/registry pattern**: each channel exports a factory function that self-registers on import
- **WhatsApp is bundled** in core; all other channels are installed as skills (separate git branches)
- All channels implement the same `Channel` interface:

```typescript
interface Channel {
  name: string;
  connect(): Promise<void>;
  sendMessage(jid: string, text: string): Promise<void>;
  isConnected(): boolean;
  ownsJid(jid: string): boolean;
  disconnect(): Promise<void>;
  setTyping?(jid: string, isTyping: boolean): Promise<void>;
  syncGroups?(force: boolean): Promise<void>;
}
```

### Available channels

| Channel | How to install | JID format |
|---------|---------------|------------|
| WhatsApp | Bundled | `1234567890@s.whatsapp.net` / `...@g.us` |
| Telegram | `/add-telegram` | `tg:-1001234567890` |
| Slack | `/add-slack` | `slack:C1234567890` |
| Discord | `/add-discord` | `dc:1234567890123456` |
| Gmail | `/add-gmail` | `gmail:user@example.com` |

---

## 8. Scheduled Tasks & IPC

### Task types

| Type | `schedule_value` format | Example |
|------|------------------------|---------|
| `cron` | Standard cron expression | `"0 9 * * 1-5"` |
| `interval` | Milliseconds as string | `"3600000"` |
| `once` | Local ISO timestamp (no Z) | `"2026-06-01T09:00:00"` |

### IPC file protocol

Agents communicate with the host by writing JSON files to `/workspace/ipc/`:

**Send a message:**
```json
// /workspace/ipc/messages/{timestamp}-{random}.json
{
  "type": "message",
  "chatJid": "120363336345536173@g.us",
  "text": "Hello from the agent",
  "sender": "Researcher"
}
```

**Schedule a task:**
```json
// /workspace/ipc/tasks/{timestamp}-{random}.json
{
  "type": "schedule_task",
  "taskId": "task-...",
  "prompt": "Send daily briefing",
  "schedule_type": "cron",
  "schedule_value": "0 9 * * 1-5",
  "context_mode": "isolated",
  "targetJid": "120363336345536173@g.us"
}
```

Other IPC task types: `pause_task`, `resume_task`, `cancel_task`, `update_task`, `register_group`.

### Task scheduler loop

- Polls every 60 seconds
- Computes `next_run` using cron-parser or interval arithmetic
- Spawns a container when a task is due
- Logs duration, status, and output to `task_run_logs` table

---

## 9. Database Schema

SQLite database at `store/messages.db`.

```sql
-- All groups/chats across all channels
chats (jid, name, last_message_time, channel, is_group)

-- Full message history for agent context
messages (id, chat_jid, sender, sender_name, content, timestamp, is_from_me, is_bot_message)

-- Groups registered to receive agent responses
registered_groups (jid, name, folder, trigger_pattern, added_at, container_config, requires_trigger)

-- Scheduled tasks
scheduled_tasks (
  id, group_folder, chat_jid, prompt,
  schedule_type, schedule_value,
  next_run, last_run, last_result, status,
  created_at, context_mode
)

-- Task execution audit log
task_run_logs (task_id, run_at, duration_ms, status, result, error)

-- Persists lastTimestamp per group for message polling
router_state (key, value)

-- Claude session IDs for persistent memory
sessions (group_folder, session_id)
```

---

## 10. Key Configuration Values

Set via `.env` file:

| Variable | Default | Description |
|----------|---------|-------------|
| `ASSISTANT_NAME` | `Andy` | Trigger name (`@Andy`) |
| `POLL_INTERVAL` | `2000` ms | How often to check for new messages |
| `SCHEDULER_POLL_INTERVAL` | `60000` ms | How often to check for due tasks |
| `CONTAINER_TIMEOUT` | `1800000` ms (30 min) | Max container lifetime |
| `IDLE_TIMEOUT` | `1800000` ms (30 min) | Kill container after this idle period |
| `MAX_CONCURRENT_CONTAINERS` | `5` | Max simultaneously running containers |
| `CREDENTIAL_PROXY_PORT` | `3001` | Port for credential injection proxy |
| `TZ` | system default | Timezone for cron scheduling |

---

## 11. Claude Agent SDK vs Microsoft AutoGen/Semantic Kernel

### Defining an agent

**Microsoft (AutoGen/Semantic Kernel):**
```python
agent = AssistantAgent(
    name="Andy",
    system_message="You are a helpful assistant...",
    llm_config={"model": "gpt-4", "tools": [my_tool]}
)
```

**Claude Agent SDK (NanoClaw):**
```typescript
// No agent class. Just call query() with a working directory.
query({
  prompt: messages,
  options: {
    cwd: '/workspace/group',   // CLAUDE.md here = system message
    resume: sessionId,          // = conversation history
    allowedTools: [...],
    mcpServers: { ... },
  }
})
```

### Concept mapping

| Microsoft concept | Claude Agent SDK equivalent |
|-------------------|-----------------------------|
| `AssistantAgent(name=..., system_message=...)` | `CLAUDE.md` file in the `cwd` directory |
| `llm_config.tools = [fn1, fn2]` | `allowedTools` (built-ins) + MCP servers |
| Explicit message history passed in | `resume: sessionId` — SDK manages it |
| `GroupChat` / `AgentTeam` | `TeamCreate` + `SendMessage` built-in tools |
| Python `@tool` decorator | MCP server tool definition (protocol-agnostic) |
| Multiple agent objects | Multiple groups, each with own `cwd` + session |
| Agent loop written by you | Handled internally by `query()` |

### Key mental model shift

In Microsoft frameworks, **you build agents** — instantiate classes, wire tools, manage the loop.

In Claude Agent SDK, **you configure Claude** — the model itself is the agent. You control it through:
- **What it knows** → `CLAUDE.md` (persona, instructions, memory)
- **What it can do** → `allowedTools` + MCP servers
- **What it remembers** → `sessionId` (SDK-managed conversation history)
- **Where it works** → `cwd` (filesystem sandbox)

There is no agent loop you write. `query()` handles tool calls → observe → next action → final answer internally.

### Tools: function decoration vs MCP

**Microsoft (Semantic Kernel):**
```python
@kernel_function(description="Send a message")
def send_message(text: str) -> str:
    ...
```

**Claude (MCP server in ipc-mcp-stdio.ts):**
```typescript
server.tool(
  'send_message',
  "Send a message to the user or group immediately while you're still running.",
  { text: z.string(), sender: z.string().optional() },
  async (args) => {
    writeIpcFile(MESSAGES_DIR, { type: 'message', chatJid, text: args.text });
    return { content: [{ type: 'text', text: 'Message sent.' }] };
  }
);
```

The MCP server runs as a **separate process** (stdio transport). The SDK communicates with it via JSON-RPC. This means tools can be written in any language and reused across any MCP-compatible agent framework.

---

## Quick Reference: Adding Capabilities

| Goal | How |
|------|-----|
| Change agent persona | Edit `groups/{name}/CLAUDE.md` |
| Add a new agent (group) | Create `groups/{channel}_{name}/CLAUDE.md` + register via main agent |
| Add a new custom tool | Add a `server.tool(...)` entry in `container/agent-runner/src/ipc-mcp-stdio.ts` |
| Add a new channel | Run the corresponding skill (`/add-telegram`, `/add-slack`, etc.) |
| Give an agent access to a folder | Add to `containerConfig.additionalMounts` in the registered_groups DB entry |
| Schedule a recurring task | Ask the main agent or any group agent to call `schedule_task` |
