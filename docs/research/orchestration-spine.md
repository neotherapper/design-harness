---
title: Orchestration spine
date: 2026-09-03
question: How does a step block until a validator passes, in Mastra, the Claude Agent SDK, and Pydantic AI v2?
---

## Summary

All three frameworks can gate execution, but only two of them gate through a mechanism built for
exactly this purpose. Mastra workflows block a step from proceeding by looping it with `.dountil()`
or `.dowhile()` until a condition is true, and a thrown error inside a step fails that step and the
run; Mastra's scorers and evals are explicitly non-blocking and only report a score. The Claude Agent
SDK's `PreToolUse` hook can return `permissionDecision: "deny"` and that denial is enforced before any
other permission rule runs, including in `bypassPermissions` mode; this is a real deny, not a log line.
Pydantic AI v2 has two separate gates: core's `ModelRetry` / output-validator retry budget, which raises
`UnexpectedModelBehavior` and ends the run when exhausted, and the separate `pydantic-ai-harness`
package's `OutputGuardrail`, whose `block` outcome raises `OutputBlocked`. Durability differs sharply:
Mastra workflow suspend/resume persists a snapshot to configured storage; the Claude Agent SDK persists
conversation transcripts to disk (not a workflow-engine checkpoint); Pydantic AI splits this into a
narrow, explicitly-partial step-persistence capability and separate, vendor-integration "durable
execution" capabilities (Temporal, DBOS, Prefect, Restate). None of the three make evals a hard gate by
default; where a design-harness needs "cannot publish unless validators pass," that gate has to be built
on the deny/exception mechanisms, not on the scoring/eval mechanisms.

## Mastra

Source: docs pages under mastra.ai/docs, fetched 2026-09-03, and the mastra-ai/mastra repository pinned
to commit `4d26df887baa9bb7b898b557e697a7082dbc92d1`.

**Step chaining.** Steps are added with `.then()` and the workflow is closed with `.commit()`. Per the
control-flow docs: "Use `.then()` to run steps in order, allowing each step to access the result of the
step before it." `.branch()` picks exactly one path: "Only one branch executes based on which condition
evaluates to `true` first." `.parallel()` runs steps concurrently and fails hard on any error: "If any
parallel step throws an error, the entire parallel block fails." The docs recommend catching errors
inside a step and returning a typed success/failure result for downstream steps to filter on, rather
than relying on uncaught-exception propagation for `.then()` chains.

**Blocking on a validator.** The construct that actually blocks execution until a condition passes is
the loop, not `.then()`. The control-flow docs state: "Use `.dountil()` to run a step repeatedly until a
condition becomes true" and "Use `.dowhile()` to run a step repeatedly while a condition remains true."
A validator step can be wrapped in `.dountil(validatorStep, ({ inputData }) => inputData.passed)` so the
workflow does not advance past it until the validator's own output says so. To prevent an infinite loop
the docs say: "Use `iterationCount` to limit how many times a loop runs. If the count exceeds your
threshold, throw an error to fail the step." Throwing inside a step is therefore the hard-fail path,
and workflow run results carry a `status` of `success`, `failed`, `suspended`, `tripwire`, or `paused`,
with a `failed` result carrying an `error` field.

**Suspend and resume (durability).** A step suspends explicitly: "If the condition isn't met, the
workflow pauses and returns `suspend()`." Stated use cases: "Pause a workflow at any step to collect
additional data, wait for an API callback, throttle a costly operation, or request human-in-the-loop
input." The persisted state is a snapshot: "Suspension saves the current execution state as a snapshot.
Later, resume from a specific step ID to restore the exact captured state. Snapshots are stored in your
configured storage provider and persist across deployments and application restarts." Resume is
explicit and callable from outside the workflow process: `run.resume({ step: step1, resumeData: {...} })`
or by step ID string, and "If only one step is suspended, you can omit the step argument entirely."

