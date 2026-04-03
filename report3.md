# Phase 4 — Comparative Analysis Against the No-Code AI Agents Builder Market

## 4.1 What Claude Code Is (and Isn't)

Claude Code is a developer-grade agentic runtime, not a no-code platform. Its primary abstractions — YAML/Markdown agent definitions, TypeScript hooks, git worktrees, tmux panes — all require engineering knowledge. However, it has solved many hard problems that every no-code AI agent platform must eventually solve, and the solutions are well-engineered enough to serve as direct blueprints.

The strategic question is: which of its solved problems apply directly to a no-code builder, and which are software-engineing-specific and therefore irrelevant?

## 4.2 The No-Code Agent Builder Context (What Your Platform Needs)

A no-code AI agents builder typically exposes:

- A canvas or visual editor where users drag-and-drop nodes (agents, tools, conditions, data)
- A gallery of ready-made agents with natural-language descriptions
- A configuration panel for each agent (name, instructions, tools, model, memory)
- A trigger system (on schedule, on event, on API call, on user message)
- A messaging/routing system (how agents hand off to each other)
- A human-in-the-loop system (approval gates, review steps)
- A monitoring system (what is this agent doing right now, what did it produce)
- A persistence system (what does this agent remember across runs)

Claude Code has engineering solutions for every one of these. The challenge is translating their developer abstractions into no-code UX.

---

# Phase 5 — Deep Feature-by-Feature Extraction

## 5.1 Agent Definition System → Steal Entirely

### What Claude Code does:

Every agent is a Markdown file with YAML frontmatter:

```markdown
---
name: researcher
description: "For research and investigation tasks"
model: claude-opus-4-5
tools: [Bash, Read, Edit, "Agent(worker)"]
disallowedTools: [WebSearch]
permissionMode: default
maxTurns: 50
background: true
memory: user
isolation: worktree
---
You are a specialized research agent. Your job is...
```

Resolution priority: policy > project > user > plugin > built-in — later-defined agents override earlier ones with the same name.

### What to steal for no-code:

The data model is perfect. Every field maps directly to a no-code UI concept:

| Claude Code field | No-Code UI Widget |
|---|---|
| name | Agent name input |
| description (= whenToUse) | Purpose/description textarea |
| model | Model dropdown |
| tools: [...] | Tool picker checklist |
| disallowedTools | Blocked tools list |
| permissionMode | "Requires approval" toggle |
| maxTurns | Turn limit slider |
| background | "Run in background" toggle |
| memory | Memory scope dropdown |
| isolation: worktree | "Isolated filesystem" toggle (power user feature) |
| System prompt body | Instructions text editor |

Build now: Name, description, instructions, model, tool picker, approval toggle.  
Build later: Memory scope, tool type restrictions (Agent(worker)), maxTurns, isolation.  
Ignore: source (SettingSource) — internal priority system, not user-facing.

---

## 5.2 Tool Catalog → Steal the Design, Replace the Tools

### What Claude Code has:

```text
ASYNC_AGENT_ALLOWED_TOOLS (workers can use):
  Read, WebSearch, TodoWrite, Grep, WebFetch, Glob, Bash,
  Edit, Write, NotebookEdit, Skill, ToolSearch, EnterWorktree, ExitWorktree

IN_PROCESS_TEAMMATE_ALLOWED_TOOLS (swarm teammates only):
  TaskCreate, TaskGet, TaskList, TaskUpdate, SendMessage, ScheduleCron

ALL_AGENT_DISALLOWED_TOOLS (no sub-agent can use):
  TaskOutput, ExitPlanMode, EnterPlanMode, AgentTool*, AskUserQuestion, TaskStop
```

### Why this matters for no-code:

Claude Code invented a layered permission model that distinguishes:

- What any agent can use (safe for background work)
- What only trusted peer agents can use (messaging, task management)
- What only the parent/coordinator can use (spawn, stop, plan-mode control)

This 3-tier tool permission system is exactly what a no-code builder needs. In no-code terms:

- Tier 1 = Data tools (read, search, write) — always available
- Tier 2 = Coordination tools (create task, notify another agent) — for team agents
- Tier 3 = Orchestration tools (spawn agent, stop agent, approve plan) — for coordinator agents

### What to steal:

Build a tool catalog where each tool has a category tag. Users see a filtered list based on what role their agent plays. Don't expose the raw set algebra — just show the right subset in context.

