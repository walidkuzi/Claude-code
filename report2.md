# Claude Code — Phase 3c & Phase 6 Analysis

> **Context**: This document is the continuation of `docs/phases-1-3-research.md`.
> Phase 3c delivers a deep design analysis of the Claude Code codebase — evaluating its
> architectural patterns, design quality, strengths, and weaknesses.
> Phase 6 delivers the MVP implementation roadmap: a concrete blueprint for building a
> **no-code AI agents builder** that learns from Claude Code's design.

---

## Phase 3c — Deep Design Analysis

### 3c.1 Architectural Patterns Worth Adopting

#### Pattern 1 — Agent-as-Markdown with YAML Frontmatter

Claude Code defines agents as Markdown files with YAML frontmatter. This is a deliberate UX
decision: the author writes a prose system prompt and attaches structured configuration in the same
file. It is immediately readable and editable without tooling.

```markdown
---
tools: [Read, Bash, Agent(worker)]
model: sonnet
permissionMode: acceptEdits
maxTurns: 50
memory: project
background: true
---
You are a code reviewer. Your job is to...
```

**Why it works**: Keeps the schema and the prose together. YAML frontmatter is parsed by Zod and
validated at load time — malformed files emit per-file errors without crashing the loader. A no-code
builder should expose this same mental model in its UI: a "definition form" (frontmatter) adjacent to
a "prompt editor" (Markdown body).

#### Pattern 2 — Tool-First Permission Model (Allow/Deny/Ask Tiers)

Permissions in Claude Code are expressed as **rule arrays** rather than a single toggle. Every
permission check has three possible outcomes:

- **Allow** — matching rule in `settings.permissions.allow[]`
- **Deny** — matching rule in `settings.permissions.deny[]`, blocks immediately
- **Ask** — matching rule in `settings.permissions.ask[]`, pauses agent for human approval

Rules support glob-style patterns:

```json
{
  "permissions": {
    "allow": ["Bash(git:*)", "Edit(src/**/*.ts)"],
    "deny":  ["Bash(rm -rf:*)", "WebFetch(example.com)"],
    "ask":   ["Bash(*)", "Edit(*.json)"]
  }
}
```

**Why it works**: Granular-but-composable. A no-code builder can render this as a visual rule
editor with "Allow / Deny / Ask" radio buttons per tool category, plus pattern fields.

#### Pattern 3 — Concurrency-Safe Tool Execution (StreamingToolExecutor)

The `StreamingToolExecutor` class partitions tool calls into **concurrency-safe** and
**exclusive** groups. Tools that implement `isConcurrencySafe()` can run in parallel
within the same turn; others are serialized. This is the mechanism behind Claude Code's
ability to read multiple files simultaneously in one turn.

```typescript
class StreamingToolExecutor {
  // Safe tools: Read, Grep, Glob, WebFetch, WebSearch → run in parallel
  // Unsafe tools: Bash, Edit, Write → run one at a time
  // Results are buffered and emitted in declaration order
}
```

**Why it works**: Maximizes throughput without race conditions. A no-code builder's execution engine
should implement the same pattern: classify each tool call as safe/unsafe, fan out the safe ones,
and serialize the rest.

#### Pattern 4 — Context Injection as Named Map (Not a Big String)

`getSystemContext()` and `getUserContext()` return `Record<string, string>` rather than a
concatenated blob. Each key names a context block (`gitStatus`, `claudeMd`, `currentDate`).
The system prompt assembler decides how to render each block into the final prompt.

This makes it easy to:
- Cache individual blocks independently
- Skip blocks (e.g., omit `claudeMd` for read-only agents with `omitClaudeMd: true`)
- Audit what is in the context

**A no-code builder should adopt the same pattern**: represent context as a typed map with named
slots, not a single string. Each slot has a source, a freshness TTL, and an injection order.

#### Pattern 5 — Coordinator Prompt as Self-Contained Specification

The coordinator system prompt is ~500 lines of structured prose, and it is the primary mechanism
for teaching orchestration behavior. It includes:
- An explicit table of phases (Research → Synthesis → Implementation → Verification)
- Good/bad prompt examples
- Rules for when to continue vs. spawn a worker
- A worked end-to-end example conversation

