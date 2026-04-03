# Claude Code — Phases 1–3 Research Findings

> **Context**: This document captures the Phase 1–3 analysis of the leaked Claude Code source code
> (leaked 2026-03-31 via npm source-map), with a focus on understanding what Claude Code is, how it is
> architected, and what its core systems do — as a foundation for subsequent phases (Phase 4–7) that
> evaluate it as a reference design for a no-code AI agents builder.

---

## Phase 1 — Repository Overview & Architecture

### 1.1 What Is Claude Code?

Claude Code is Anthropic's official **agentic CLI** for software engineering. It lets users interact
with Claude directly from the terminal to perform software engineering tasks: editing files, running
shell commands, searching codebases, managing git workflows, and orchestrating multi-agent pipelines.

- **Leaked**: 2026-03-31 via a `.map` source file exposed in the npm registry
- **Language**: TypeScript (strict)
- **Runtime**: Bun
- **Terminal UI**: React + [Ink](https://github.com/vadimdemedes/ink) (React rendered to TTY)
- **Scale**: ~1,900 files, 512,000+ lines of code
- **LLM API**: Anthropic SDK (`@anthropic-ai/sdk`)

### 1.2 High-Level Directory Structure

```
src/
├── main.tsx                    # CLI entrypoint (Commander.js parser, 4,683 lines)
├── QueryEngine.ts              # Core LLM query engine (1,295 lines)
├── Tool.ts                     # Tool type system / base interface (792 lines)
├── tools.ts                    # Tool registry (assembleToolPool, getTools, 389 lines)
├── commands.ts                 # Slash command registry (754 lines)
├── context.ts                  # System/user context injection (189 lines)
├── query.ts                    # Turn-level LLM orchestration loop
├── Task.ts                     # Task type system (IDs, statuses, state)
│
├── assistant/                  # KAIROS: "assistant mode" (proactive AI assistant)
├── bootstrap/                  # Startup state, session IDs, CLI option singletons
├── bridge/                     # Always-on bridge: forward sessions to claude.ai
├── buddy/                      # Companion sprite / friend observer
├── cli/                        # CLI transport handlers
├── commands/                   # ~50 slash command implementations
├── components/                 # ~140 Ink UI components
├── constants/                  # Shared constants (XML tags, tool names, etc.)
├── context/                    # Notifications, stats
├── coordinator/                # Coordinator mode: orchestrates background workers
├── entrypoints/                # SDK types, CLI entry, agent SDK
├── hooks/                      # React hooks (useCanUseTool, useAppState, etc.)
├── memdir/                     # CLAUDE.md memory-file loading (memdir)
├── plugins/                    # Plugin system (bundled + marketplace)
├── screens/                    # Full-screen UI views (Doctor, REPL, Resume)
├── services/                   # External integrations (Anthropic API, MCP, analytics)
├── skills/                     # Built-in skill definitions
├── state/                      # AppState store (React context + zustand-like store)
├── tasks/                      # Task runners (LocalAgent, RemoteAgent, InProcessTeammate, Dream)
├── tools/                      # ~40 tool implementations (one directory each)
├── types/                      # Shared TypeScript types
├── utils/                      # Utility modules (hooks, permissions, model, session, git…)
└── voice/                      # Voice input support
```

### 1.3 Core Execution Path

```
CLI (main.tsx)
  └─→ init() + feature-flag loading (GrowthBook / Statsig)
  └─→ MCP server connections
  └─→ React/Ink render → REPL.tsx (main interactive loop)
        └─→ useAppState() + useMergedTools()
        └─→ processUserInput() → QueryEngine.runQuery()
              └─→ query() (turn loop)
                    ├─→ Anthropic API (claude.ts)
                    ├─→ Tool dispatch → tool.call()
                    ├─→ Permission checks (permissions.ts)
                    ├─→ Hook execution (hookHelpers.ts)
                    └─→ Context compaction (compact.ts) when near limit
```

### 1.4 Operating Modes

| Mode | Description | Activation |
|---|---|---|
| **Interactive (REPL)** | Default TUI with prompt input | Default |
| **Non-interactive / pipe** | Single-shot: read prompt from stdin, print result, exit | `echo "..." | claude` |
| **Bare mode** | Minimal context, no CLAUDE.md auto-discovery | `--bare` flag |
| **Coordinator mode** | Main thread becomes an orchestrator; spawns background workers | `CLAUDE_CODE_COORDINATOR_MODE=1` |
| **Teammate mode** | Peer to another Claude in a swarm; shares task list via tmux/iTerm2 pane | `--team` flag |
| **In-process teammate** | Same-process peer (no tmux required) | Feature-gated |
| **Remote mode** | Runs in Anthropic's CCR (Cloud Container Runtime) | `--remote` |
| **Assistant mode (KAIROS)** | Proactive AI assistant variant | Feature-gated |
| **Simple mode** | Only Bash + Read + Edit tools | `CLAUDE_CODE_SIMPLE=1` |
| **SDK mode** | Exposed as an SDK for other applications | `@anthropic-ai/claude-code` |

---

## Phase 2 — Core Systems Deep Dive

### 2.1 QueryEngine — The LLM Loop

**File**: `src/QueryEngine.ts` (1,295 lines)  
**Class**: `QueryEngine`

The QueryEngine is the core interaction loop. It manages a full conversation session:

**Key responsibilities:**
- Loads/reloads system prompt (CLAUDE.md, git status, memory files, tool definitions)
- Handles multi-turn agentic loops (tool use → result → continue)
- Tracks token usage and cost
- Triggers context compaction when approaching the context window limit
- Fires lifecycle hooks (`SessionStart`, `Stop`, `PreToolUse`, `PostToolUse`, etc.)
- Writes conversation transcripts to JSONL sidechain files
- Handles permission checks for every tool call
- Manages file history snapshots (for undo/rewind)

**Config type:**
```typescript
type QueryEngineConfig = {
  model: ModelSetting
  permissionMode: PermissionMode
  maxTurns?: number
  tools: Tools
  systemPrompt: SystemPrompt
  abortController: AbortController
  // ... ~20 more fields
}
```

**The turn loop** (`query.ts`) runs until:
1. The model produces a response with no tool calls (done), or
2. `maxTurns` is reached, or
3. `TaskStopTool` is called (graceful stop), or
4. An abort signal fires

### 2.2 Tool System

**Files**: `src/Tool.ts` (base interface), `src/tools.ts` (registry), `src/tools/*/` (implementations)

Every tool implements the `Tool` interface:

```typescript
interface Tool {
  name: string
  searchHint?: string               // for ToolSearch
  description(): Promise<string>    // shown to model
  prompt(): Promise<string>         // injected into system prompt
  inputSchema: ZodSchema
  outputSchema?: ZodSchema
  call(input, context): Promise<ToolResult>
  checkPermissions(input, context): Promise<PermissionDecision>
  renderToolUseMessage(input): ReactNode     // terminal UI
  renderToolResultMessage(output): ReactNode // terminal UI
  getActivityDescription?(input): string    // "Reading auth.ts" for monitoring
  isEnabled(): boolean
  isConcurrencySafe?(): boolean
  shouldDefer?: boolean
}
```

**Tool registry** (`src/tools.ts`):
- `getAllBaseTools()` — the exhaustive list of all built-in tools
- `getTools(permissionContext)` — filtered for current mode
- `assembleToolPool(permissionContext, mcpTools)` — built-ins + MCP tools, deduplicated and sorted for prompt-cache stability

**Full tool inventory** (40 built-in tools):

| Tool | Category | Purpose |
|---|---|---|
| `AgentTool` | Orchestration | Spawn a sub-agent (background task) |
| `SendMessageTool` | Orchestration | Send a message to a named/ID'd agent |
| `TaskStopTool` | Orchestration | Stop a running background agent |
| `TaskOutputTool` | Orchestration | Emit structured output from an agent |
| `TeamCreateTool` | Swarm | Create a swarm team |
| `TeamDeleteTool` | Swarm | Delete a swarm team |
| `TaskCreateTool` | Task mgmt | Create a task in the shared task list (TodoV2) |
| `TaskGetTool` | Task mgmt | Read a specific task |
| `TaskUpdateTool` | Task mgmt | Update a task's status/owner |
| `TaskListTool` | Task mgmt | List all tasks in the task list |
| `TodoWriteTool` | Task mgmt | Manage session todo list (TodoV1) |
| `BashTool` | Execution | Run shell commands |
| `REPLTool` | Execution | Sandboxed REPL VM (ant-only) |
| `SleepTool` | Execution | Sleep for N ms (async agent pacing) |
| `FileReadTool` | File I/O | Read file contents |
| `FileEditTool` | File I/O | Edit files (string replacement) |
| `FileWriteTool` | File I/O | Write new files |
| `NotebookEditTool` | File I/O | Edit Jupyter notebooks |
| `GlobTool` | Search | Find files by glob pattern |
| `GrepTool` | Search | Regex search across files |
| `WebSearchTool` | Web | Search the internet |
| `WebFetchTool` | Web | Fetch a URL |
| `WebBrowserTool` | Web | Browser automation (feature-gated) |
| `SkillTool` | Skills | Invoke a slash command / skill |
| `ToolSearchTool` | Meta | Search available tools by description |
| `LSPTool` | Dev | Language server protocol operations |
| `EnterPlanModeTool` | Plan | Enter plan-only mode |
| `ExitPlanModeV2Tool` | Plan | Submit plan and exit plan mode |
| `EnterWorktreeTool` | Isolation | Enter a git worktree sandbox |
| `ExitWorktreeTool` | Isolation | Exit worktree sandbox |
| `ConfigTool` | Config | Read/write settings (ant-only) |
| `ScheduleCronTool` | Triggers | Create/delete/list cron triggers |
| `RemoteTriggerTool` | Triggers | Register remote webhook trigger |
| `AskUserQuestionTool` | HITL | Ask user a question (pauses agent) |
| `MCPTool` | MCP | Call an external MCP tool |
| `ListMcpResourcesTool` | MCP | List MCP server resources |
| `ReadMcpResourceTool` | MCP | Read an MCP resource |
| `BriefTool` | UI | Produce a brief status summary |
| `TungstenTool` | Internal | Terminal capture (ant-only) |
| `WorkflowTool` | Workflow | Execute workflow scripts |

### 2.3 Agent Definition System

**File**: `src/tools/AgentTool/loadAgentsDir.ts`

Agents are defined as **Markdown files with YAML frontmatter** stored in:
- `~/.claude/agents/<name>.md` — user-scope
- `.claude/agents/<name>.md` — project-scope
- `~/.claude/managed-agents/<name>.md` — policy/enterprise-scope
- Built-in: `src/tools/AgentTool/built-in/`

**Agent definition schema:**

```typescript
type BaseAgentDefinition = {
  agentType: string          // unique name (e.g. "researcher")
  whenToUse: string          // description shown to orchestrator
  tools?: string[]           // allowed tools (["Read", "Bash", "Agent(worker)"])
  disallowedTools?: string[] // blocked tools
  skills?: string[]          // pre-loaded skills
  mcpServers?: AgentMcpServerSpec[] // agent-specific MCP servers
  hooks?: HooksSettings      // session-scoped lifecycle hooks
  color?: AgentColorName     // color in terminal UI
  model?: string             // model override ("claude-opus-4-5", "inherit")
  effort?: EffortValue       // thinking effort level
  permissionMode?: PermissionMode  // default | acceptEdits | bypassPermissions | plan
  maxTurns?: number          // max turns before stopping
  background?: boolean       // always run in background
  initialPrompt?: string     // prepended to first user turn
  memory?: 'user' | 'project' | 'local'  // persistent memory scope
  isolation?: 'worktree' | 'remote'      // file system isolation
  omitClaudeMd?: boolean     // skip CLAUDE.md injection (performance)
  getSystemPrompt(): string  // the agent's system prompt
}
```

**Agent source priority** (later overrides earlier of same name):
```
built-in → plugin → user (global) → project → policy/enterprise
```

**Three source types:**
- `BuiltInAgentDefinition` — has `source: 'built-in'`, dynamic `getSystemPrompt(toolUseContext)`
- `CustomAgentDefinition` — has `source: SettingSource`, static `getSystemPrompt()`
- `PluginAgentDefinition` — has `source: 'plugin'`, static `getSystemPrompt()`

### 2.4 Built-In Agents

**Directory**: `src/tools/AgentTool/built-in/`

| Agent | Role | Key Properties |
|---|---|---|
| `Explore` | Fast codebase explorer | Read-only tools, `omitClaudeMd: true`, Haiku model |
| `Plan` | Planning / analysis | Same tools as Explore, `omitClaudeMd: true` |
| `general-purpose` | Full-capability sub-agent | `tools: ['*']` (all tools) |
| `verification` | Code review + testing | `background: true` |
| `claudeCodeGuide` | Explains Claude Code features | Read-only |

### 2.5 Coordinator Mode

**File**: `src/coordinator/coordinatorMode.ts`

When `CLAUDE_CODE_COORDINATOR_MODE=1`, the main session becomes a **coordinator**:

- Gets only orchestration tools: `AgentTool`, `SendMessageTool`, `TaskStopTool`, and MCP tools
- Gets a ~500-line injected system prompt (`getCoordinatorSystemPrompt()`) that teaches it:
  - **Role**: delegate, don't do
  - **Task workflow**: Research → Synthesis → Implementation → Verification
  - **Concurrency**: launch independent workers in parallel
  - **Worker results**: arrive as `<task-notification>` XML in user-role messages
  - **Anti-patterns**: lazy delegation, missing context, worker-checks-on-worker
- Workers run fully async and notify coordinator via `<task-notification>` when done

**Task notification format:**
```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>{human-readable status summary}</summary>
  <result>{agent's final text response}</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

### 2.6 Task System

**Files**: `src/Task.ts`, `src/utils/tasks.ts`, `src/tasks/*/`

**Task types** (7 total):

| TaskType | ID Prefix | Description |
|---|---|---|
| `local_agent` | `a` | A background sub-agent (AgentTool spawn) |
| `remote_agent` | `r` | An agent running in Anthropic CCR |
| `in_process_teammate` | `t` | A swarm peer in the same process |
| `local_bash` | `b` | A shell command (BashTool) |
| `local_workflow` | `w` | A workflow script execution |
| `monitor_mcp` | `m` | An MCP server monitoring loop |
| `dream` | `d` | Speculative/idle background task |

**Task lifecycle states**: `pending → running → completed | failed | killed`

**TodoV2 Task schema** (`~/.claude/tasks/<taskListId>/`):
```typescript
type Task = {
  id: string
  subject: string             // brief title
  description: string        // what needs to be done
  activeForm?: string        // "Running tests" (present continuous for spinner)
  status: 'pending' | 'in_progress' | 'completed'
  owner?: string             // which agent owns this task
  blocks: string[]           // task IDs this blocks
  blockedBy: string[]        // task IDs that block this
  metadata: Record<string, unknown>
}
```

**Shared task list**: In swarm/team mode, all teammates share the same task list keyed by team name.

### 2.7 Session Storage & Transcript Persistence

**File**: `src/utils/sessionStorage.ts`

Every conversation is persisted as a JSONL sidechain at:
```
~/.claude/projects/<project-hash>/<sessionId>.jsonl
```

Sub-agent transcripts are persisted at:
```
~/.claude/subagents/<agentId>/transcript.jsonl
```

The transcript stores all `TranscriptMessage` types:
- `user` — user messages (including `<task-notification>` injections)
- `assistant` — model responses
- `attachment` — file/image attachments
- `compact_boundary` — context compaction markers
- `tool_use_summary` — compressed tool use records

**Resume**: `--resume` flag reloads a prior session transcript and continues from the last message.

**Agent resume**: `SendMessageTool` targeting a stopped agent automatically reloads its transcript and appends the new message, resuming the agent as a new `LocalAgentTask`.

### 2.8 Context System

**File**: `src/context.ts`

Two injected context blocks per session:

**System context** (`getSystemContext()` — cached, stable for session):
- `gitStatus` — current branch, main branch, git user, last 5 commits, working tree status

**User context** (`getUserContext()` — cached, stable for session):
- `claudeMd` — contents of all CLAUDE.md files found (project + parent dirs + additional dirs)
- `currentDate` — today's date

Context is injected into the API request as additional user/system turn prefixes and prompt-cached for cost efficiency.

### 2.9 Permission System

**File**: `src/utils/permissions/PermissionMode.ts`

**Permission modes:**

| Mode | Description |
|---|---|
| `default` | Ask permission for write/execute operations |
| `acceptEdits` | Auto-accept file edits, ask for everything else |
| `bypassPermissions` | Accept all tool calls without prompting |
| `plan` | Read-only until agent calls `ExitPlanMode` with a plan |
| `auto` | Classifier auto-decides (ant-only) |
| `bubble` | Surface permission prompts to parent terminal |

**Hook-based permission checks**: The `PreToolUse` hook can also return a `block` or `ask` decision, adding a second permission layer on top of the mode.

**Permission rules**: Stored in settings as allow/deny rules with optional tool name patterns (supports glob matching, e.g. `mcp__slack__*`).

### 2.10 Hook System

**Files**: `src/utils/hooks/`, `src/schemas/hooks.ts`, `src/entrypoints/sdk/coreTypes.ts`

**27 lifecycle hook events:**

```typescript
const HOOK_EVENTS = [
  'PreToolUse',         // before any tool call — can block
  'PostToolUse',        // after successful tool call
  'PostToolUseFailure', // after failed tool call
  'Notification',       // agent sends a notification
  'UserPromptSubmit',   // user submits a prompt
  'SessionStart',       // session starts
  'SessionEnd',         // session ends
  'Stop',               // agent finishes normally
  'StopFailure',        // agent fails
  'SubagentStart',      // sub-agent spawned
  'SubagentStop',       // sub-agent completes
  'PreCompact',         // before context compaction
  'PostCompact',        // after context compaction
  'PermissionRequest',  // agent requests permission
  'PermissionDenied',   // permission was denied
  'Setup',              // initial setup event
  'TeammateIdle',       // teammate waiting for work
  'TaskCreated',        // new task created
  'TaskCompleted',      // task completed
  'Elicitation',        // MCP elicitation request
  'ElicitationResult',  // MCP elicitation result
  'ConfigChange',       // settings changed
  'WorktreeCreate',     // git worktree created
  'WorktreeRemove',     // git worktree removed
  'InstructionsLoaded', // CLAUDE.md loaded
  'CwdChanged',         // working directory changed
  'FileChanged',        // file was modified
]
```

**Hook handler types (4 kinds):**
1. **Bash command hook** — run a shell script
2. **HTTP webhook hook** — POST to a URL
3. **Prompt injection hook** — inject text into the agent's next user turn
4. **Agent hook** — spawn a sub-agent to handle the event

**Hook matchers**: Each hook event maps to an array of matchers. A matcher specifies which tool names (or `*` for all) trigger the hook, plus the handler definition.

### 2.11 Memory System

**File**: `src/tools/AgentTool/agentMemory.ts`

Three persistent memory scopes per agent type:

| Scope | Path | Shared? |
|---|---|---|
| `user` | `~/.claude/agent-memory/<agentType>/MEMORY.md` | Across all projects |
| `project` | `.claude/agent-memory/<agentType>/MEMORY.md` | Per project, version controlled |
| `local` | `.claude/agent-memory-local/<agentType>/MEMORY.md` | Per project, not committed |

Memory is a Markdown file loaded at agent startup and injected as context. Agents write to memory using `FileWriteTool` or `FileEditTool` directly.

**Snapshot propagation**: `agentMemorySnapshot.ts` allows copying a project-scope memory snapshot to user-scope (team knowledge sharing).

### 2.12 Context Compaction

**Files**: `src/services/compact/`

When a session approaches the context window limit, Claude Code automatically compacts:

1. **Auto-compact trigger**: When token count exceeds ~80% of the context limit
2. **Compaction**: A secondary model call summarizes the conversation so far
3. **Post-compact messages**: Recent file edits are re-injected as full context
4. **Micro-compact**: Lightweight incremental compaction between full compacts
5. **Reactive compact**: Triggered by the model itself via structured signal
6. **Hooks**: `PreCompact` and `PostCompact` hooks fire around compaction

### 2.13 MCP Integration

**Files**: `src/services/mcp/`

MCP (Model Context Protocol) is used to extend Claude Code with external tools:

**Transport types**:
- `stdio` — subprocess, default
- `sse` — HTTP Server-Sent Events
- `http` — HTTP streaming
- `ws` / `sse-ide` — IDE extension connections
- `sdk` — in-process (SDK embedding)

**Scopes**: `local | user | project | dynamic | enterprise | claudeai | managed`

**Per-agent MCP servers**: Agents can declare their own private MCP servers in their frontmatter:
```yaml
mcpServers:
  - slack                          # reference to existing named server
  - { myserver: { type: stdio, command: npx, args: [my-mcp] } }  # inline definition
```

### 2.14 Swarm / Teammate System

**Files**: `src/utils/swarm/`, `src/tasks/InProcessTeammateTask/`

The **swarm system** lets multiple Claude instances collaborate as a team:

**Backends:**
- `tmux` — each teammate spawns a tmux pane
- `iterm2` — each teammate spawns an iTerm2 split pane
- `in-process` — all teammates share the same process (no terminal needed)

**Teammate-only tools** (beyond async agent tools):
- `TaskCreateTool`, `TaskGetTool`, `TaskListTool`, `TaskUpdateTool` — shared task list
- `SendMessageTool` — peer-to-peer messaging
- `CronCreateTool`, `CronDeleteTool`, `CronListTool` — teammate-owned cron triggers

**Team leader**: registers a "leader team name" which determines the shared task list ID.

### 2.15 Skill System

**Files**: `src/tools/SkillTool/`, `src/commands/`, `src/skills/`

Skills are **slash commands** (`/commit`, `/verify`, etc.) that are reusable agent sub-routines:

- Defined as Markdown files with YAML frontmatter in `.claude/commands/` or `~/.claude/commands/`
- Can include allowed tools, a model override, hooks, and a system prompt
- Invoked by the `SkillTool` (which the LLM calls as a tool)
- Execute as **forked sub-agents** that share the parent's transcript context
- Track usage for the skill suggestion system

**Official marketplace skills**: validated skills published by Anthropic (verified via `isOfficialMarketplaceName()`).

### 2.16 Cost & Token Tracking

**File**: `src/cost-tracker.ts`

Per-session tracking:
- Input tokens, output tokens, cache write tokens, cache read tokens
- Cost in USD (computed from model-specific pricing)
- Per-model breakdown
- Persisted to project config for resume

---

## Phase 3 — Feature Inventory

### 3.1 Core Agent Features

| Feature | Status | Notes |
|---|---|---|
| Single-agent interactive session | ✅ Shipped | Default mode |
| Multi-turn agentic loop | ✅ Shipped | Runs until done/maxTurns |
| Tool use + permission checks | ✅ Shipped | Per-tool granularity |
| Plan mode (read-then-approve) | ✅ Shipped | `permissionMode: plan` |
| Accept edits mode | ✅ Shipped | `permissionMode: acceptEdits` |
| YOLO/bypass mode | ✅ Shipped | `permissionMode: bypassPermissions` |
| Context auto-compaction | ✅ Shipped | Auto at ~80% context |
| Session resume | ✅ Shipped | `--resume` flag |
| File history / undo | ✅ Shipped | `fileHistoryMakeSnapshot()` |
| Thinking / extended reasoning | ✅ Shipped | `ThinkingConfig`, effort levels |
| Effort levels (1–5 + named) | ✅ Shipped | `low | medium | high | max` |

### 3.2 Multi-Agent Features

| Feature | Status | Notes |
|---|---|---|
| Background sub-agents (AgentTool) | ✅ Shipped | Async, results via `<task-notification>` |
| Named agent routing | ✅ Shipped | `agentNameRegistry` Map |
| Agent auto-resume on message | ✅ Shipped | `resumeAgent.ts` + transcript reload |
| Coordinator mode | ✅ Shipped | `CLAUDE_CODE_COORDINATOR_MODE=1` |
| Parallel fan-out (multi-agent in one turn) | ✅ Shipped | Multiple AgentTool calls = parallel |
| Agent-to-agent messaging | ✅ Shipped | `SendMessageTool` |
| Message queue with priorities | ✅ Shipped | `messageQueueManager.ts` |
| Task broadcast (`to: '*'`) | ✅ Shipped | `SendMessageTool` broadcast |
| Worker context inheritance (fork) | ✅ Shipped | `FORK_SUBAGENT` experiment |
| Shared task list (all agents) | ✅ Shipped | `TaskCreateTool`/`TaskUpdateTool` |
| Task dependency edges | ✅ Shipped | `blocks` / `blockedBy` fields |
| Agent progress + live summary | ✅ Shipped | `AgentProgress` + `AgentSummary` service |
| Worktree file isolation | ✅ Shipped | `isolation: worktree` per-agent |
| Remote agent execution (CCR) | ✅ Shipped (ant-only) | `isolation: remote` |
| Swarm / teammate multi-pane | ✅ Shipped | tmux / iTerm2 / in-process backends |
| Verification agent | ✅ Shipped | `background: true`, fires after main agent |

### 3.3 Integration & Extension Features

| Feature | Status | Notes |
|---|---|---|
| MCP tool integration | ✅ Shipped | stdio / SSE / HTTP / WS transports |
| Per-agent MCP servers | ✅ Shipped | Inline or by-name MCP server specs |
| Hook system (27 events) | ✅ Shipped | Bash / HTTP / prompt / agent handlers |
| Plugin system | ✅ Shipped | Marketplace + local plugins |
| Skill system (slash commands) | ✅ Shipped | Forked sub-agents, markdown definitions |
| Workflow scripts | ✅ Shipped (feature-gated) | `WORKFLOW_SCRIPTS` flag |
| Cron triggers | ✅ Shipped (feature-gated) | `AGENT_TRIGGERS` flag |
| Remote webhook triggers | ✅ Shipped (feature-gated) | `AGENT_TRIGGERS_REMOTE` flag |
| GitHub webhooks (subscribe PR) | ✅ Shipped (feature-gated) | `KAIROS_GITHUB_WEBHOOKS` flag |
| Push notifications | ✅ Shipped (feature-gated) | `KAIROS` / `KAIROS_PUSH_NOTIFICATION` flag |
| File watch triggers | ✅ Shipped | `FileChanged` hook event |

### 3.4 Memory & Persistence Features

| Feature | Status | Notes |
|---|---|---|
| Session transcript persistence | ✅ Shipped | JSONL sidechain per session |
| Sub-agent transcript persistence | ✅ Shipped | `~/.claude/subagents/<id>/transcript.jsonl` |
| Agent persistent memory (3 scopes) | ✅ Shipped | user / project / local MEMORY.md |
| Memory snapshot propagation | ✅ Shipped | project → user scope sharing |
| Todo list (v1) | ✅ Shipped | Session-scoped array |
| Task list (v2 / TodoV2) | ✅ Shipped | File-backed, shareable, with deps |
| CLAUDE.md hierarchical context | ✅ Shipped | Walk up from cwd, merge all files |
| File history snapshots | ✅ Shipped | For undo/rewind |
| Cost history per session | ✅ Shipped | Persisted to project config |

### 3.5 UI & UX Features

| Feature | Status | Notes |
|---|---|---|
| Terminal TUI (React/Ink) | ✅ Shipped | Full-featured terminal UI |
| Live agent progress panel | ✅ Shipped | Tool use count, last activity, summary |
| Task list panel | ✅ Shipped | Live task list sidebar |
| Teammate view | ✅ Shipped | Multi-pane team visualization |
| Coordinator task panel | ✅ Shipped | Worker rows with status |
| Permission dialogs | ✅ Shipped | Interactive approve/deny |
| Plan review UI | ✅ Shipped | Show proposed plan, ask for approval |
| Diff view | ✅ Shipped | File diffs before/after edits |
| Token/cost display | ✅ Shipped | Per-session token and cost tracking |
| Speculative edits (pre-render) | ✅ Shipped | Optimistic UI for expected edits |
| Session resume chooser | ✅ Shipped | Browse and select prior sessions |
| Doctor / diagnostics screen | ✅ Shipped | `/doctor` command |
| Voice input | ✅ Shipped | `src/voice/` |
| Companion sprite | ✅ Shipped | `src/buddy/` |
| Asciicast recorder | ✅ Shipped | `installAsciicastRecorder()` |

### 3.6 Security & Safety Features

| Feature | Status | Notes |
|---|---|---|
| Permission mode system | ✅ Shipped | 5 modes + enterprise override |
| Per-tool allow/deny rules | ✅ Shipped | Glob pattern matching |
| SSRF guard (HTTP hooks) | ✅ Shipped | `ssrfGuard.ts` |
| Locked surfaces (enterprise) | ✅ Shipped | Can lock skills, hooks, settings |
| HTTP hook URL allowlist | ✅ Shipped | `hookAllowedHosts` setting |
| Env var allowlist (HTTP hooks) | ✅ Shipped | `hookAllowedEnv` setting |
| Path traversal prevention | ✅ Shipped | Worktree slug validation |
| Scratchpad sandboxing | ✅ Shipped (feature-gated) | `tengu_scratch` gate |
| MDM / enterprise policy | ✅ Shipped | `managed-settings.json` |

### 3.7 Developer / SDK Features

| Feature | Status | Notes |
|---|---|---|
| CLI entrypoint | ✅ Shipped | `claude` command |
| SDK embedding | ✅ Shipped | `@anthropic-ai/claude-code` package |
| SDK control protocol | ✅ Shipped | `SDKControlRequest` / `SDKControlResponse` |
| Non-interactive / pipe mode | ✅ Shipped | stdin → stdout |
| SDK message stream | ✅ Shipped | Async iterable of `SDKMessage` |
| Custom tool injection | ✅ Shipped | Pass tools to `claude()` SDK call |
| LSP integration | ✅ Shipped (opt-in) | `ENABLE_LSP_TOOL=1` |
| REPL / sandboxed VM mode | ✅ Shipped (ant-only) | Bun VM for code execution |
| Workflow scripts | ✅ Shipped (feature-gated) | Programmatic workflow definitions |

### 3.8 Feature Flags Inventory

Claude Code uses **Statsig / GrowthBook** feature gates for gradual rollout. The following flags control experimental or in-progress features:

| Flag | Feature |
|---|---|
| `COORDINATOR_MODE` | Coordinator / multi-worker orchestration |
| `AGENT_TRIGGERS` | Cron-based agent triggers |
| `AGENT_TRIGGERS_REMOTE` | Webhook-based remote triggers |
| `KAIROS` | Assistant / proactive AI mode |
| `KAIROS_PUSH_NOTIFICATION` | Mobile push notifications |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub PR event subscriptions |
| `WORKFLOW_SCRIPTS` | Workflow script execution |
| `VERIFICATION_AGENT` | Auto-spawn verification agent |
| `FORK_SUBAGENT` | Context-inheriting parallel forks |
| `PROACTIVE` | Proactive agent suggestions |
| `WEB_BROWSER_TOOL` | Browser automation tool |
| `HISTORY_SNIP` | Conversation history snipping |
| `CONTEXT_COLLAPSE` | Context collapse optimization |
| `TERMINAL_PANEL` | Terminal capture tool |
| `UDS_INBOX` | Unix domain socket peer discovery |
| `REACTIVE_COMPACT` | Model-triggered context compaction |
| `MONITOR_TOOL` | MCP server monitoring tool |
| `OVERFLOW_TEST_TOOL` | Context overflow testing |
| `BREAK_CACHE_COMMAND` | Cache-breaking debug command |
| `EXPERIMENTAL_SKILL_SEARCH` | Remote skill search |

---

## Phase 3 Supplemental — Data Models

### Agent Progress Model

```typescript
type AgentProgress = {
  toolUseCount: number
  tokenCount: number
  lastActivity: ToolActivity | null
  recentActivities: ToolActivity[]   // last 5
  summary?: string                   // AI-generated 1-2 sentence summary (fires every ~60s)
}

type ToolActivity = {
  toolName: string
  input: unknown
  activityDescription: string        // e.g. "Reading src/auth/validate.ts"
  isSearch: boolean
  isRead: boolean
}
```

### AppState Key Fields

```typescript
type AppState = {
  settings: SettingsJson
  mainLoopModel: ModelSetting
  permissionMode: PermissionMode
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>     // name → agentId routing
  todos: { [agentId: string]: TodoList }       // TodoV1 per-agent
  mcp: { clients, tools, commands, resources }
  messages: Message[]                          // main conversation
  // ... ~60 more fields
}
```

### Model Aliases

```
sonnet   → latest claude-sonnet-* (general use)
opus     → latest claude-opus-*   (most capable)
haiku    → latest claude-haiku-*  (fast/cheap)
best     → alias for opus
sonnet[1m] → sonnet with 1M token context
opus[1m]   → opus with 1M token context
opusplan   → opus in plan mode
```

### Tool Permission Tiers

```
Tier 1 — Any async agent:
  Read, WebSearch, TodoWrite, Grep, WebFetch, Glob,
  Bash, Edit, Write, NotebookEdit, Skill, ToolSearch,
  EnterWorktree, ExitWorktree

Tier 2 — In-process teammates only:
  TaskCreate, TaskGet, TaskList, TaskUpdate,
  SendMessage, CronCreate, CronDelete, CronList

Tier 3 — Main thread / coordinator only:
  AgentTool, AskUserQuestion, ExitPlanMode,
  EnterPlanMode, TaskOutput, TaskStop

Always blocked for sub-agents:
  TaskOutput, ExitPlanMode, EnterPlanMode,
  AgentTool (unless USER_TYPE=ant), AskUserQuestion,
  TaskStop, WorkflowTool
```

---

*End of Phase 1–3 Research Findings.*
*See `docs/phases-4-5-7-extra.md` for Phase 4–7 analysis and MVP recommendations for a no-code AI agents builder.*