Concrete tool types to offer as MVP:

- HTTP Request (replaces WebFetch + WebSearch)
- Read/Write to a connected data source (replaces Read/Write/Edit)
- Run code snippet (replaces Bash) — sandboxed
- Send message to another agent (replaces SendMessage)
- Create/update a task in the workflow (replaces TaskCreate/TaskUpdate)
- Webhook output (new)
- Human approval gate (replaces AskUserQuestion + plan-mode)

---

## 5.3 Coordinator Pattern → Steal as the Core Orchestration Model

### What Claude Code does:

The coordinator (CLAUDE_CODE_COORDINATOR_MODE=1) is a fundamentally different agent configuration:

- Gets only orchestration tools (Agent, SendMessage, TaskStop) — NO file tools
- Receives a specialized 500-line system prompt that teaches it: Research → Synthesis → Implementation → Verification
- All workers are async; coordinator never blocks on them
- Worker results arrive as structured XML (`<task-notification>`) injected as user-role messages
- Coordinator learns the "continue vs spawn fresh" decision tree

### The single most important insight to steal:

**The coordinator does not do work. It delegates work.**

This is not obvious to LLMs without explicit prompting, and Claude Code has perfected the prompts for it. The key phrases from the system prompt:

- "Answer questions directly when possible — don't delegate work that you can handle without tools"
- "Never fabricate or predict agent results in any format — results arrive as separate messages"
- "After launching agents, briefly tell the user what you launched and end your response"
- "Parallelism is your superpower"
- "Worker-checks-on-worker is forbidden"

### What to steal for no-code:

- Coordinator node type: A special "Orchestrator" node in the canvas that has no file tools but CAN spawn/message/stop other agent nodes
- Async notification model: Sub-agents complete and send results back to orchestrator as messages, not synchronous return values
- Phase vocabulary: Expose the Research → Synthesis → Implementation → Verification phases as a template in the agent description
- Synthesis requirement: In the instructions for coordinator-type agents, teach the model to synthesize (not just relay) before delegating

What NOT to steal: The specific XML format (`<task-notification>`), the env var flag — those are implementation details.

---

## 5.4 SendMessageTool → Steal the Routing Logic

### What Claude Code does:

SendMessageTool is the inter-agent messaging primitive. Its routing logic (from `call()`) is sophisticated:

```text
1. Bridge/UDS → cross-machine message (ignore for no-code initially)
2. name registry lookup → by registered human name
3. agentId format lookup → by raw ID
4. If running → queue message in task.pendingMessages (delivered at next turn boundary)
5. If stopped → auto-resume from transcript + append message
6. to:'*' → broadcast to all team members
7. Mailbox file → swarm/teammate mode
8. Structured message → shutdown/plan-approval protocols
```

### The key insight to steal:

**Auto-resume:** when you send a message to a stopped agent, it automatically resumes from its last transcript. This means users don't have to manually restart agents — just send them a message.

In no-code terms: agents can be dormant between tasks and automatically wake up when messaged. This is a huge UX simplification — no "start/stop" buttons needed for most workflows.

### What to steal for no-code:

- Named agent routing (send to "researcher" not to "agent-a1b2c3d4")
- Broadcast (`to: '*'`) — notify all agents of a shared event
- Auto-resume semantics — idling agents wake up on message
- Message queueing — messages delivered at the next safe moment, not interrupting active work

What to skip for MVP: The mailbox file system, UDS/bridge cross-session channels, structured shutdown protocol — these are swarm-specific features.

---

## 5.5 Task System → Steal as the Primary State Model

### What Claude Code has:

Two overlapping task systems:

- **TodoWrite (v1):** Session-scoped todo list stored in AppState. An array of `{content, status, priority}` items.
- **Task tools (v2 — TodoV2):** File-backed, persistable tasks at `~/.claude/tasks/<taskListId>/`:

```typescript
Task {
  id: string
  subject: string
  description: string
  activeForm?: string      // "Running tests" (present continuous for spinner)
  status: 'pending' | 'in_progress' | 'completed'
  owner?: string           // which agent owns it
  blocks: string[]         // task IDs this task blocks
  blockedBy: string[]      // task IDs that block this task
  metadata: Record<string, unknown>
}
```

Operations: `TaskCreate`, `TaskGet`, `TaskList`, `TaskUpdate`.