**What this teaches**: The model does not "know" how to coordinate — it is told, in detail, every
time. A no-code builder should let users author (or choose) a coordinator template that explains
their specific workflow structure.

#### Pattern 6 — Task Notification via XML in User Role

Worker results arrive as `<task-notification>` XML injected into the **user** message turn, not as
a separate channel. This is a clever use of the standard Anthropic API: it needs no new API
capability; any model sees it as a user message.

```xml
<task-notification>
  <task-id>agent-a1b</task-id>
  <status>completed</status>
  <result>Found the bug at line 42 of auth.ts...</result>
</task-notification>
```

**A no-code builder should adopt the same pattern** for result delivery — it decouples the
infrastructure from the model API and works today without function-call support.

#### Pattern 7 — Settings as Layered Override Chain

Settings in Claude Code merge across **five scopes** in order of increasing specificity:

```
built-in defaults
  ↓ MDM (enterprise policy) — ~/.claude/managed-settings.json
      ↓ user global         — ~/.claude/settings.json
          ↓ project          — .claude/settings.json
              ↓ local         — .claude/settings.local.json
                  ↓ CLI flags (one-shot override)
```

**Why it matters**: Enterprise operators can lock policy at the top; individuals can personalize
within their allowed surface. A no-code builder needs exactly this hierarchy: platform defaults →
organization policy → workspace → personal preferences → session flags.

#### Pattern 8 — Feature Flags as Dead-Code Elimination

Claude Code uses `feature('FLAG_NAME')` from `bun:bundle`. This evaluates at **build time**, not
runtime, so feature-gated code is entirely absent from the shipped binary when the flag is off. It
is used for in-progress features without incurring any runtime overhead.

**A no-code builder should use the same approach** for experimental features: build-time flags that
produce separate bundles rather than runtime `if` branches that grow indefinitely.

---

### 3c.2 Design Strengths

| Strength | Evidence |
|---|---|
| **Composable tool system** | `Tool` interface is pure (no class hierarchy), Zod input schema, React UI renderers co-located — easy to add new tools |
| **Async agent lifecycle is first-class** | Every sub-agent has a transcript, progress state, cost tracking, and cancellation — not an afterthought |
| **Permission system is auditable** | Every tool call produces a `PermissionDecision` with a `reason` — operators can audit via hooks |
| **Streaming by default** | All API calls use async generators; responses stream token-by-token; tool results stream as they complete |
| **Schema-first settings** | `SettingsSchema` is Zod, versioned, backward-compatible by policy (detailed docs at the top of the file), exported as JSON Schema for IDE support |
| **Analytics PII boundary** | `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` is a nominal type that forces developers to consciously annotate what they log |
| **Prompt cache optimization** | Tool list is sorted alphabetically and split from MCP tools to hit the server-side cache prefix consistently — proactively reduces cost |
| **Multiple LLM providers** | `APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'` — single-line env switch |
| **Resilient retry logic** | `withRetry` retries foreground requests up to 10 times with exponential backoff; background classifiers bail immediately to avoid capacity amplification |
| **Worktree file isolation** | Agents can run in isolated git worktrees — changes never touch main until merged; clean abort means no disk pollution |

---

### 3c.3 Design Weaknesses & Gaps