**Durable agents.** This is a distinct feature from workflow suspend/resume, documented at
mastra.ai/docs/harness/durable-agents (fetched 2026-09-03). A durable agent "embeds the agentic loop
inside a workflow": "The workflow runs the same loop as `Agent.stream()` but each step can be memoized
and replayed." Streaming survives disconnects via a cache and pub/sub layer: "The loop publishes chunks
to a PubSub topic keyed by the run ID... An optional cache... stores published events so that a late
subscriber can catch up," and "The same client can reconnect by calling `observe()` with the `runId`."
Crash recovery is opt-in, not automatic: "durable agent runs are excluded from the generic boot-time
restart of active workflow runs," and enabling `recovery.durableAgents: 'auto'` "discovers every
registered durable agent with runs stuck in `running` status and re-drives them from the last persisted
snapshot," with the explicit caveat "Make sure your tools are idempotent before enabling automatic
recovery" because recovery re-executes tool calls. The repository's `examples/durable-agents/README.md`
(commit `4d26df887baa9bb7b898b557e697a7082dbc92d1`) confirms three concrete implementations:
`createDurableAgent` (Redis-backed resumable stream, no separate durable-execution engine),
`createEventedAgent` (adds the workflow engine, non-blocking start), and `createInngestAgent` (runs the
agentic loop on Inngest so it survives process restarts).