In swarm mode: all teammates share the same task list (keyed by team name), so the leader and workers can all read/write the same tasks.

### What to steal:

This is a shared task list / work queue that multiple agents can read from and write to. It's the missing coordination primitive in most no-code builders. Instead of:

- One orchestrator managing everything in its prompt
- Agents communicating entirely via messages

You get:

- A structured shared state (the task list) visible to all agents in a workflow
- Dependency edges (`blocks`/`blockedBy`) — agents can wait for prerequisites
- Ownership (`owner`) — "take" a task like a Jira ticket
- Status transitions — pending → in_progress → completed

For no-code MVP:

- Show the live task list as a sidebar/panel in the workflow run view
- Let agents create and complete tasks
- Let the orchestrator pre-populate tasks as the plan, then workers claim and execute them

This is the central pattern for transparent, auditable multi-agent workflows.

---

## 5.6 Agent Progress + Summarization → Steal for the Monitoring Panel

### What Claude Code does:

Every background agent tracks:

```typescript
AgentProgress {
  toolUseCount: number
  tokenCount: number
  lastActivity: ToolActivity { toolName, input, activityDescription, isSearch, isRead }
  recentActivities: ToolActivity[]  // last 5
  summary?: string                  // from AgentSummarization service
}
```

The AgentSummary service (`src/services/AgentSummary/agentSummary.ts`):

- Fires every N minutes while agent runs
- Forks the agent's transcript
- Calls a secondary (cheap) model: "Summarize this agent's work in 1-2 sentences"
- Stores result in `progress.summary`
- Emits `emitTaskProgress` SDK events for VS Code panel

### What to steal for no-code:

This is exactly what makes background agents observable. In a no-code builder, users need to know what their agent is doing without reading raw transcripts.

Build a monitoring panel that shows for each running agent:

- Live status line: last tool used + tool input description (e.g., "Reading auth/validate.ts")
- Running summary: 1-2 sentence AI-generated summary ("Investigating the null pointer in the session expiry handler")
- Token/turn counters: so users know roughly how much work is left
- Recent activity list: last 5 tool calls with human-readable descriptions

The `getActivityDescription()` per-tool method (in the Tool interface) is the key — each tool knows how to describe itself.  
E.g., `FileReadTool.getActivityDescription({ file_path: 'auth/validate.ts' })` returns "Reading auth/validate.ts".

For no-code, add:

- Confidence/progress percentage (heuristic: toolUseCount / estimated maxTurns)
- An "explain what it's doing" button that fires an on-demand summary

---

## 5.7 Hook System → Steal as the Integration/Automation Layer

### What Claude Code has:

```typescript
HOOK_EVENTS = [
  'PreToolUse',         // before any tool call
  'PostToolUse',        // after successful tool call
  'PostToolUseFailure', // after failed tool call
  'Notification',       // when agent sends a notification
  'UserPromptSubmit',   // when user submits a prompt
  'SessionStart',       // agent session starts
  'SessionEnd',         // agent session ends
  'Stop',               // agent finishes normally
  'StopFailure',        // agent fails
  'SubagentStart',      // sub-agent spawned
  'SubagentStop',       // sub-agent completes
  'PreCompact',         // before context compaction
  'PostCompact',        // after context compaction
  'PermissionRequest',  // agent requests permission
  'PermissionDenied',   // permission was denied
  'TeammateIdle',       // teammate waiting for work
  'TaskCreated',        // new task created
  'TaskCompleted',      // task completed
  'WorktreeCreate',     // new git worktree created
  'WorktreeRemove',     // worktree removed
  'FileChanged',        // file was modified
  'CwdChanged',         // working directory changed
]
```

Each hook can be:

- Bash command: run a shell script
- HTTP webhook: POST to a URL
- Prompt injection: inject text into the agent's conversation
- Agent hook: spawn a subagent to handle the event

### What to steal for no-code:

This is a powerful automation/integration system that no-code builders usually implement as "triggers" or "automations". But Claude Code's hook system is more powerful because:

- Hooks run at 27 distinct lifecycle events, not just "start" and "end"
- Hooks can inject context INTO the agent's conversation (prompt injection)
- Hooks can block tool execution (PreToolUse can return a "block" decision)
- Hooks themselves can be agents

For no-code MVP, expose these hooks visually:

- Stop → "On complete: send webhook, create Jira ticket, send Slack message"
- PreToolUse(file_edit) → "Before editing files: notify user"
- TaskCreated → "When task created: update external project board"
- PermissionRequest → "When agent needs approval: pause and notify user"

This becomes the no-code builder's "Integrations" tab or "Automations" panel.

---

## 5.8 Permission/Approval System → Steal as the Human-in-the-Loop Model

### What Claude Code has:

Permission modes:

- `default` — ask for permission on any write operation
- `acceptEdits` — auto-accept file edits, ask for everything else
- `bypassPermissions` — accept everything (YOLO mode)
- `plan` — agent must submit a plan and get approval before executing
- `auto` — classifier decides what needs approval (ant-only)
- `bubble` — surface prompts to parent agent's terminal

The plan mode is the most interesting for no-code:

- Agent runs in plan mode — can only READ, not WRITE
- Agent calls `ExitPlanMode` with its proposed plan
- Plan is shown to user (or team lead)
- User approves (via `SendMessageTool(type:'plan_approval_response', approve:true)`)
- Agent re-runs in default mode, inheriting the leader's mode

### What to steal for no-code:

This is a "Plan before Act" gate. Make it a toggleable property on any agent node:

- OFF: agent executes autonomously
- ON: agent first generates a plan, user approves, then agent executes

This is THE killer feature for enterprise/risk-averse users.  
"My AI agent will never do anything destructive without showing me its plan first."

Also steal:

- Per-tool approval gates (PreToolUse hook returning `'ask'`)
- Batch approval — approve N pending operations at once
- Approval delegation — team lead approves on behalf of teammates

For no-code MVP:

- Simple toggle: "Require approval before executing"
- Show plan as a structured diff preview (what files will be changed, what APIs will be called)

---

## 5.9 Memory System → Steal the Scoping Model

### What Claude Code has:

Three memory scopes for agents:

```text
user    → ~/.claude/agents/<agentType>/MEMORY.md   (persists across all projects)
project → .claude/agents/<agentType>/MEMORY.md     (persists per-project, version controlled)
local   → .claude/agents/<agentType>-local/MEMORY.md (persists per-project, not shared)
```

Additionally:

- `agentMemorySnapshot` — project-level snapshots that can be copied to user-level (team knowledge propagation)
- Memory is loaded at agent startup and injected as context
- Agents write memory by modifying the MEMORY.md file directly (via Write tool)

### What to steal for no-code:

Three memory scopes map directly to no-code concepts:

- User scope = "Agent remembers across all of my workflows" (user-level memory)
- Project scope = "Agent remembers within this workflow/project" (workflow-scoped memory)
- Local scope = "Agent remembers on this run only" (run-scoped memory)

For no-code, also add:

- Shared scope = "All agents in this workflow share this memory"
- Team scope = "All users in this org share this memory"

Memory snapshot propagation (from project → user scope) is a "share knowledge" button — one workflow's learnings can bootstrap another. This is valuable for enterprise teams.

For no-code MVP: Just implement run-scoped and workflow-scoped memory. User/global scope is Phase 2.

---

## 5.10 Agent Resume → Steal the "Conversations Never Die" Pattern

### What Claude Code does:

Every agent's full conversation history is written to a JSONL sidechain transcript at `~/.claude/subagents/<agentId>/transcript.jsonl`.  
When SendMessageTool targets a stopped/completed/evicted agent:

- Transcript is loaded from disk
- Message filters clean it up (`filterUnresolvedToolUses`, `filterOrphanedThinkingOnlyMessages`)
- The new message is appended as a fresh user turn
- Agent resumes as a new LocalAgentTask with full prior context

This means: an agent you ran 3 days ago can be continued today by just sending it a message. No context is lost. No re-explaining needed.

### What to steal for no-code:

- Persistent agent sessions — every agent run is saved with full history
- Resume anywhere — send any stopped agent a follow-up message
- Conversation view — users can browse the full conversation history of any agent run
- Context transfer — when spawning a new agent for related work, optionally include prior agent's findings (analogous to `forkContextMessages`)

This inverts the typical no-code model (run → result → discard) into run → result → continue anytime. This dramatically reduces repeated work on long multi-day projects.

---

## 5.11 Parallel Execution → Steal the Concurrency Model

### What Claude Code does:

The coordinator makes multiple AgentTool calls in a single assistant message turn — all execute as parallel background tasks. The coordinator then:

- Gets back immediate "agent launched" results from all of them
- Continues working or tells the user what was launched
- Receives results asynchronously via `<task-notification>` messages, in whatever order they complete

The critical design: the tool itself is marked `isConcurrencySafe: true`, which signals to the runtime that multiple instances can execute in parallel.

### What to steal for no-code:

- Fan-out pattern: A special "Parallel" node that sends the same task to N copies of an agent simultaneously
- Aggregate node: Waits for all parallel agents to complete, then combines results
- First-wins node: Runs N agents in parallel, takes the result from whichever finishes first (for redundancy or A/B testing)
- Conditional fan-out: Based on the orchestrator's analysis, dynamically decide how many parallel agents to spawn

The key insight: parallel execution is not "run N workflows simultaneously" — it's "an agent can spawn N sub-agents and coordinate all of them." This is fundamentally more powerful than naive workflow parallelism.

---

## 5.12 Fork Subagent → Steal for Parallel Research Workflows

### What Claude Code does:

When an agent forks (FORK_SUBAGENT experiment), all fork children share an identical prompt-cache prefix. This means:

- Spawning 5 parallel researchers costs ~1.2× a single researcher in tokens (the shared prefix is cached)
- Each child gets the full conversation context of the parent
- Each child gets a unique directive appended after the shared context

### What to steal for no-code:

Context inheritance for parallel agents. When you spawn multiple agents to research different aspects of a problem, give them the full conversation context as their starting point, not just an isolated task description. This dramatically improves result quality because agents understand the full problem.

The structured output format from fork children is also brilliant:

```text
Scope: <what I was assigned>
Result: <key findings>
Key files: <relevant paths>
Files changed: <changes made>
Issues: <anything to flag>
```

Build this as a required output schema for research-type agents in your no-code builder. Users configure what fields they want, and agents must produce structured output in that format.

---

# Phase 7 — MVP Recommendations + Prioritization

## 7.1 What to Build NOW (MVP Core)

### 1. Agent Definition Canvas (Week 1–3)

A visual form (not necessarily a graph editor) that captures:

- Name, description/purpose text
- Model selector
- System prompt (instructions) text editor
- Tool picker (curated list of 8–12 tools, not 40)
- "Requires approval before acting" toggle
- Max turns slider (10–200)

This is the AgentDefinition from Claude Code, translated to a form UI. No drag-and-drop canvas needed yet.

### 2. Coordinator + Worker Role Templates (Week 2–3)

Pre-built agent configurations for the two roles:

- Coordinator template: Instructions focus on delegation, not doing. Has only messaging/spawn tools. System prompt includes the Research→Synthesis→Implementation→Verification framework.
- Worker template: Instructions focus on execution. Has file/code/data tools. Output format includes key findings, files changed, and summary.

Don't invent this from scratch — steal directly from `getCoordinatorSystemPrompt()`.

### 3. Async Task Notification Model (Week 2–4)

Implement the `<task-notification>` delivery pattern:

- Background agents run independently
- When complete, a notification is queued and delivered to the orchestrator
- Orchestrator receives it as a "message" (not a function return)
- Notification includes: agent ID, status, summary, result, token usage

This is the hardest piece architecturally but the most important for multi-agent workflows. Without it, you're stuck with sequential agents only.

### 4. Named Agent Routing + Auto-Resume (Week 3–5)

- Agents get human-readable names
- `SendMessage(to: 'researcher', message: '...')` routes by name
- Stopped agents automatically resume from their last conversation when messaged
- Running agents receive messages at the next safe boundary

This is the `agentNameRegistry` + `queuePendingMessage` + `resumeAgentBackground` system.

### 5. Shared Task List (Week 4–6)

A structured work queue visible to all agents in a workflow:

- Fields: subject, description, status, owner, blockedBy
- Coordinator creates tasks; workers claim and complete them
- Live task list visible in the monitoring panel as the workflow runs

This replaces ad-hoc "tell the agent what to do" with a structured, transparent execution plan.

### 6. Monitoring Panel (Week 5–7)

For each running agent:

- Last tool used (e.g., "Reading config.json")
- Running AI-generated summary (fire every 60s via cheap model)
- Turn count + estimated tokens used
- Status: running / paused / complete / failed