| Weakness | Details |
|---|---|
| **Bun runtime lock-in** | `bun:bundle` feature flags, `Bun.spawn`, `Bun.write` are pervasive. The codebase cannot run on Node.js without a major port. |
| **Terminal-only UX** | The React/Ink TUI is powerful for CLI users but creates a porting challenge for web/mobile. The agent core and the UI are not cleanly separated. |
| **No native graph of agent dependencies** | Task dependencies exist (`blocks`/`blockedBy`) but there is no runtime scheduler that automatically unblocks tasks when their dependencies complete — agents must do this manually via `SendMessage`. |
| **Agent prompts are not versioned** | CLAUDE.md files and agent definition `.md` files have no built-in version or migration system. Changing a prompt silently affects all future runs. |
| **No built-in evaluation framework** | There is a `TestingPermissionTool` for development but no production-grade evaluation or regression harness for agent behavior. |
| **Large main.tsx (4,683 lines)** | The main entrypoint contains CLI parsing, auth flows, feature-flag wiring, UI rendering, and hook setup. This is not a problem in production but makes the architecture hard to understand. |
| **Memory is unstructured Markdown** | Agent memory is a free-form Markdown file. There is no schema, no structured recall, and no deduplication. This works for short-term context but degrades over long-lived agents. |
| **No built-in rate limiting across agents** | In coordinator mode, many workers can launch in parallel — each makes independent API calls. There is a `max-budget-usd` flag but no token-rate limiter across the agent tree. |
| **MCP discovery is manual** | Users must add MCP servers by editing JSON config or running `/mcp add`. There is a nascent official registry but no automated MCP discovery from a project's existing tools. |
| **Inline HTML/Markdown not rendered in TUI** | The terminal renders Markdown visually (via Ink) but long structured outputs (tables, code blocks) can exceed the terminal viewport with no scroll history. |

---

### 3c.4 Patterns That Do Not Translate to No-Code Builders

The following patterns from Claude Code are tightly coupled to its CLI / developer context
and should **not** be directly copied into a no-code builder:

| Pattern | Why Not to Copy |
|---|---|
| **CLAUDE.md file scanning** | No-code users don't have a git repo and a cwd. Replace with a "workspace context" UI where users explicitly upload or link documents. |
| **Bash / shell command execution** | No-code agents shouldn't have unrestricted shell access. Replace with curated action blocks (API calls, DB queries, form submissions). |
| **tmux/iTerm2 swarm backends** | Terminal multiplexers make no sense outside a developer environment. Replace with a visual workflow canvas or a job queue. |
| **Manual JSONL transcript sidechain** | Works for CLI but brittle for a multi-tenant SaaS. Replace with a proper database-backed conversation store. |
| **Git worktrees for file isolation** | Only meaningful when agents manipulate actual code files. Replace with sandboxed execution environments (containers, e2b, etc.). |
| **MDM enterprise policy** | macOS/Windows MDM is not available to a web-based no-code platform. Replace with an org-level admin settings panel backed by your database. |

---

### 3c.5 Security Design Review

#### What Is Done Well

- **SSRF guard on HTTP hooks** (`ssrfGuard.ts`) blocks requests to private IP ranges and
  localhost by default.
- **HTTP hook URL allowlist** (`allowedHttpHookUrls`) and **env var allowlist**
  (`httpHookAllowedEnvVars`) limit what hooks can exfiltrate.
- **MCP server allowlist/denylist** (enterprise) prevents users from connecting to
  arbitrary MCP servers.
- **Filesystem sandbox** (`SandboxSettings.filesystem.allowWrite`, `denyWrite`) restricts
  which paths agents can write to.
- **Network sandbox** (`SandboxSettings.network.allowedDomains`) restricts outbound domains.
- **Worktree slug validation** prevents path traversal when creating isolated worktrees.
- **`DENY_ALL` permission mode** is available as a hard stop before the agent does anything.
- **`allowManagedPermissionRulesOnly`** and **`allowManagedMcpServersOnly`** allow
  enterprises to lock policy so user overrides are ignored.

#### What Is Missing

- **No audit log for agent decisions** — there is an analytics event stream, but no
  durable, tamper-evident audit trail of what each agent did and why it was permitted.
- **No secret scanning on tool outputs** — an agent could echo a secret key to stdout
  and it would be stored verbatim in the transcript.
- **Prompt injection not defended** — content fetched by `WebFetchTool` or read by
  `FileReadTool` goes directly into the context. There is no content filter or injection
  detector before model ingestion.
- **`bypassPermissions` mode has no time-boxing** — once the flag is set, all tool
  calls are auto-approved for the entire session.
- **Agent tokens can grow unbounded** — `maxTurns` is optional; `max-budget-usd` is
  a best-effort check. A runaway agent can exhaust quota before being stopped.

---

### 3c.6 Observability Design Review

Claude Code has sophisticated runtime observability for an in-process tool:

| Signal | Mechanism |
|---|---|
| **Startup profiling** | `startupProfiler.ts` — checkpoint timestamps from `main_tsx_entry` through to first render |
| **Tool usage analytics** | `logEvent()` on every tool call, permission decision, and agent lifecycle event |
| **Agent progress summaries** | `agentSummary.ts` forks the agent every 30s to generate a 1-2 sentence summary |
| **Token/cost tracking** | Per-model, per-session, per-agent breakdowns, persisted to disk |
| **Diagnostic logs** | `logForDiagnosticsNoPII` emits structured events that can be read via `/doctor` |
| **Heap dumps** | `/heapdump` command captures V8 heap snapshot |
| **Asciicast recorder** | Full TTY session recording |
| **OpenTelemetry headers** | `otelHeadersHelper` — exec a script to attach OTEL headers to all API calls |

**Gaps** for a production platform:
- No distributed tracing across agent-tree spans
- No SLO tracking (P99 latency, tool error rates)
- No real-time alerting when an agent tree exceeds budget or stalls
- Analytics uses internal Anthropic infrastructure (Statsig/GrowthBook/Datadog) — not
  portable to a standalone deployment

---

## Phase 6 — MVP Implementation Roadmap

> **Target**: A **no-code AI agents builder** that lets non-developers create, configure,
> and run multi-agent workflows through a visual interface — drawing directly on the design
> patterns validated in Claude Code.

---

### 6.1 Product Vision

The product allows a user to:
1. **Define agents** through a form UI (not code): name, role description, allowed actions,
   model, and memory scope.
2. **Connect agents into a workflow**: sequential chains, parallel fan-outs, or a coordinator
   + workers topology — without writing orchestration code.
3. **Trigger workflows** on demand, on schedule, or via webhook.
4. **Monitor runs** live: see what each agent is doing, what it produced, and what it cost.
5. **Review and approve** sensitive actions before agents proceed.

---

### 6.2 MVP Scope (Version 1)

#### In Scope

| Capability | Description |
|---|---|
| **Agent builder UI** | Name, description ("when to use"), allowed action categories, model, max turns, memory scope |
| **Single-agent runs** | Execute one agent against a text input; stream response |
| **Two-agent chain** | Sequential: Agent A → result → Agent B |
| **Coordinator + workers** | One coordinator agent, up to 5 parallel workers, result collection |
| **3 action types** | Web search, URL fetch, custom webhook call |
| **Webhook trigger** | POST to a URL to start a run |
| **Manual approval gate** | Pause on specific action types and wait for human approval |
| **Run history** | List of past runs per agent/workflow, view transcript |
| **Token + cost display** | Per-run token count and USD cost |
| **Workspace memory** | One shared Markdown context document per workspace |

#### Out of Scope for V1

- Visual workflow canvas (wire nodes together) — use form-based topology selection instead
- File upload / document ingestion
- Agent-to-agent messaging beyond coordinator pattern
- Cron scheduling
- Plugin marketplace
- Enterprise policy controls
- Custom models (V1: Anthropic API only)

---

### 6.3 System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Browser (Next.js)                              │
│                                                                             │
│  AgentBuilder UI  │  WorkflowDesigner UI  │  RunMonitor UI  │  Settings UI │
│                                                                             │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │ REST / WebSocket
┌─────────────────────────────────▼───────────────────────────────────────────┐
│                            API Server (Node / Bun)                          │
│                                                                             │
│  AgentRouter   │  WorkflowEngine   │  PermissionGate   │  RunStore         │
│  ToolExecutor  │  ContextBuilder   │  HookDispatcher   │  CostTracker      │
│                                                                             │
└──────────┬───────────────┬──────────────────────┬──────────────────────────┘
           │               │                      │
    ┌──────▼──────┐  ┌─────▼──────┐  ┌───────────▼──────┐
    │ Anthropic   │  │ Action     │  │ Persistence       │
    │ API         │  │ Runners    │  │ (PostgreSQL +      │
    │ (claude-*) │  │ WebFetch   │  │  Redis for queues) │
    └─────────────┘  │ WebSearch  │  └──────────────────┘
                     │ Webhook    │
                     └────────────┘
