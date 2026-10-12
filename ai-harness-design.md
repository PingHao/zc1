# AI Harness — System Design (v0)

*Design only; no implementation. Scope: an **agent harness** — the deterministic program wrapped around a foundation model to make it a useful agent. (If "harness" meant a batch *evaluation* harness instead, this doc's kernel, context, and tool layers still apply; the eval-specific layer would be designed separately.)*

---

## 1. What we're building and why

An AI harness is everything around the model that isn't the model: the agent loop, the context window management, the tools, the permission system, the memory, and the session infrastructure. The model predicts tokens; the harness owns everything else. It is the model's operating system.

**Thesis.** Agent quality is multiplicative: `capability = model × harness`. A frontier model in a weak harness underperforms a mid-tier model in a great harness on real work. Unlike the model, the harness is fully under your control — it is where reliability, safety, cost, and UX actually live.

**Primary target.** A terminal/desktop coding-and-general agent: a user delegates tasks in natural language; the harness drives a model through tool calls against a real machine (files, shell, browser, web) until the task is done, under a configurable permission policy. Interactive by default, headless when embedded in CI or scripts.

**Non-goals (v1).**
- Model training, fine-tuning, or RL — the harness consumes models, it doesn't produce them.
- Deterministic DAG orchestration (that's a workflow engine). This harness provides *ad-hoc autonomy*, not guaranteed schedules.
- Multi-tenant server product. Single-user, local-first, with a clean seam for a server later.

---

## 2. Design principles

1. **The model is untrusted input.** Every byte the model emits — text, tool names, arguments — is schema-validated and policy-checked before it reaches the real world. Nothing executes because the model "said so."
2. **Deterministic shell, stochastic core.** All control flow *outside* the model is ordinary, testable code: state machine, retries, budgets, stop conditions. The model chooses *what next*; it never controls *how the loop works*.
3. **Context is the scarcest resource.** Every feature is judged by its context cost. Prompts, tool schemas, and results compete for a fixed window; the harness budgets them explicitly.
4. **Append-only, event-sourced state.** The session is a log of events. The context, the UI, and crash recovery are all *projections* of that log. This buys replay, fork, resume, and audit almost for free.
5. **Errors are data, not exceptions.** Tool failures return to the model as structured results so it can self-correct. The harness retries only what is transient; it never crashes the loop on a model mistake.
6. **Interruptible and steerable.** The user can redirect or cancel at any point mid-run. The agent is a collaborator, not a batch job.
7. **One kernel, many shells.** Interactive REPL, IDE panel, headless CI run, and sub-agents are the same kernel behind different I/O adapters.

---

## 3. Bird's-eye architecture

```
┌─────────────────────────────────────────────────────────────────┐
│        UI / Clients   (terminal REPL · IDE · headless CI · SDK)  │
└──────────────▲──────────────────────────────────▲───────────────┘
               │ streamed events / control        │ user input
┌──────────────┴──────────────────────────────────┴───────────────┐
│                         Session Service                         │
│     sessions · append-only event log · resume/fork · budgets    │
├─────────────────────────────────────────────────────────────────┤
│                          Agent Kernel                           │
│   Agent Loop (state machine)                                    │
│   Context Assembler · Tool Registry · Permission Gate · Memory  │
├─────────────────────────────────────────────────────────────────┤
│    Model Layer   (providers · normalization · retries · cache)  │
├─────────────────────────────────────────────────────────────────┤
│    Tool Runtime  (built-ins · MCP · sandbox · result budgets)   │
└─────────────────────────────────────────────────────────────────┘
```

Modules and responsibilities:

| Module | Owns |
|---|---|
| **Agent Kernel** | The loop state machine, context assembly, stop conditions, interruption handling |
| **Model Layer** | Provider abstraction, message normalization, retries/failover, prompt caching, capability descriptors |
| **Tool Runtime** | Tool execution, concurrency, result truncation/formatting, sandboxing |
| **Permission Gate** | Policy engine, action classification, approval UX, audit events |
| **Memory & Context** | Compaction, retrieval, todo/notes, long-term memory store |
| **Session Service** | Event store, session lifecycle, resume/fork, cost accounting |
| **Extension System** | Hooks, skills, slash commands, MCP client management |
| **I/O Adapters** | REPL, IDE, headless, SDK — thin, interchangeable |

---

## 4. The core: the agent loop

The heart of the system is one state machine, run once per session:

```
session = load_or_create()
loop:
    ctx    = assemble_context(session)          # budgeted projection of the event log
    resp   = model.call(ctx, tools)             # retries/failover handled in Model Layer
    append(session, resp)                       # log assistant message + proposed tool calls
    calls   = validate(resp.tool_calls)         # schema check + permission gate
    for call in schedule(calls):                # parallel for read-only; serialized for mutations
        result = tool_runtime.execute(call)     # sandboxed, result-budgeted
        append(session, result)
    if resp.stop_reason == done or interrupted or budget_exhausted:
        break
```

Semantics:

- **Turn** = one user→assistant exchange (may contain many steps). **Step** = one model call plus its tool executions.
- **Tool batching.** The model may emit several tool calls per step. Read-only calls run in parallel; mutating calls serialize per resource (file-level locking); a global concurrency cap prevents stampedes.
- **Stop conditions.** Model signals completion; user interrupt; token/step/cost budget exhausted; wall-clock deadline; error threshold (N consecutive failed steps). Every stop produces a resumable state, never a dead session.
- **Interruption model.** User input arriving mid-step is *queued* and delivered at the next context assembly as a steering message (soft interrupt). Hard cancel unwinds the current step — in-flight tool calls are cancelled or run-to-completion by tool class — and the session remains resumable.
- **Crash recovery.** Since state = event log, a restart reconstructs the exact context and resumes from the last completed step. Side effects of the interrupted step are reconciled by re-reading environment state, not by trusting the log.

**Why this shape.** Alternatives — graph frameworks, planner/executor splits, predefined workflows — either add deterministic scaffolding the model doesn't need or remove determinism the harness needs. One loop + strong tools + disciplined context management has consistently beaten elaborate orchestration for open-ended tasks.

---

## 5. Context management

The context window is re-assembled fresh every step as a projection of the event log, in a fixed, cache-friendly order:

```
[system prompt]      constitution: identity, rules, tool guidance     (stable, cached)
[tool schemas]       only tools plausibly needed this session         (stable, cached)
[environment block]  cwd, git state, time, selection                  (small, always fresh)
[memory block]       project instructions + retrieved notes           (medium)
[task state]         todo list, decisions, open questions             (curated)
[conversation]       newest turns verbatim; older turns compacted     (the rest)
```

Budget rule of thumb: system + tools + environment ≤ 15% of the window; memory ≤ 15%; conversation gets the remainder. The assembler is **cache-aware**: stable blocks first, volatile blocks last, so prompt caching actually hits.

### Compaction

When the conversation exceeds ~70% of the window: summarize the oldest half into a structured **handoff note** (goal, current state, decisions made, open threads, key file paths) while keeping the most recent N turns verbatim. Compaction is itself a model call with its own prompt; its output is appended as a `compaction` event and the summarized span is hidden — never deleted, so replay stays exact. The handoff note is written to the scratchpad file too, so it survives further compaction.

### External memory — three tiers

1. **Task list / scratchpad** — a file the model itself maintains via a `todo` tool. Forcing the model to externalize its plan is the cheapest reliability win available; it also survives compaction.
2. **Session notes** — facts and decisions auto-persisted within a session; retrieved by recency + relevance.
3. **Long-term memory** — cross-session, project-scoped store (markdown files + optional vector index). Written explicitly by the model or distilled at session end. Retrieved via tools on demand, **not** auto-stuffed — auto-stuffing pays a permanent context tax for memories that are irrelevant to most steps.

### Sub-agents for context isolation

A sub-agent is a fresh session with its own window, a narrowed toolset, and a self-contained task brief. Only its final result returns to the parent. Use it for: broad searches (the 50-file sweep), independently parallelizable work, and context-heavy transformations whose intermediate bulk the parent never needs to see. This is the single most effective context tool in the design — it trades tokens for window space.

### Progressive disclosure

Skills and tool catalogs advertise a one-line description in context; full instructions or schemas load only when triggered. A large third-party tool surface is exposed as a `search_tools` meta-tool rather than a wall of schemas.

---

## 6. Tool system

A tool is five things: a **JSON schema** (for the model), an **implementation** (for the runtime), a **permission class** (for the gate), a **result budget** (truncation/formatting rules), and an **idempotence flag** (whether a retry after an unknown outcome is safe).

### Built-in set (small and sharp)

- **read / write / edit** — files; edit is exact-string replacement (safer than line numbers), read is paginated with line numbers.
- **glob / grep** — search is how the model navigates a codebase *without* loading it into context.
- **bash** — sandboxed shell; the escape hatch that makes all other tools optional.
- **web fetch / web search**
- **todo** — the task list the model maintains.
- **note** — write session/long-term memory.
- **agent** — spawn a sub-agent.
- **ask_user** — blocking question with structured options, for genuine decision points only.

### Ergonomics rules (this is prompt engineering in disguise)

- Results over budget → write to a file, return the tail plus the path; the model learns to `read` selectively.
- Errors return the *actionable* form: what failed, why, and the nearest valid alternative (e.g., "no exact match for old_string; here is the closest block").
- Explicit truncation markers ("… 400 more lines, use offset"), stable file references, line numbers on reads.
- **Concurrency:** read-only tools parallel by default; mutating tools serialize per resource; global cap.

### Extension: MCP as the universal tool protocol

External tool servers contribute namespaced tools (`mcp__github__create_issue`). The registry handles schema translation, namespacing, permission classification (default: ask), and result budgeting for external tools, exactly as for built-ins. Most capability growth should come from new tools, not new code paths.

---

## 7. Permissions & safety

The permission gate sits between validated tool calls and execution. Every call passes through: **schema check → policy check → (maybe) user approval → sandboxed execution → audit event**.

### Action classes (by blast radius and reversibility, not by tool name)

| Class | Examples | Default |
|---|---|---|
| **L0 observe** | read files, search, list | auto-allow |
| **L1 workspace-mutate** | write/edit inside the workspace | auto-allow in trusted projects (git makes it reversible) |
| **L2 environment-mutate** | bash, installs, config changes outside workspace, network writes | configurable: ask / allow-by-pattern / deny |
| **L3 irreversible** | deletes outside workspace, publishing, spending, credential access, force-push | always ask, per action; no "always allow" |

### Policy model

Rules match on tool + resource pattern with a decision (allow / ask / deny) and a scope (once / session / project). Precedence: CLI > session > project > user > defaults. Trust is anchored to directories: a newly opened project starts untrusted and earns trust explicitly.

### Sandboxing (defense in depth — the gate is for UX, the sandbox is for containment)

- **Filesystem:** workspace + declared scratch/cache dirs only; secrets (`.env`, SSH keys, credentials) masked from reads by default.
- **Process:** OS sandbox per platform (Seatbelt on macOS, namespaces + seccomp or containers on Linux); network egress policy tied to permission level; no ambient credentials in tool environments.
- **Command screening:** model-emitted bash is parsed and pattern-screened (destructive deletes, fork bombs, credential exfiltration) — not to silently block, but to *classify* it into the right permission class so the policy and the user can decide.

### Auditability

Every tool call, permission decision, approval, and prompt lives in the event log. The compliance story is one sentence: *replay the session.*

---

## 8. Model layer

- **Normalized interface** across providers (native tool-loop APIs, chat-completions-style APIs, local runtimes). Adapters translate messages, tool calls, stop reasons, and stream events into one internal protocol.
- **Capability descriptors** per model: context size, max output, tool-call support, vision, reasoning budget, cost. The kernel queries this and degrades gracefully — e.g., a model without native tool calls gets a strict-format escape hatch with defensive parsing.
- **Resilience:** retries with exponential backoff + jitter on 429/5xx; failover to a configured fallback model; streaming with heartbeats so the UI never looks dead; prompt caching via deterministic serialization of the cached prefix.
- **Routing (v2, optional):** per-role model selection — cheap/fast model for mechanical steps (summarize, triage), frontier model for planning and hard edits. Routing is explicit config with a default, never magic.

---

## 9. Reliability: the failure model

| Failure | Class | Harness response |
|---|---|---|
| Provider 429/5xx | transient | backoff, retry, failover |
| Malformed tool call | model | structured error back to model with repair hint; never a crash |
| Tool execution error | environment | returned as result data so the model self-corrects |
| Same failure repeated ≥ N times | model stuck | escalation ladder: warn → strategy nudge → suggest different approach → stop and report |
| Context overflow | capacity | emergency compaction; if impossible, end turn with a handoff note |
| Budget exhausted | resource | graceful stop with status; session resumable |
| Harness crash | software | event-log recovery on restart |

**Anti-loop detection** is first-class, not a timeout footnote: fingerprint `(tool, canonical-args)` over a sliding window; repeats climb the escalation ladder. Infinite loops are the classic harness failure mode and deserve their own state, observability, and tests.

**Defensive parsing everywhere:** a failed tool-call parse is a recoverable event (re-ask with the parse error), never an exception that kills the loop.

---

## 10. Sessions, observability, control

- **Event log (JSONL)** — every model response, tool call, result, permission decision, compaction, and user input. Source of truth; context and UI are projections. Enables resume, fork ("branch from step 12"), replay/audit, time-travel debugging, and later, trajectory evals.
- **Cost accounting** — tokens and dollars per step/turn/session, surfaced in the UI, with warnings before budget hard-stops.
- **Tracing** — spans for sub-agents and tool calls; a session viewer (even a minimal one) pays for itself: "why did the agent do that?" is most of support.
- **Headless/CI mode** — same kernel with a stdin/stdout adapter and a non-interactive permission policy (deny L2+ or use an allowlist); exit codes distinguish done / blocked / failed. An agent that can't run in CI can't be trusted anywhere.
- **Configuration** — layered defaults < user < project < session < CLI, with transparent precedence and a `print-config` escape hatch for debugging.

---

## 11. Extension points (priority order)

1. **Tools via MCP** — the big one; most new capability should be a new tool, not a new code path.
2. **Hooks** — user scripts at lifecycle events (pre/post tool call, on compaction, on stop) that can veto, modify, or log. This is how organizations inject policy without forking.
3. **Skills / slash commands** — packaged instructions plus optional scripts, progressively disclosed. Cheap and high-leverage.
4. **Sub-agent profiles** — predefined agent types (explorer, reviewer, test-runner) with fixed toolsets and prompts.
5. **I/O adapters** — REPL, IDE plugin, bot, CI.

---

## 12. Hard-won lessons to design for

- **Context rot is real:** model quality degrades non-linearly with long contexts. Aggressive compaction and sub-agent isolation are *correctness features*, not optimizations.
- **Tool results are prompts:** a truncated diff or vague error message changes model behavior more than any system-prompt tweak. Budget and format results as carefully as you write prompts.
- **Keep the loop boring:** every framework that let the model control control flow eventually clawed determinism back. The model lives *inside* a harness-owned state machine.
- **Permission fatigue is a security risk:** if everything asks, users auto-approve everything. Keep L0/L1 frictionless so L2/L3 prompts stay meaningful.
- **Most "model failures" are harness failures:** unclear tool errors, missing context, forgotten state. Fix the harness before blaming the model.
- **Memory: start with append-only + summarize + retrieve.** Graph memory adds failure modes faster than it adds capability.

---

## 13. Milestone slice

Each milestone ships end-to-end value; nothing is scaffolding-only.

1. **Skeleton** — single loop, one provider, read/grep/bash + permission gate (ask on L2), event log, REPL. This alone is a usable agent.
2. **Real work** — write/edit, todo list, compaction. Now it can do actual tasks on real codebases.
3. **Extensibility** — sub-agents, MCP, hooks, skills.
4. **Productionization** — headless/CI mode, budgets, model routing.
5. **Scale users** — long-term memory, session viewer, fork/resume polish.