Steal AgentProgress data model + AgentSummary service pattern exactly.

## 7.2 What to Build LATER (Phase 2)

### 7. Plan-Mode Approval Gate

Agent submits a structured plan, user approves before execution. Implement via:

- First agent run: read-only, ends by outputting a plan in structured format
- User reviews plan in UI
- On approval: second run with write permissions + plan injected as context

The Claude Code plan-mode protocol (`ExitPlanMode` → `SendMessage` approval → re-run) is elegant but complex. The simpler two-run model above achieves the same UX.

### 8. Worktree File Isolation

Each agent gets an isolated copy of the filesystem. Changes are merged back only when the agent is done. Prevents parallel agents from conflicting on the same files.

This requires git integration. Build once the basic multi-agent model is working.

### 9. Agent Memory (Persistent)

Cross-session memory per agent stored in a markdown file. Agent reads it at startup; writes to it when it learns something important.

Implement scopes: run → workflow → user → org.

### 10. Cron/Trigger System

Agents that wake up on a schedule, on a webhook, on a file change, or when another agent finishes.

The `ScheduleCronTool` + `AGENT_TRIGGERS` feature flag in Claude Code shows how this integrates with the agent loop.

### 11. Hook-Based Integrations

Lifecycle hooks (on complete, on tool use, on permission request) that fire webhooks to external systems.

Steal the `HOOK_EVENTS` catalog directly — all 27 events are worth eventually supporting. Start with Stop and TaskCompleted.

### 12. Parallel Fan-Out Nodes

Visual node type: "run these 3 agents in parallel, wait for all, pass results to coordinator."

Powered by the `isConcurrencySafe: true` model and the `runAsyncAgentLifecycle` pattern.

## 7.3 What to IGNORE (Irrelevant for No-Code)

| Claude Code Feature | Why to Ignore |
|---|---|
| tmux/iTerm2 pane spawning | Dev tooling; no-code users don't have terminals |
| Git worktree management (`createAgentWorktree`) | Dev-specific; complex git prerequisite |
| Remote CCR execution (`isolation: 'remote'`) | Ant-internal infrastructure |
| UDS/bridge cross-machine messaging | Edge-case infrastructure |
| REPL prompt input system | Terminal-specific UI |
| Snapshot-based prompt caching (`buildForkedMessages` prefix optimization) | API-level optimization; no-code abstracts away |
| Sidechain JSONL transcript format | Implementation detail; use a database instead |
| Permission classifier (`classifyYoloAction`) | Internal auto-mode feature; not user-facing |
| processEnv env var-based configuration | Replace with database-backed configuration |
| Frontmatter-based agent loading | Replace with visual form + database storage |
| LSP tool integration | Developer-specific (language server protocol) |
| Notebook editing tools | Jupyter-specific |
| PowerShell tool | Dev-specific |
| `CLAUDE_CODE_*` env var system | Replace with settings UI |
| Statsig/GrowthBook feature gates | Replace with your own feature flag system |

## 7.4 Top 10 Ideas to Steal (Ranked)

1. **The Coordinator Pattern + System Prompt**  
   The 500-line coordinator system prompt in `getCoordinatorSystemPrompt()` is a finished, battle-tested product. Adapt it verbatim as the system prompt for "Orchestrator" agent nodes. It teaches the model how to delegate, synthesize, run parallel workers, handle failures, and avoid lazy delegation. This alone is worth the entire codebase study.

2. **Agent Resume ("Conversations Never Die")**  
   Any agent can be continued from any point by sending it a message. Full conversation history is persisted. This turns one-shot agents into persistent collaborators. In no-code UX: "Resume" button on any finished agent run.

3. **Task Notification as User Message**  
   Agent results arrive as user-role messages (`<task-notification>` XML), not as function returns. This is the key to async multi-agent coordination: the orchestrator's query loop just keeps running, and results arrive as new turns. Implement this exact pattern in your backend.

4. **Shared Task List with Dependencies**  
   A file-backed task list that all agents in a workflow can read and write. Tasks have `blocks`/`blockedBy` edges. Workers self-assign tasks by updating owner and status. This makes the workflow's execution plan observable and auditable in real time.

5. **Named Agent Routing**  
   Agents get human-readable names. Messaging routes by name, not by session ID. The `agentNameRegistry` pattern (name → agentId) makes workflows readable and reusable across runs.