```

---

### 6.4 Core Data Models

#### Agent Definition

```typescript
type AgentDefinition = {
  id: string
  name: string                       // slug: "code-reviewer"
  displayName: string                // human: "Code Reviewer"
  description: string                // when-to-use, shown in UI
  systemPrompt: string               // the agent's full system prompt
  model: 'sonnet' | 'opus' | 'haiku' | string   // model alias or full ID
  maxTurns: number                   // default 20
  memoryScope: 'none' | 'workspace' | 'agent'
  allowedActions: ActionCategory[]   // ['web_search', 'web_fetch', 'webhook']
  webhookConfigs?: WebhookConfig[]   // pre-approved webhooks for this agent
  permissionMode: 'auto' | 'ask' | 'allow_all'
  owner: UserId
  workspaceId: WorkspaceId
  createdAt: Date
  updatedAt: Date
}
```

#### Workflow Definition

```typescript
type WorkflowDefinition = {
  id: string
  name: string
  topology: 'single' | 'chain' | 'coordinator_workers' | 'parallel'
  agents: WorkflowAgentRef[]
  triggers: TriggerConfig[]         // webhook, schedule (V2), manual
  maxBudgetUSD?: number
  workspaceId: WorkspaceId
}

type WorkflowAgentRef = {
  agentId: AgentDefinitionId
  role: 'coordinator' | 'worker' | 'step'
  position: number                  // order for chain topology
}
```

#### Run

```typescript
type Run = {
  id: string
  workflowId: WorkflowDefinitionId
  status: 'queued' | 'running' | 'completed' | 'failed' | 'paused'
  trigger: 'webhook' | 'manual' | 'schedule'
  input: string                     // initial prompt
  agents: RunAgent[]                // one per spawned agent
  totalInputTokens: number
  totalOutputTokens: number
  totalCostUSD: number
  startedAt: Date
  finishedAt?: Date
  approvalRequests: ApprovalRequest[]
}

type RunAgent = {
  agentId: AgentDefinitionId
  runId: RunId
  role: 'coordinator' | 'worker'
  status: 'pending' | 'running' | 'completed' | 'failed'
  transcript: TranscriptEntry[]     // stored as JSONB
  currentActivity?: string          // "Reading auth.ts" — live status
  summary?: string                  // AI-generated 1-2 sentence summary
  inputTokens: number
  outputTokens: number
}

type ApprovalRequest = {
  id: string
  runId: RunId
  agentRunId: string
  toolName: string
  toolInput: Record<string, unknown>
  requestedAt: Date
  reviewedAt?: Date
  decision?: 'approved' | 'denied'
  reviewedBy?: UserId
}
```

---

### 6.5 Turn Loop (Engine Core)

Adapted from Claude Code's `query.ts`, simplified for the web platform:

```typescript
async function* runAgentTurn(
  agent: AgentDefinition,
  messages: Message[],
  context: RunContext,
): AsyncIterable<TurnEvent> {
  let turnCount = 0

  while (turnCount < agent.maxTurns) {
    const response = await anthropicClient.messages.stream({
      model: resolveModel(agent.model),
      system: buildSystemPrompt(agent, context),
      messages: normalizeMessages(messages),
      tools: getToolDefinitions(agent.allowedActions),
    })

    for await (const chunk of response) {
      yield { type: 'chunk', chunk }
    }

    const message = await response.finalMessage()
    messages.push({ role: 'assistant', content: message.content })

    if (message.stop_reason === 'end_turn') break

    if (message.stop_reason === 'tool_use') {
      for (const toolUse of extractToolUses(message)) {
        // Check permission gate
        const permission = await checkPermission(agent, toolUse, context)
        if (permission === 'ask') {
          yield { type: 'approval_required', toolUse }
          permission = await waitForApproval(toolUse.id, context)
        }
        if (permission === 'deny') {
          messages.push(buildDenialResult(toolUse))
          continue
        }

        yield { type: 'tool_started', toolUse }
        const result = await executeTool(toolUse, agent, context)
        yield { type: 'tool_completed', toolUse, result }
        messages.push(buildToolResult(toolUse, result))
      }
    }

    turnCount++
  }

  yield { type: 'done', messages }
}
```

---

### 6.6 Coordinator Pattern Implementation

Adapted from `coordinatorMode.ts` and `runAgent.ts`:

```typescript
async function runCoordinatorWorkflow(
  workflow: WorkflowDefinition,
  input: string,
  context: RunContext,
): Promise<void> {
  const coordinator = getCoordinatorAgent(workflow)
  const workerPool: Map<string, RunningWorker> = new Map()

  const coordinatorMessages: Message[] = [
    buildSystemPrompt(coordinator, buildCoordinatorContext(workflow)),
    { role: 'user', content: input },
  ]

  for await (const event of runAgentTurn(coordinator, coordinatorMessages, context)) {
    if (event.type === 'tool_started' && event.toolUse.name === 'spawn_worker') {
      const worker = await spawnWorker(event.toolUse.input, workflow, context)
      workerPool.set(worker.id, worker)
      // Worker runs async — results delivered as task-notification injections
    }

    if (event.type === 'worker_completed') {
      // Inject <task-notification> XML into coordinator's user turn
      coordinatorMessages.push(buildTaskNotification(event.worker))
    }
  }
}