**Evals and scorers.** Per docs/scorers/overview (fetched 2026-09-03): "Scorers are automated tests that
evaluate Agents outputs using model-graded, rule-based, and statistical methods," and they "return
scores: numerical values (typically between 0 and 1)." Execution behavior is explicit and non-blocking:
"Live evaluations run in the background without blocking your agent responses or workflow execution."
Scorers attach either to an agent ("add built-in scorers to your agents to automatically evaluate their
outputs") or to individual workflow steps ("add scorers to individual workflow steps to evaluate outputs
at specific points in your process"), but in both cases they report a score; nothing in the fetched docs
ties a scorer result back into whether a step is allowed to proceed. Reaching the same conclusion, the
evals overview page states evals are "Scorers... plus quick checks, gates, and verdicts" as a broader
framework, but the only sentence about execution effect is the same non-blocking one.

**MCP.** Per docs/tools-mcp/mcp-overview (fetched 2026-09-03): "Mastra supports the Model Context
Protocol (MCP), an open standard for connecting AI agents to external tools and resources." `MCPClient`
connects to external MCP servers and lists their tools for an agent; `MCPServer` does the reverse:
"Use `MCPServer` to expose Mastra agents, tools, workflows, prompts, and resources to other
MCP-compatible systems." The docs also mention "tool approval workflows" for MCP-sourced tools, but the
fetched page did not give the approval mechanism's exact semantics (deny vs. ask vs. log), so that is
listed under "What I did not check."

**API stability.** The `@mastra/core` package at the pinned commit is versioned `1.65.0-alpha.1`
(`packages/core/package.json`), and the root workspace package is `mastra-turbo@0.1.11`
(`package.json`). No stability/versioning statement was found in the root `README.md` beyond the
Apache-2.0 / Enterprise (`ee/`) dual-license split. The alpha pre-release tag on `@mastra/core` is the
project's own signal that the core package's API is not yet stable.

## Claude Agent SDK

Source: docs pages under code.claude.com/docs/en/agent-sdk (the platform.claude.com and docs.claude.com
URLs given in the task redirect here with a 307), fetched 2026-09-03, and the
anthropics/claude-agent-sdk-typescript and anthropics/claude-agent-sdk-python repositories pinned to
`e3d162bdf035169d0ea44bb6314d6e8fc5c730fe` and `a8b1e285f97f8dbcb7b10226d74ba0d551b493f4` respectively.

**Hooks and PreToolUse.** Hooks are "callback functions that run your code in response to agent
events," and the docs list "Block dangerous operations before they execute" as a stated purpose, not
just logging. The event sequence: "a tool is about to be called (`PreToolUse`), a tool returned a result
(`PostToolUse`)... After performing any operations... your callback returns an output object that tells
the agent what to do: allow the operation, block it, modify the input, or inject context into the
conversation." The worked example is exact: a `PreToolUse` hook matched on `"Write|Edit"` "checks if the
file path targets a `.env` file, and returns `permissionDecision: "deny"` to block the operation." The
output schema nests under `hookSpecificOutput`: "For `PreToolUse` hooks, this is where you set
`permissionDecision` (`"allow"`, `"deny"`, `"ask"`, or `"defer"`), `permissionDecisionReason`, and
`updatedInput`." Priority among conflicting signals is explicit: "When multiple hooks or permission
rules apply, `deny` takes priority over `defer`, which takes priority over `ask`, which takes priority
over `allow`. If any hook returns `deny`, the operation is blocked regardless of other hooks." A second,
narrower hook (`PreModelSwitch`) "can block" a requested model switch before it happens.

**Where the deny sits in the full evaluation order.** The permissions doc lays out a six-step flow:
hooks, deny rules, ask rules, permission mode, allow rules, `canUseTool` callback. Hooks run first and
their deny is unconditional: "A hook can deny the call outright or pass it on... If any hook returns
`deny`, the operation is blocked regardless of other hooks," and this applies "even in `bypassPermissions`
mode." The bypass-mode section repeats this from the other direction: "Hooks still execute and can
block operations if needed," and lists hooks alongside deny rules and explicit ask rules as the three
things still evaluated before the bypass-everything step. A `PreToolUse` hook that returns `allow` does
not skip the deny/ask rules below it, and a hook allow cannot override a deny on a "critical path" `rm`/
`rmdir`. This makes `PreToolUse` the correct place for a design-harness gate that must hold regardless of
whatever permission mode the calling session happens to be in.

**Subagents.** Subagents are "separate agent instances that your main agent can spawn to handle focused
subtasks," created programmatically via the `agents` option, as markdown files under `.claude/agents/`,
or via the built-in `general-purpose` agent. Benefits stated in the docs: context isolation ("intermediate
tool calls and results stay inside the subagent; only its final message returns to the parent"),
parallelization, specialized instructions, and tool restriction ("A tool you leave out isn't in the
subagent's session at all: Claude works without it, with no permission prompt or error"). Subagents can
themselves be capped: `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` (default 3), `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`
(default 20), and a `maxBudgetUsd` spend cap that "refuses to spawn more subagents... stops background
subagents that are still running, and ends the query with the `error_max_budget_usd` result subtype."
Subagents inherit the parent session's permission mode by default, so a `PreToolUse` deny hook registered
at the top-level session applies to subagent tool calls too, unless a subagent's own `AgentDefinition.permissionMode`
overrides it (which it cannot do if the parent uses `bypassPermissions`, `acceptEdits`, or `auto`).

**Sessions and durability.** "A session is the conversation history the SDK accumulates while your agent
works... The SDK writes it to disk automatically so you can return to it later." This is transcript
persistence, explicitly not filesystem or workflow-state checkpointing: "Sessions persist the
conversation, not the filesystem." Sessions are stored as `~/.claude/projects/<encoded-cwd>/*.jsonl`.
`continue` finds the most recent session in the current directory; `resume` takes a specific session ID
and works "Restart your process. You captured the ID before shutdown and want to restore the
conversation"; `fork` copies history into a new session ID while leaving the original untouched. Cross-host
resume requires either a `SessionStore` adapter or manually moving the `.jsonl` file, because "Session
files are local to the machine that created them." There is no built-in workflow-engine-style durable
execution (no automatic re-drive of an interrupted tool call); the SDK's durability guarantee is "the
conversation transcript survives a restart," not "the run resumes exactly where an in-flight tool call
left off."

**Evals.** Nothing under the fetched hooks/permissions/subagents/sessions pages describes a scoring or
eval mechanism; the Claude Agent SDK's contribution to a design-harness pipeline is the deny gate, not a
graded eval. Any eval loop (a11y score, token-lint score) would have to be wired in externally and then
enforced back through a `PreToolUse` (or a custom check before the `Publish`-equivalent call), since the
SDK itself does not ship a scorer.

**API stability.** `claude-agent-sdk` (Python) is at `0.2.152` (`pyproject.toml` at the pinned SHA); the
TypeScript package's `package.json` did not expose a version field at the fetched path, but the docs
themselves repeatedly gate behavior by SDK version number (for example, "requires TypeScript Agent SDK
v0.3.149 or later," "SDK versions that bundle an older CLI still behave this way"), which is the
project's own evidence that behavior has changed release to release under a pre-1.0 scheme. The
TypeScript README states: "The Claude Code SDK is now the Claude Agent SDK. Please check out the
migration guide... for details on breaking changes," confirming breaking changes are an accepted part
of the release cadence.

## Pydantic AI v2

Source: pydantic.dev/docs/ai pages (ai.pydantic.dev 301-redirects there), fetched 2026-09-03; the
pydantic/pydantic-ai repository pinned to `b1ffb6e8d4e7f6e3fbf0753d78ebc35b93386472`; and the separate
pydantic/pydantic-ai-harness repository pinned to `913a175ec0fe4f86686509b1ae8b1619fb5a13a0`.

**Two packages, one name overlap to watch.** "Harness" here names a specific, separate PyPI package,
`pydantic-ai-harness`, not a mode of the core `pydantic-ai` package. Per the harness overview page: "The
core `pydantic-ai` package provides a lightweight harness with basic agent functionality. The
`pydantic-ai-harness` package extends this with 50+ additional capabilities for complex, long-running
tasks," and "The boundary between the packages is mechanical, not a maturity tier." The classifiers in
each repository's `pyproject.toml` at the pinned commits confirm a real maturity difference regardless:
core `pydantic-ai` and `pydantic-ai-slim` both declare `Development Status :: 5 - Production/Stable`;
`pydantic-ai-harness` declares `Development Status :: 3 - Alpha`.

**The capability primitive.** A capability is a single object passed into `Agent(..., capabilities=[...])`
that can bundle "Tools -- via toolsets or native tools, Lifecycle hooks -- intercept and modify model
requests, tool calls, and the overall run, Instructions -- static or dynamic instruction additions, Model
settings... Models -- static or adaptive model selection." Capabilities "can be always-on or loaded by
the model on demand... all compose, with each other and with your own." The fetched capabilities
overview page does not itself describe a step-blocking mechanism; that is implemented by two of the
capabilities described below (guardrails) and by core's output-validator retry budget, not by the
capability primitive in general.

**Guardrails: gate, with a real exception path.** `pydantic-ai-harness` ships `InputGuardrail`,
`OutputGuardrail`, and `ToolGuardrail` capabilities that wrap a guard callable. Per the guardrails docs
and confirmed against the pinned source: a guard returns one of five outcomes -- allow, block, replace,
retry, approve -- and "Blocking the output means the model already produced something you do not want
exposed, so raising forces the caller to decide what to do next." The source at the pinned commit
(`pydantic_ai_harness/guardrails/_exceptions.py`) defines this as a real exception, not a log record:

```
class OutputBlocked(GuardrailError):
    """Raised by [`OutputGuardrail`][pydantic_ai_harness.OutputGuardrail] when the final output fails validation."""
```

`GuardrailResult` (in `_shared.py`) is "the single result type every guard returns," with an `action`
field typed `Literal['allow', 'block', 'replace', 'retry', 'approve']`; not every outcome applies to
every guard type ("`retry` and `approve` are rejected by `InputGuardrail`"). Input-side blocking is
cheaper and graceful ("Blocking the input spends no tokens, so a graceful refusal is almost always
right") while output-side blocking raises and forces caller handling. This is a genuine gate: a `block`
verdict from an `OutputGuardrail` stops that output from reaching the caller as a normal result.

**Core retries: also a gate, on a fixed budget.** Independent of the harness package, core `pydantic-ai`
has `@agent.output_validator`-registered functions that can raise `ModelRetry`. Each raise "consumes one
unit of the run's output retry budget"; the docs state "The default retry count is 1 but can be altered
for the entire agent... a specific tool... or outputs." When the budget is exhausted, "the run raises
`UnexpectedModelBehavior` with message `'Exceeded maximum output retries (N)'`" -- an unhandled exception
that ends the run rather than returning an unvalidated result. Validation errors from tool-parameter
parsing and from structured-output parsing both funnel through this same retry-then-raise mechanism.

**Durable execution: split into two unrelated things.** Core `pydantic-ai` documents "durable execution"
as an optional capability layer, not core behavior: "these integrations are capabilities you attach to
an agent," officially covering Temporal, DBOS, and Prefect (shipped, co-maintained with vendor teams),
with Restate maintained separately against pydantic-ai's public interface, and Kitaru/Airflow as further
integrations. Separately, the `pydantic-ai-harness` package ships a narrower `StepPersistence` capability
whose own README is explicit about its limits: "`StepPersistence` records what an agent did at each
boundary, separate from whether the run can be safely resumed... It is not a full graph-state checkpoint.
Capability-state restore, workspace snapshots, and graph-node resume are out of scope and tracked
separately." It gives an append-only step-event trail, "continuable snapshots" with a `complete` /
`interrupted` state, and a tool-effect ledger keyed by `(run_id, tool_call_id)` so a crash mid-tool-call
can be flagged `unknown_after_crash` rather than silently assumed safe. For an actual restart-surviving,
resume-exactly-where-it-stopped guarantee, the project's own docs point to the Temporal/DBOS/Prefect
capabilities, not to `StepPersistence`.

**Evals.** The fetched pages did not surface a distinct "evals" or "scorer" concept comparable to
Mastra's; validation in Pydantic AI v2 is expressed as guardrails (harness package) and output validators
/ `ModelRetry` (core), both of which are gates by construction rather than a separate reporting layer.
This is listed under "What I did not check" below since a dedicated evals/observability page
(`pydantic.dev/docs/ai/evals/` or similar) was not opened.

**API stability.** Core `pydantic-ai` / `pydantic-ai-slim`: `Development Status :: 5 - Production/Stable`
in both `pyproject.toml` files at the pinned commit. `pydantic-ai-harness`: `Development Status :: 3 -
Alpha`, and its own docs state the consequence in the project's words: "While Pydantic AI Harness is on
0.x releases, the API may change between minor releases... These capabilities are tested end-to-end and
meant for production use, but their APIs may still move between minor releases (0.1 -> 0.2): renamed
parameters, changed defaults, restructured APIs, always with deprecation warnings where practical."

## Comparison table

| framework | step blocks on validator (how) | hook/guard can deny (how) | durable/resumable (how) | evals: gate or report | language | API stability note (project's own words) |
|---|---|---|---|---|---|---|
| Mastra | `.dountil()` / `.dowhile()` loops a step until its own output says the condition (e.g. validator passed) is true; a thrown error inside a step fails that step and the run | Not a separate mechanism from step failure: throwing inside a step, or a `.parallel()` step throwing, fails the block; no distinct "deny" primitive found | Workflow `suspend()` persists an execution snapshot to configured storage, resumable by step ID or step object, "across deployments and application restarts"; durable agents add a Redis-backed pub/sub stream, optionally on a workflow engine (Inngest) for process-restart survival | Report only: "Live evaluations run in the background without blocking your agent responses or workflow execution" | TypeScript | `@mastra/core` at pinned commit is versioned `1.65.0-alpha.1` |
| Claude Agent SDK | Not a workflow primitive; a `PreToolUse` hook or `canUseTool` callback runs before each tool call and can be looped by the calling code, but the SDK itself has no `.dountil()`-style construct | Yes: `PreToolUse` hook returns `permissionDecision: "deny"`; deny "takes priority" over allow/ask/defer from other hooks, and applies "even in `bypassPermissions` mode" | Session transcripts persist to `~/.claude/projects/<encoded-cwd>/*.jsonl` and are resumable via `resume`/`continue`/`fork`; this is conversation-history durability, explicitly "not the filesystem," and not an in-flight-tool-call checkpoint | No built-in eval/scorer construct in the fetched docs; would need to be built externally and enforced via a hook | Python and TypeScript | Python SDK at pinned commit is `0.2.152`; TS README: "The Claude Code SDK is now the Claude Agent SDK... breaking changes" |
| Pydantic AI v2 | Core: `@agent.output_validator` raising `ModelRetry` consumes a retry budget; exhausting it raises `UnexpectedModelBehavior` and ends the run. Harness: `OutputGuardrail` `block` raises `OutputBlocked` | Yes, two mechanisms: core's retry-exhaustion exception, and the separate `pydantic-ai-harness` package's guardrail `block`/`retry`/`approve` outcomes (input, output, and tool guardrails) | Core: optional durable-execution capabilities on Temporal, DBOS, or Prefect ("capabilities you attach to an agent"), officially "survive restarts and failures." Harness: a narrower `StepPersistence` capability that is explicitly "not a full graph-state checkpoint" | Not found as a distinct construct; validation is expressed as gates (guardrails, output validators), not scores | Python | Core: `Development Status :: 5 - Production/Stable`. Harness: `Development Status :: 3 - Alpha`, "APIs may still move between minor releases" |

## Implications for design-harness

A "cannot publish unless deterministic validators pass" pipeline needs a mechanism whose failure mode is
an unhandled exception or an explicit deny, not a score nothing reads. All three frameworks have such a
mechanism, but it lives in a different layer in each:

- In Mastra, the gate is a loop (`.dountil()`) around the validator step plus a thrown error on exhausted
  retries; the scorer/eval system that the vault's starting map treated as a natural validation loop is,
  per the primary docs, explicitly non-blocking, so scorers cannot be the enforcement point and would
  need to sit beside a `.dountil()`/throw pattern instead.
- In the Claude Agent SDK, the gate is a `PreToolUse` hook returning `permissionDecision: "deny"`, which
  is the one mechanism confirmed to hold even under `bypassPermissions`. Because subagents inherit the
  parent's permission mode, a deny hook registered once at the top level covers agent and subagent tool
  calls alike, which matters if a design-harness runs a component-builder subagent and a separate
  accessibility-reviewer subagent under one session.
- In Pydantic AI v2, two independent gates exist at different maturity levels: core's output-validator
  retry-then-raise (stable, `Development Status :: 5`) and the harness package's guardrails (alpha,
  `Development Status :: 3`, and a separate package to pin a version of). A design-harness betting on
  `pydantic-ai-harness` for its gate is betting on a pre-1.0 API with its own documented churn.

Durability claims need the same layer-separation: "durable" in Mastra's durable-agents docs, in the
Claude Agent SDK's session docs, and in Pydantic AI's `StepPersistence` docs each mean something
narrower than "resumes an in-flight validator run exactly where it crashed." Only Mastra's workflow
suspend/resume and Pydantic AI's Temporal/DBOS/Prefect integrations make an explicit restart-survival
claim in the primary source; the Claude Agent SDK's session persistence and Pydantic AI's
`StepPersistence` do not, and both say so in their own words.

This file makes no recommendation between the three frameworks.

## What I did not check

- Read in full: mastra.ai/docs/workflows/overview, /workflows/suspend-and-resume, /workflows/control-flow,
  /scorers/overview, /evals/overview, /agents/overview, /tools-mcp/mcp-overview, /harness/durable-agents
  (all fetched 2026-09-03); code.claude.com/docs/en/agent-sdk/hooks, /permissions, /subagents, /sessions
  (fetched 2026-09-03); pydantic.dev/docs/ai/harness/, /harness/guardrails/, /capabilities/overview/,
  /core-concepts/output/, /core-concepts/agent/, /durable-execution/overview/ (fetched 2026-09-03).
- Read in part (via `curl`/`gh api` against the pinned commit, not the whole repository): mastra-ai/mastra
  `examples/durable-agents/README.md`, `packages/core/package.json`, root `README.md`, root `package.json`;
  anthropics/claude-agent-sdk-typescript root `README.md`; anthropics/claude-agent-sdk-python
  `pyproject.toml`; pydantic/pydantic-ai-harness `pyproject.toml`, `guardrails/_exceptions.py`,
  `guardrails/_shared.py` (first ~60 lines), `step_persistence/README.md` (first ~80 lines);
  pydantic/pydantic-ai `pydantic_ai_slim/pyproject.toml` and root `pyproject.toml` (classifier section
  only).
- Not checked at all: Mastra Studio and its playground UI; `examples/agent-builder` and
  `examples/sandbox-deployer`; the exact semantics of MCP "tool approval workflows" mentioned in passing
  on the MCP overview page; Mastra's `mastra.ai/reference/agents/durable-agent` API reference page (only
  surfaced via search, not fetched); the Claude Agent SDK's Python-specific and TypeScript-specific
  full API reference pages (`/agent-sdk/python`, `/agent-sdk/typescript`) beyond what the hooks/permissions/
  subagents/sessions pages quoted inline; the Claude Agent SDK's `Workflow` tool for "dynamic workflows"
  (mentioned in the subagents page but not independently fetched); Pydantic AI's dedicated evals or
  observability documentation page, if one exists separately from Logfire integration; the Restate,
  Kitaru, and Airflow durable-execution integrations beyond the one summary sentence on the durable-execution
  overview page; and the TypeScript Agent SDK's own `package.json` version field, which did not resolve at
  the path tried.
- Could not verify from the starting map: the map's claim that Pydantic AI v2's Harness is a
  "core/Harness split" inside one package is not how the primary source describes it; the primary source
  describes two separate PyPI packages (`pydantic-ai` and `pydantic-ai-harness`) with a "mechanical, not
  maturity" boundary, which is a materially different claim from a single package with two internal
  layers. This file uses the primary-source framing, not the map's.

## Sources

| source | pinned URL | read |
|---|---|---|
| Mastra workflows overview | https://mastra.ai/docs/workflows/overview (fetched 2026-09-03) | part |
| Mastra suspend and resume | https://mastra.ai/docs/workflows/suspend-and-resume (fetched 2026-09-03) | part |
| Mastra control flow | https://mastra.ai/docs/workflows/control-flow (fetched 2026-09-03) | part |
| Mastra scorers overview | https://mastra.ai/docs/scorers/overview (fetched 2026-09-03) | part |
| Mastra evals overview | https://mastra.ai/docs/evals/overview (fetched 2026-09-03) | part |
| Mastra agents overview | https://mastra.ai/docs/agents/overview (fetched 2026-09-03) | part |
| Mastra MCP overview | https://mastra.ai/docs/tools-mcp/mcp-overview (fetched 2026-09-03) | part |
| Mastra durable agents | https://mastra.ai/docs/harness/durable-agents (fetched 2026-09-03) | part |
| Mastra repo: durable-agents example README | https://raw.githubusercontent.com/mastra-ai/mastra/4d26df887baa9bb7b898b557e697a7082dbc92d1/examples/durable-agents/README.md | full |
| Mastra repo: core package.json | https://raw.githubusercontent.com/mastra-ai/mastra/4d26df887baa9bb7b898b557e697a7082dbc92d1/packages/core/package.json | part |
| Mastra repo: root README.md | https://raw.githubusercontent.com/mastra-ai/mastra/4d26df887baa9bb7b898b557e697a7082dbc92d1/README.md | full |
| Mastra repo: root package.json | https://raw.githubusercontent.com/mastra-ai/mastra/4d26df887baa9bb7b898b557e697a7082dbc92d1/package.json | part |
| Claude Agent SDK: hooks | https://code.claude.com/docs/en/agent-sdk/hooks (fetched 2026-09-03) | full |
| Claude Agent SDK: permissions | https://code.claude.com/docs/en/agent-sdk/permissions (fetched 2026-09-03) | full |
| Claude Agent SDK: subagents | https://code.claude.com/docs/en/agent-sdk/subagents (fetched 2026-09-03) | full |
| Claude Agent SDK: sessions | https://code.claude.com/docs/en/agent-sdk/sessions (fetched 2026-09-03) | full |
| claude-agent-sdk-typescript repo: README.md | https://raw.githubusercontent.com/anthropics/claude-agent-sdk-typescript/e3d162bdf035169d0ea44bb6314d6e8fc5c730fe/README.md | full |
| claude-agent-sdk-python repo: pyproject.toml | https://raw.githubusercontent.com/anthropics/claude-agent-sdk-python/a8b1e285f97f8dbcb7b10226d74ba0d551b493f4/pyproject.toml | part |
| Pydantic AI: Harness overview | https://pydantic.dev/docs/ai/harness/ (fetched 2026-09-03) | part |
| Pydantic AI: guardrails | https://pydantic.dev/docs/ai/harness/guardrails/ (fetched 2026-09-03) | part |
| Pydantic AI: capabilities overview | https://pydantic.dev/docs/ai/capabilities/overview/ (fetched 2026-09-03) | part |
| Pydantic AI: output (core concepts) | https://pydantic.dev/docs/ai/core-concepts/output/ (fetched 2026-09-03) | part |
| Pydantic AI: agent (core concepts) | https://pydantic.dev/docs/ai/core-concepts/agent/ (fetched 2026-09-03) | part |
| Pydantic AI: durable execution overview | https://pydantic.dev/docs/ai/durable-execution/overview/ (fetched 2026-09-03) | part |
| pydantic-ai-harness repo: pyproject.toml | https://raw.githubusercontent.com/pydantic/pydantic-ai-harness/913a175ec0fe4f86686509b1ae8b1619fb5a13a0/pyproject.toml | part |
| pydantic-ai-harness repo: guardrails/_exceptions.py | https://raw.githubusercontent.com/pydantic/pydantic-ai-harness/913a175ec0fe4f86686509b1ae8b1619fb5a13a0/pydantic_ai_harness/guardrails/_exceptions.py | full |
| pydantic-ai-harness repo: guardrails/_shared.py | https://raw.githubusercontent.com/pydantic/pydantic-ai-harness/913a175ec0fe4f86686509b1ae8b1619fb5a13a0/pydantic_ai_harness/guardrails/_shared.py | part |
| pydantic-ai-harness repo: step_persistence/README.md | https://raw.githubusercontent.com/pydantic/pydantic-ai-harness/913a175ec0fe4f86686509b1ae8b1619fb5a13a0/pydantic_ai_harness/step_persistence/README.md | part |
| pydantic-ai repo: pydantic_ai_slim/pyproject.toml | https://raw.githubusercontent.com/pydantic/pydantic-ai/b1ffb6e8d4e7f6e3fbf0753d78ebc35b93386472/pydantic_ai_slim/pyproject.toml | part |
| pydantic-ai repo: root pyproject.toml | https://raw.githubusercontent.com/pydantic/pydantic-ai/b1ffb6e8d4e7f6e3fbf0753d78ebc35b93386472/pyproject.toml | part |