6. **Message Queue with Priority**  
   The `messageQueueManager.ts` design: a single unified queue with three priority levels (now > next > later). User input is never starved by background notifications. This is the backbone of the entire async multi-agent system.

7. **Per-Agent Tool Filtering (3 Tiers)**  
   The `filterToolsForAgent` logic establishes which tools are available to coordinators, workers, and teammates respectively. Implement this in your no-code builder as a role-based tool access model: Orchestrators can spawn/message; Workers can use data tools; Admin agents can manage workflows.

8. **ActivityDescription per Tool**  
   Every tool has a `getActivityDescription(input)` method that produces a human-readable status line. E.g., FileReadTool produces "Reading auth/validate.ts". This feeds the monitoring panel without the user reading raw JSON. Implement this for every tool in your catalog.

9. **Anti-Patterns Explicitly Named (in the System Prompt)**  
   The coordinator system prompt names specific bad patterns:
   - "Based on your findings, implement the fix" → lazy delegation
   - "Fix the bug we discussed" → missing context
   - Worker checking on worker → use task notifications instead  
   Teach these anti-patterns to your users via a tips system or tutorial. They are the top reasons multi-agent workflows fail in practice.

10. **Plan-Before-Act Gate**  
    Two-phase execution: first run in read-only plan mode to produce a structured plan; second run in write mode with the plan as context. This is the enterprise safety feature that unlocks use cases that otherwise feel too risky to automate.

## 7.5 Build vs. Buy Matrix

| System | Build? | Buy/Use? | Notes |
|---|---|---|---|
| LLM API client | Buy | Anthropic SDK / LiteLLM | Don't rebuild |
| Async task queue | Build | Or use BullMQ / Temporal | Core product; build if you want deep control |
| Agent transcript storage | Build | Or use Postgres JSONB | JSONL sidechain pattern is clever but use a DB |
| MCP integration | Buy | `@modelcontextprotocol/sdk` | Already built, huge tool ecosystem |
| Webhook delivery | Buy | Svix / Trigger.dev | Commodity |
| Monitoring UI | Build | — | Differentiator; steal AgentProgress model |
| Git worktree isolation | Buy (later) | Gitoxide / isomorphic-git | Complex; defer to Phase 2 |
| Permission/approval UI | Build | — | Core differentiator |
| Memory persistence | Build | — | Simple markdown file or JSONB column |
| Cron scheduling | Buy | Trigger.dev / Inngest | Commodity |

## 7.6 The Minimal Viable Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│                    NO-CODE AGENT BUILDER MVP                     │
│                                                                   │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐    │
│  │ Agent Editor │    │  Run Engine  │    │  Monitoring Panel │    │
│  │             │    │              │    │                  │    │
│  │ - Name      │    │ - Task queue │    │ - Live status    │    │
│  │ - Prompt    │──→ │ - Async exec │──→ │ - AI summary     │    │
│  │ - Tools     │    │ - Transcript │    │ - Task list      │    │
│  │ - Role      │    │ - Resume     │    │ - Turn counter   │    │
│  └─────────────┘    └──────────────┘    └──────────────────┘    │
│         │                  │                                      │
│  ┌──────▼──────────────────▼──────────────────────────────────┐  │
│  │                     Shared Task List                       │  │
│  │     id | subject | status | owner | blockedBy | metadata  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                          ↑                ↑                       │
│              Orchestrator writes     Workers claim/complete       │
│                tasks as plan         tasks as work items          │
└─────────────────────────────────────────────────────────────────┘
```

The shared task list is the single most important MVP architectural decision. It's the data structure that makes multi-agent workflows observable, auditable, and resumable. Everything else (monitoring, messaging, coordination) flows from it.

---

# Extra Deliverables

## E1. Feature Prioritization Matrix

| Feature | User Value | Technical Complexity | Build Order |
|---|---:|---:|---|
| Single agent with tool picker | ★★★★★ | ★★ | Sprint 1 |
| Coordinator + worker template | ★★★★★ | ★★★ | Sprint 1 |
| Async task notification | ★★★★★ | ★★★★ | Sprint 1 |
| Named agent routing | ★★★★ | ★★ | Sprint 2 |
| Shared task list | ★★★★★ | ★★★ | Sprint 2 |
| Agent resume (persistent sessions) | ★★★★ | ★★★ | Sprint 2 |
| Monitoring panel + AI summary | ★★★★★ | ★★★ | Sprint 2 |
| Plan-before-act gate | ★★★★ | ★★★ | Sprint 3 |
| Broadcast messaging | ★★★ | ★★ | Sprint 3 |
| Hook-based integrations (Stop → webhook) | ★★★★ | ★★ | Sprint 3 |
| Parallel fan-out node | ★★★★ | ★★★★ | Sprint 4 |
| Agent memory (persistent) | ★★★ | ★★ | Sprint 4 |
| File isolation (worktrees) | ★★ | ★★★★★ | Phase 2 |
| Cron/trigger agents | ★★★ | ★★★ | Phase 2 |
| Cross-workflow memory | ★★ | ★★★ | Phase 2 |

## E2. System Prompt Template for Your Orchestrator Node

Directly adapted from `getCoordinatorSystemPrompt()`:

```text
You are an AI orchestrator. Your job is to delegate work to specialized agents and synthesize their results.