function buildTaskNotification(worker: RunningWorker): Message {
  return {
    role: 'user',
    content: `<task-notification>
<task-id>${worker.id}</task-id>
<status>${worker.status}</status>
<summary>${worker.summary}</summary>
<result>${worker.finalOutput}</result>
<usage>
  <total_tokens>${worker.totalTokens}</total_tokens>
  <tool_uses>${worker.toolUseCount}</tool_uses>
  <duration_ms>${worker.durationMs}</duration_ms>
</usage>
</task-notification>`,
  }
}
```

---

### 6.7 Action System (Tool Equivalent)

The no-code builder exposes **action categories** instead of raw tools. Each category maps
to one or more underlying tool implementations:

| Action Category | Underlying Implementation | Config Required |
|---|---|---|
| `web_search` | Search API (Brave / Google) | API key in workspace settings |
| `web_fetch` | HTTP GET with SSRF guard | Domain allowlist |
| `webhook_call` | HTTP POST to configured URL | Webhook URL + auth header |
| `read_context` | Read from workspace memory | None |
| `write_context` | Write to workspace memory | None |
| `ask_user` | Pause and surface question to user | None |

**Action definition schema:**
```typescript
type ActionDefinition = {
  name: string           // e.g. "web_fetch"
  displayName: string    // "Fetch URL"
  description: string    // shown to agent in tool list
  inputSchema: ZodSchema
  outputSchema: ZodSchema
  requiresApproval: boolean
  execute(input: unknown, context: RunContext): Promise<ActionResult>
  renderInput(input: unknown): React.ReactNode   // shown in UI
  renderOutput(output: unknown): React.ReactNode
}
```

---

### 6.8 Permission Gate

Adapted from Claude Code's three-tier permission model:

```typescript
type PermissionDecision = 'allow' | 'deny' | 'ask'

async function checkPermission(
  agent: AgentDefinition,
  toolUse: ToolUse,
  context: RunContext,
): Promise<PermissionDecision> {
  // 1. Check explicit deny rules (highest priority)
  if (matchesDenyRule(agent, toolUse)) return 'deny'

  // 2. Check if action category is allowed for this agent
  if (!isActionAllowed(agent.allowedActions, toolUse)) return 'deny'

  // 3. Apply permission mode
  switch (agent.permissionMode) {
    case 'allow_all': return 'allow'
    case 'ask':       return 'ask'
    case 'auto':      return inferFromContext(toolUse, context)
  }

  // 4. Check webhook pre-approval (pre-approved webhooks don't need confirmation)
  if (toolUse.name === 'webhook_call') {
    return isPreApprovedWebhook(agent, toolUse.input.url) ? 'allow' : 'ask'
  }

  return 'ask'
}
```

---

### 6.9 Context Builder

Adapted from Claude Code's `getSystemContext()` / `getUserContext()`:

```typescript
type ContextSlot = {
  key: string
  content: string
  source: 'workspace_memory' | 'user_input' | 'run_history' | 'system'
  cacheable: boolean
}

async function buildContextSlots(
  agent: AgentDefinition,
  run: Run,
  workspace: Workspace,
): Promise<ContextSlot[]> {
  const slots: ContextSlot[] = []

  // System slot: always injected
  slots.push({
    key: 'currentDate',
    content: `Today's date is ${new Date().toISOString().split('T')[0]}.`,
    source: 'system',
    cacheable: false,
  })

  // Workspace memory (if agent uses it)
  if (agent.memoryScope !== 'none') {
    const memory = await loadWorkspaceMemory(workspace.id, agent.memoryScope)
    if (memory) {
      slots.push({ key: 'memory', content: memory, source: 'workspace_memory', cacheable: true })
    }
  }

  // Run-specific context injected by workflow coordinator
  if (run.coordinatorContext) {
    slots.push({ key: 'task', content: run.coordinatorContext, source: 'run_history', cacheable: false })
  }

  return slots
}

function buildSystemPrompt(agent: AgentDefinition, slots: ContextSlot[]): string {
  const contextBlock = slots
    .map(s => `<${s.key}>\n${s.content}\n</${s.key}>`)
    .join('\n\n')

  return [agent.systemPrompt, contextBlock].filter(Boolean).join('\n\n')
}
```

---

### 6.10 Hook System (Simplified)

V1 supports a subset of Claude Code's 27 hook events, covering the most useful integration points:

| Event | Trigger | Use Case |
|---|---|---|
| `run.started` | Run begins | Send Slack notification |
| `run.completed` | Run finishes successfully | Post result to webhook |
| `run.failed` | Run fails | Alert on-call |
| `agent.tool_used` | Any tool call | Audit log |
| `approval.requested` | Agent needs human approval | Notify approver by email |
| `approval.timeout` | Approval not given in N mins | Auto-deny and continue |

Hook handlers in V1:
- **Webhook** — POST to a URL with event payload
- **Email** — send an email to configured recipients

---

### 6.11 Memory System

Simplified from Claude Code's three-scope model:

| Scope | Description | Storage |
|---|---|---|
| `none` | Agent has no persistent memory | N/A |
| `workspace` | Shared across all agents in workspace | DB: `workspace_memory.content` |
| `agent` | Private to this agent type | DB: `agent_memory.content` |

Memory is stored as Markdown text, editable from the UI. Agents read it at the start of each
run as a context slot. Agents write to it by calling the `write_context` action.

**V2 improvement**: Replace free-form Markdown with structured key-value pairs with timestamps
and provenance, enabling automatic deduplication and summarization.

---

### 6.12 Run Monitor UI

Real-time run visualization, delivered via WebSocket:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Run #4821   "Research competitors and write a brief"               │
│  Status: ● Running   Cost: $0.14   Duration: 1m 23s               │
└─────────────────────────────────────────────────────────────────────┘

  [Coordinator]  ● Running
    "Launching research workers..."

  [Worker 1]  ● Running
    Current: "Fetching https://competitor-a.com/pricing"

  [Worker 2]  ✓ Completed (32s)
    Summary: "Found 3 pricing tiers ranging $10-$99/month"

  [Worker 3]  ● Running
    Current: "Searching web for 'competitor C market share'"

  ┌─────────────────────────────────────────────────────────────────┐
  │  ⏸ Approval Required                                           │
  │  Worker 1 wants to POST to https://internal-api.example.com    │
  │  Payload: { "query": "pricing data" }                           │
  │  [ Allow ]  [ Deny ]                                            │
  └─────────────────────────────────────────────────────────────────┘
```

**Event stream fields** (sent via WebSocket per run):
```typescript
type RunEvent =
  | { type: 'agent_started'; agentId: string; role: string }
  | { type: 'agent_activity'; agentId: string; activity: string }
  | { type: 'agent_summary'; agentId: string; summary: string }
  | { type: 'agent_completed'; agentId: string; outputTokens: number }
  | { type: 'approval_required'; agentId: string; toolUse: ToolUse }
  | { type: 'approval_resolved'; approvalId: string; decision: string }
  | { type: 'run_completed'; totalCostUSD: number; durationMs: number }
```

---

### 6.13 Implementation Phases

#### Phase 6a — Foundation (Weeks 1–4)