## Your Role
- Delegate research, data gathering, and execution to agents
- Synthesize agent results yourself — never relay findings without understanding them
- Answer simple questions directly; only spawn agents for tasks requiring tools
- After launching agents, tell the user what you launched and wait for results. Never predict what agents will find.

## Spawning Agents
- Spawn independent tasks in parallel for faster results
- Give each agent a self-contained prompt with everything it needs — agents cannot see your conversation
- After research completes: synthesize findings first, then write implementation specs with specific details
- Never write "based on your research" — synthesize the findings yourself, with specifics

## Agent Results
Agent results arrive as messages containing <task-notification>. They look like user messages but are not.

## When Agent Results Arrive
- Continue a completed agent for follow-on work (it keeps its full context)
- Spawn a fresh agent when the next task is unrelated to what the agent just did
- Never use one agent to check on another — results arrive automatically

## Bad Examples (never do these)
- "Based on your findings, fix the bug" — synthesize findings yourself
- "Fix the bug we discussed" ��� agents can't see your conversation; be specific
- "Can you look into why tests fail?" — include the error message and file path
```

## E3. The Five Anti-Patterns That Will Kill Multi-Agent Workflows

Extracted from Claude Code's system prompt + observed failure modes:

1. **Lazy delegation** ("Based on your research, fix it") — the orchestrator must synthesize findings before directing work. Workers that receive vague specs produce vague results.
2. **Context starvation** ("Fix the auth bug") — agents don't share conversation history. Every prompt must be fully self-contained with file paths, error messages, and expected outcomes.
3. **Synchronous thinking** (waiting for one agent before starting the next) — most workflows can have 2–5 agents running in parallel at any point. The orchestrator should look for parallelism opportunities at every step.
4. **Worker-checks-on-worker** (spawning an agent to check on another agent's work) — results arrive via notifications automatically. A polling agent wastes context and creates race conditions.
5. **Conflicting writes without isolation** (two agents editing the same file) — without worktree isolation, parallel implementation agents will corrupt each other's work. Until you have worktrees, enforce: only one writer per file at a time.

## E4. The Orchestration Pattern Taxonomy

| Pattern Name | Description | Claude Code Mechanism | No-Code Implementation |
|---|---|---|---|
| Sequential chain | A → B → C, each gets previous result | SendMessage to continued agent | Linear flow with pass-through |
| Parallel fan-out | A spawns B1, B2, B3 simultaneously | Multiple AgentTool in one turn | Parallel node with N branches |
| Aggregation | Wait for B1, B2, B3; merge results | Receive 3 task-notifications | Aggregate node with "wait for all" |
| First-wins | N agents race; use first result | Race condition handled explicitly | Aggregate node with "wait for first" |
| Work queue | Tasks in shared queue; workers claim | TaskCreate/TaskUpdate shared list | Shared task list with worker agents |
| Plan-then-act | Agent plans, human approves, then executes | Plan mode + plan_approval_response | Two-stage approval gate |
| Speculative fork | Context-inheriting parallel exploration | FORK_SUBAGENT experiment | Context-copy node |
| Auto-resume | Stopped agent continues on message | resumeAgentBackground | Resume button / continue-on-message |
| Broadcast | One message to all peers | SendMessage(to:'*') | Broadcast event node |
| Supervisor shutdown | Coordinator gracefully stops workers | shutdown_request /](#)