- [ ] Project scaffold: Next.js frontend, Bun/Node.js API server, PostgreSQL + Redis
- [ ] Authentication: OAuth (GitHub/Google) + workspace management
- [ ] Agent builder UI: name, description, system prompt, model, max turns
- [ ] Single-agent runner: POST prompt → stream response → persist transcript
- [ ] Basic action types: `web_search`, `web_fetch`
- [ ] Run history list + transcript viewer
- [ ] Token/cost display per run

#### Phase 6b — Multi-Agent (Weeks 5–8)

- [ ] Coordinator + workers topology
- [ ] Task notification protocol (XML injection in user turn)
- [ ] Parallel worker fan-out with result collection
- [ ] Per-agent progress streaming (activity + AI summary every 30s)
- [ ] Worker context passing from coordinator
- [ ] `ask_user` action (approval gate)
- [ ] Approval UI in run monitor

#### Phase 6c — Integrations & Memory (Weeks 9–12)

- [ ] Webhook action (pre-approved URL list per agent)
- [ ] Webhook trigger (start a run via POST)
- [ ] Workspace memory (write + read context actions)
- [ ] Hook system: `run.completed` and `run.failed` → webhook handler
- [ ] Agent-level memory scope (isolated per agent type)
- [ ] Chain topology (sequential agents)
- [ ] Budget limit per run (`maxBudgetUSD`)

#### Phase 6d — Enterprise (Weeks 13–16)

- [ ] Organization-level settings (model allowlist, domain allowlist)
- [ ] Role-based access (viewer / editor / admin)
- [ ] Audit log (tamper-evident record of every tool call + permission decision)
- [ ] SSO (SAML/OIDC)
- [ ] Rate limiting across agent tree
- [ ] Run analytics dashboard (cost, duration, error rate per workflow)

---

### 6.14 Technology Stack Recommendations

| Layer | Choice | Rationale |
|---|---|---|
| **Frontend** | Next.js (App Router) + TypeScript | SSR for run history, RSC for live updates, Tailwind for UI |
| **API server** | Bun or Node.js (TypeScript) | Async generators for streaming, Zod for validation |
| **LLM client** | `@anthropic-ai/sdk` | Type-safe, streaming support, mirrors Claude Code |
| **Database** | PostgreSQL (via Drizzle ORM) | JSONB for transcripts, strong typing |
| **Queue** | Redis + BullMQ | Reliable job queue for agent runs |
| **Real-time** | WebSockets (Socket.io or native) | Live run monitor |
| **Auth** | NextAuth.js | OAuth + session management |
| **Schema validation** | Zod (throughout) | Matches Claude Code's approach |
| **Feature flags** | Statsig or LaunchDarkly | Gradual rollout, matches Claude Code's model |
| **Deployment** | Railway or Fly.io (V1) → AWS/GCP (V2) | Fast iteration |

---

### 6.15 Key Design Decisions Summary

| Decision | Recommendation | Rationale |
|---|---|---|
| **Agent definition format** | JSON (not Markdown) | No-code users don't write Markdown; a form UI maps cleanly to JSON |
| **Permission model** | Allow / Deny / Ask per action category | Simpler than per-tool rules; sufficient for V1 |
| **Context injection** | Named slots in DB, not file scan | No filesystem; context is managed explicitly |
| **Worker result delivery** | XML in user turn (copied from Claude Code) | Works with standard API today |
| **Memory** | Markdown text in DB per scope | Easy to edit, renders well; upgrade to structured later |
| **Transcript storage** | JSONB column per agent-run | Queryable; no sidechain files |
| **Streaming** | WebSocket from server + Anthropic streaming | Provides live token stream to UI |
| **Cost control** | maxBudgetUSD hard stop + per-run token display | Prevents runaway spend |
| **Topology V1** | Single, Chain, Coordinator+Workers (no canvas) | Canvas adds complexity; form-based is faster to ship |
| **Action system** | Curated categories (not raw tools) | No-code users should not configure arbitrary bash commands |

---

*End of Phase 3c & Phase 6 Analysis.*
*See `docs/phases-1-3-research.md` for the Phase 1–3 foundational research this document builds on.*
