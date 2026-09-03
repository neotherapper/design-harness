---
title: Gate layer
date: 2026-09-03
question: A Component entity on Temper versus a TypeScript guarded state machine, what each guarantees, costs, and how it churns.
---

## Summary

A `Component` gate needs one property: `Publish` is unreachable unless three validator flags are
true, and only validator credentials can set those flags. Temper delivers this as a kernel-enforced
guard plus server-side identity resolution, backed by a formal cascade (SMT satisfiability, exhaustive
bounded model checking, deterministic simulation, shadow-tested hot swap) and a durable event journal,
at the cost of a Rust toolchain, a Postgres-backed multi-crate service, and a project that states its
own API surface is not frozen. A TypeScript guarded state machine (XState v5) gives the same guard
shape, a pure function returning true or false, for one npm dependency, but supplies none of the
identity resolution, formal proof, journal, or self-approval protection on its own; each has to be
hand-built and kept correct by discipline rather than by the runtime. For a four-state, three-flag
machine the state space is small enough that an exhaustive TypeScript test suite can get equivalent
reachability coverage; what it cannot get for free is a proof that runs automatically on every spec
change, derived from the same struct that executes, composed with credential-bound identity and an
audit journal in one pipeline. The zygos pipeline that already ran this exact shape hit three concrete
defects (Text truncation, parameter-name exactness, Cedar principal shape) and had its auth model
rewritten once between two commit pins two weeks apart. That churn number should weigh most here.

## What the gate must guarantee

`design-harness/CLAUDE.md`: the must-have property is that "an agent's component cannot become
published unless it uses the system's design tokens and passes the system's accessibility and
guideline checks," and the gate is "a `Component` entity whose `Publish` transition requires validator
flags that only validator credentials can set. This is what makes non-conformance unreachable at
publish." The ceiling is stated in the same file: "A guard checks presence and shape, never truth
(zygos ADR-0001)."

Three separate guarantees, none of which the same mechanism proves alone:

1. **Unreachability.** No call sequence can produce a `Published` component with any flag false. A
   claim about the transition table, not about the validators' correctness.
2. **Credential separation.** The identity setting a flag is cryptographically distinct from the
   identity that authored the component, enforced by whoever resolves identity from a request, not by
   a header the caller fills in.
3. **Presence, not truth.** The gate checks the flags are true; it cannot check a validator actually
   ran, or ran against the artifact being published.

What each option **gates** (blocks the illegal transition) is kept separate below from what it only
**reports** (records a fact something downstream must act on).

## Option A: Component on Temper (sketch, not a tested spec)

Modeled directly on zygos's already-running `HarnessSpec` (IOA, CSDL, and Cedar files listed in
Sources). Everything below is a sketch by analogy; it was not written into a real spec bundle, loaded
into a Temper server, or run through the verification cascade.

### IOA sketch

```toml
# SKETCH, not a tested Temper spec.
[automaton]
name = "Component"
states = ["Draft", "UnderReview", "Published", "Archived"]
initial = "Draft"
allow_indefinite_states = ["Draft", "UnderReview", "Published"]

[[state]]
name = "has_implementation"
type = "bool"
initial = "false"
[[state]]
name = "tokens_validated"
type = "bool"
initial = "false"
[[state]]
name = "accessibility_passed"
type = "bool"
initial = "false"
[[state]]
name = "stories_present"
type = "bool"
initial = "false"

[[action]]
name = "WriteImplementation"
kind = "input"
from = ["Draft"]
to = "Draft"
params = ["Code", "TokenUsage"]
effect = [{ type = "set_bool", var = "has_implementation", value = true }]
[[action]]
name = "SubmitForReview"
kind = "internal"
from = ["Draft"]
to = "UnderReview"
guard = [{ type = "is_true", var = "has_implementation" }]
# Split the way HarnessSpec's RecordFidelityReview was split: the platform has
# no set_bool_from_param effect, so each check gets a Passed/Failed action
# pair instead of one boolean parameter. Shown for tokens only; Accessibility
# and Stories get the same Passed/Failed pair, one per flag, omitted here.
[[action]]
name = "SetTokensCheckPassed"
kind = "input"
from = ["UnderReview"]
to = "UnderReview"
params = ["ValidatorRunId"]
effect = [{ type = "set_bool", var = "tokens_validated", value = true }]
[[action]]
name = "Publish"
kind = "internal"
from = ["UnderReview"]
to = "Published"
guard = [
  { type = "is_true", var = "tokens_validated" },
  { type = "is_true", var = "accessibility_passed" },
  { type = "is_true", var = "stories_present" },
]
[[action]]
name = "Revise"
kind = "input"
from = ["Published"]
to = "UnderReview"
params = ["Reason"]
effect = [
  { type = "set_bool", var = "tokens_validated", value = false },
  { type = "set_bool", var = "accessibility_passed", value = false },
  { type = "set_bool", var = "stories_present", value = false },
]
[[action]]
name = "Archive"
kind = "input"
from = ["Draft", "UnderReview", "Published"]
to = "Archived"
params = ["Reason"]
[[invariant]]
name = "PublishRequiresAllValidatorFlags"
when = ["Published"]
assert = "tokens_validated && accessibility_passed && stories_present"
```

### CSDL fields sketch

SKETCH, trimmed, on the shape of `zygos-commons/specs/model.csdl.xml`. A `Component` EntityType with:
`Id` (Guid, key), `Name` (String), `Code` (String), `TokenUsage` (String), `has_implementation`
(Boolean, default false), `tokens_validated` (Boolean, default false), `accessibility_passed`
(Boolean, default false), `stories_present` (Boolean, default false), `Status` (the `SpecStatus`-style
enum, default `Draft`), plus a `Temper.Vocab.AuthZ.CedarPolicy` annotation pointing at
`policies/component.cedar`. The boolean property names match the IOA state-variable keys
deliberately, because that is how zygos's runtime persists them (`model.csdl.xml` header: "Boolean
properties are named to match the IOA state-variable keys ... which is how the runtime stores them").

### Cedar sketch, which agent type may call which action

```
// SKETCH, on the shape of zygos-commons/policies/harness_spec.cedar and the
// credential-bound model in zygos ADR-0004.

permit(principal is Agent, action in [Action::"WriteImplementation", Action::"SubmitForReview"],
    resource is Component) when {
    resource.Status == "Draft" &&
    principal has agent_type && principal.agent_type == "author" &&
    principal has agentTypeVerified && principal.agentTypeVerified == true
};

// Each validator credential sets only its own flag pair, while UnderReview.
permit(principal is Agent, action in [Action::"SetTokensCheckPassed", Action::"SetTokensCheckFailed"],
    resource is Component) when {
    resource.Status == "UnderReview" &&
    principal has agent_type && principal.agent_type == "token-validator" &&
    principal has agentTypeVerified && principal.agentTypeVerified == true
};
// same shape for SetAccessibilityCheckPassed/Failed (agent_type == "a11y-validator")
// and SetStoriesCheckPassed/Failed (agent_type == "story-validator")

permit(principal is Agent, action in [Action::"Publish", Action::"Revise", Action::"Archive"],
    resource is Component) when {
    principal has agent_type && principal.agent_type == "publisher" &&
    principal has agentTypeVerified && principal.agentTypeVerified == true
};
```

Three separate validator credentials, not one shared role, because the must-have text says "validator
credentials" (plural): a leaked or misused token-validator key cannot also mark accessibility passed.

### What the zygos pipeline already proved, and what defects it hit

None of the above was run. zygos's own `HarnessSpec` pipeline, built to this exact shape, was run for
a third spec (DeepSeek Harness) end to end (ADR-0002 Addendum, 2026-08-27), and the guards and Cedar
policies "held throughout the seven review cycles without bypass." The run also hit three defects the
plan had not anticipated:

1. **Text field truncation.** A `Text`-typed field written through the platform's parametric
   `set_field` effect was truncated to its type's default length handling, fixed by switching to
   property-named params. A long-form `Component` field (validator findings, say) should be expected
   to hit the same surprise until checked against a real write.
2. **Parameter-name exactness.** Bound-action parameter keys must match the CSDL PascalCase property
   name exactly; a mismatch silently no-ops the write and returns the prior state, no error.
3. **Cedar principal shape.** `ReviseDraft`'s permit required `agent_type == "reviewer"`, not
   `"admin"`; two guesses from the action name alone both returned `AuthorizationDenied` before the
   policy file itself was read.

The gate did what it says: three flags, a `Publish` guard on all three, and a presence/judgment split
stated as a design constraint, not an accident. zygos ADR-0001, on the analogous HarnessSpec guard:
"It does not, and structurally cannot, check that ... the embodiment matches the spec's own Rules."

## Option B: Component in TypeScript (XState v5 sketch)

```ts
// SKETCH, not a tested implementation.
import { setup, assign, and } from 'xstate';

type Ctx = { tokensValidated: boolean; accessibilityPassed: boolean; storiesPresent: boolean };
type Ev =
  | { type: 'SUBMIT_FOR_REVIEW' }
  | { type: 'VALIDATOR_RESULT'; check: 'tokens' | 'a11y' | 'stories'; passed: boolean; callerRole: string }
  | { type: 'PUBLISH'; callerRole: string }
  | { type: 'REVISE'; callerRole: string };

export const componentMachine = setup({
  types: {} as { context: Ctx; events: Ev },
  guards: {
    allFlagsTrue: ({ context }) =>
      context.tokensValidated && context.accessibilityPassed && context.storiesPresent,
    isValidator: ({ event }) => event.type === 'VALIDATOR_RESULT' && event.callerRole.endsWith('-validator'),
    isPublisher: ({ event }) =>
      (event.type === 'PUBLISH' || event.type === 'REVISE') && event.callerRole === 'publisher',
  },
}).createMachine({
  id: 'component',
  initial: 'draft',
  context: { tokensValidated: false, accessibilityPassed: false, storiesPresent: false },
  states: {
    draft: { on: { SUBMIT_FOR_REVIEW: 'underReview' } },
    underReview: {
      on: {
        VALIDATOR_RESULT: {
          guard: 'isValidator',
          actions: assign(({ context, event }) => {
            if (event.type !== 'VALIDATOR_RESULT') return context;
            const key = event.check === 'tokens' ? 'tokensValidated'
              : event.check === 'a11y' ? 'accessibilityPassed' : 'storiesPresent';
            return { ...context, [key]: event.passed };
          }),
        },
        PUBLISH: { guard: and(['isPublisher', 'allFlagsTrue']), target: 'published' },
      },
    },
    published: { on: { REVISE: { guard: 'isPublisher', target: 'underReview' } } },
    archived: {},
  },
});
```

`isPublisher` and `isValidator` are string comparisons against a `callerRole` carried on the event.
XState's guard contract is minimal by design: "Guards should be pure, synchronous functions that
return either `true` or `false`" (stately.ai/docs/guards, fetched 2026-09-03). Nothing in the library
resolves who `callerRole` actually is; the machine trusts whatever value the caller sets. A
client-supplied field rather than a server-verified credential here is the same header-declared-
identity shape zygos ADR-0004 describes the older TemperPaw kernel using, the shape zygos's own Cedar
rules were rewritten to stop trusting.

**Cedar in TypeScript.** `@cedar-policy/cedar-wasm` exists on npm (version 4.12.0, unpacked 13.1 MB,
`npm view`, fetched 2026-09-03). Its source at commit `144048d44d9450106690a37b6a22203be2936d8c` of
`cedar-policy/cedar` (`cedar-wasm/src/lib.rs`) re-exports Cedar's own `ffi::is_authorized` and
`validate`, among other schema and policy-text helpers, as WebAssembly bindings, so the same Cedar
language and evaluator Temper uses can run inside Node or a browser. It does not bundle identity
resolution: `is_authorized` still needs a caller-supplied principal, action, resource, and context
object, and the README (read in full at the same pin) covers only bundler loading, silent on where
the principal's fields come from. Building what Temper's kernel does server-side (ADR-0004:
"authority is resolved from the credential by the kernel, never declared by the caller") is left to
whoever integrates cedar-wasm; skipped or done casually, cedar-wasm evaluates a policy correctly
against an untrustworthy input, the identical failure mode ADR-0004 was written to close.

**Persistence and rehydration.** XState v5 persists an actor's internal state with
`actor.getPersistedSnapshot()` and restores it with `createActor(machine, { snapshot: restoredState
}).start()` (stately.ai/docs/persistence, fetched 2026-09-03: "You can restore an actor to a persisted
state by passing the persisted state into the `state` option", while its own code sample on the same
line uses the `snapshot` key, which is what the sketch above uses). Spawned or invoked child actors are
restored recursively, but actions do not re-run: "Actions from machine actors will not be re-executed,
because they are assumed to have been already executed." The docs draw the line zygos's read of
Temper's journal draws, on the other side of it: "the persisted state represents the internal state of
the actor, while snapshots represent the actor's last emitted value," and for an audit trail: "Event
sourcing is preferred for this use-case," treated as a separate pattern to build, not something
persistence gives for free.

**Model checking.** `@xstate/graph` (stately.ai/docs/graph, redirected from `/docs/xstate-graph`,
fetched 2026-09-03) provides `getShortestPaths`, `getSimplePaths`, `getAdjacencyMap`,
`toDirectedGraph`, and `createTestModel` for "model-based testing," described on the page as
generating "test cases that cover all reachable states and transitions." This is bounded path
traversal and test generation, not a prover: for dynamic context "the state space can become
infinite. Use `stopWhen` or `limit` to bound the traversal." Nothing read claims it proves an
invariant across all reachable states the way Temper's Z3 (guard satisfiability, invariant induction)
and Stateright (exhaustive bounded BFS with counterexamples) levels do.

**What is lost, moving from Temper to this sketch:**

- **The formal cascade.** No SMT check, no exhaustive BFS, no deterministic simulation with injected
  faults, no shadow-tested hot swap comparing old and new tables before a live change ships.
- **Credential-bound principals.** Temper's kernel resolves `agent_type` and `agentTypeVerified`
  server-side from a hashed Bearer credential; a caller cannot declare a role via a header (ADR-0004, its own heading:
  "No credential resolves to `Admin`, so the old publish gate locks itself out"). Nothing in XState or cedar-wasm does this; it
  must be built, and zygos's own history shows it is easy to get wrong even with a working reference,
  since the auth model changed once already between two pins, needing a dedicated ADR.
- **The self-approval ban.** Temper ADR-0172, observed live (zygos temper spec, Embodiment row
  15), refuses to let the principal that triggered a denied action approve or deny the resulting
  pending decision. This is kernel-level protection on one specific workflow, not a general
  "cannot review your own work" rule; it does not itself stop one operator holding all three
  `Component` credentials from drafting, validating, and publishing alone (ADR-0004: "the separation
  is only as real as where the keys sit"). In TypeScript there is no equivalent; a self-approval check
  is an `if` statement, present only where someone remembered to write it.
- **The event journal.** Temper appends every transition to Postgres and rebuilds actor state from it
  on restart, observed directly in zygos's run ("state rebuilt from event journal via
  TransitionTable"). An XState snapshot is a point-in-time internal state, not an append-only history;
  its own docs name event sourcing as a separate thing to build for an audit trail.

## Comparison table

| Property | Temper | TypeScript (XState v5 + optional cedar-wasm) | Evidence |
|---|---|---|---|
| Publish unreachable without all 3 flags | Yes, IOA guard on every dispatch, plus a standing invariant | Yes, guard function on the transition, only when code routes through the machine | zygos temper spec; `harness_spec.ioa.toml`; stately.ai/docs/guards |
| Flags settable only by validator credentials | Yes, Cedar permit bound to a server-resolved `agent_type`, unforgeable by the caller | Only if a credential-to-role layer is built and secured; cedar-wasm evaluates a policy, it does not authenticate | zygos ADR-0004; `cedar-wasm/src/lib.rs` at `144048d` |
| Presence checked, not truth | Yes, named as a design limit | Yes, same unavoidable limit | zygos ADR-0001; design-harness `CLAUDE.md` |
| Formal proof of guard satisfiability / reachable states | Yes, Z3 SMT and Stateright exhaustive BFS, gated pre-deploy | No; `@xstate/graph` generates bounded test paths, warns unbounded context needs manual limits | zygos temper spec Verification strategy; stately.ai/docs/graph |
| Shadow-tested hot swap before a live change | Yes, blocks unless a fixed case suite matches old and new tables | No equivalent; a new guard ships as an ordinary code deploy | zygos temper spec, Loop and Verification strategy |
| Durable event journal, replay-based recovery | Yes, observed live across a process restart | No; snapshot restores internal state, not full history; docs point to event sourcing separately | zygos temper spec Boundaries and Embodiment; stately.ai/docs/persistence |
| Self-approval ban on denied actions | Yes, kernel-level, observed live | No structural equivalent | zygos temper spec Which axis, ADR-0172 |
| Runtime cost per gated action | About 1.4ms, dominated by the Postgres event append | Sub-millisecond in-process call; no durable write unless one is added | zygos temper spec Tradeoffs |
| Toolchain and process footprint | Rust nightly toolchain, 27 first-party crates, Postgres, single-process only | One npm dependency inside the harness's own process | zygos temper spec Tradeoffs and Boundaries |
| Stated API stability | "Version 0.1.0 ... API surface is not frozen"; `lifecycle: version-changing` | Not evaluated for a stability statement in this file | zygos temper spec frontmatter and Limits |

## Operational cost and churn

**Temper, in the project's own words, as already verified in the pinned zygos spec.** Status is
"pre-release": "Version 0.1.0 ... API surface is not frozen." Scale is stated plainly: "Single-node
architecture. The current runtime is single-process." Setup for the zygos run was `cargo build -p
temper-cli` on a pinned nightly toolchain, 2 minutes 19 seconds on a laptop; the service is 27
first-party crates, with Docker Compose available for Postgres, a Redis stub, ClickHouse, and OTEL,
though the zygos run itself used only the embedded libSQL store. `lifecycle: version-changing` is not
a formality: between the two commit pins zygos tracked (`2f43ece` to `ff0774f`, 18 upstream commits,
about two weeks), one new recovery mechanism (install-time rollback, ADR-0173) and one new
authorization rule (the self-approval ban, ADR-0172) landed upstream, bringing the recovery points
the zygos spec counts to three, and a durability caveat on non-transition writes was recorded
(ADR-0157). Separately, on the TemperPaw checkout zygos runs, for which the cited sources give no
dates, the auth model changed underneath zygos entirely, from header-declared identity to
credential-bound identity (ADR-0004), forcing zygos to discard a 147-file mechanical pin bump, rewrite
`harness_spec.cedar`, and reissue all three operational keys. That is the largest churn event in the
material read, and it hit an already-running pipeline, not a paper design.

**Keys and roles are a real operational surface, not a one-time setup step.** zygos's three-key model
(`tools/temper/README.md`; ADR-0004) requires an operator key to bootstrap two agent types and issue
two more credentials, each read from the shell and never committed. Rotation is manual: issue a new
key, then revoke the old credential's SHA-256 id over HTTP. ADR-0004 names its own gap directly:
"Today all three are exported in one shell. A single operator with all three keys can still draft,
review, and publish a spec alone."

**TypeScript adds one dependency, not a service.** XState v5 runs in whatever process already hosts
the harness: no separate server, no database migration, no toolchain beyond Node and a package
manager. `@cedar-policy/cedar-wasm` adds a WebAssembly binary (13.1 MB unpacked) if the team wants
Cedar's policy language, rather than a plain role check in a guard function. Either way, none of
Temper's guarantees above (the cascade, the journal, the credential resolution, the self-approval ban)
come with that dependency; each is a separate build, and its correctness rests on code review and test
coverage rather than a kernel that refuses to serve a spec that fails its own checks.

**Being honest about what formal verification buys here.** A `Component` machine with four states and
three boolean flags has a state space small enough (32 combinations of state and flags, fewer once
illegal combinations are excluded) that an exhaustive TypeScript test suite, enumerating every
(state, event) pair, can already cover the ground Temper's Z3 and Stateright levels cover for a
machine this size; it does not need a prover to be exhaustively checked. What the proof buys instead
is that the check runs automatically at spec load or change rather than only when someone remembers to
update a test file, that it is derived from the same struct the runtime executes rather than a second
model kept in sync by hand, and that it composes with credential-bound identity, a durable journal,
and a shadow-tested hot swap in one pipeline instead of three separately maintained concerns. Those
are process guarantees, not state-count guarantees, and they are the actual case for Temper here, not
"the state space is too big to check by hand," which for four states and three flags is not true.

## What I did not check

- Corrected after the 2026-09-03 fidelity review: one quote attributed to ADR-0004 was a splice of its heading and an unrelated phrase (replaced with the heading verbatim); the self-approval ban cited two Embodiment rows where only one supports it; the churn passage counted three new recovery mechanisms where the spec records one new one and a total of three, and tied the auth rewrite to a two-week window the sources do not date; one XState quote substituted `snapshot` for the source's `state`.

- Neither sketch above was built, loaded into a real Temper server, or run as an actual XState v5
  program; both are sketches by analogy, labeled as such throughout.
- Temper's own source was not re-fetched; every Temper-specific claim reuses the already-verified
  zygos spec pinned at `ff0774f572197a75987f3329b48553ae9f8b3c29`, per this task's instruction not to
  re-research Temper from scratch, and that spec's Limits carry forward unchanged. Whether Temper's or
  TemperPaw's auth model has moved again since the 2026-09-02 pin used by ADR-0004 was not checked;
  given the churn already observed between two pins two weeks apart, it should be assumed capable of
  having moved again.
- `@cedar-policy/cedar-wasm`'s generated JS/TS API surface (exact function names and casing after the
  wasm-bindgen build, e.g. whether `is_authorized` becomes `isAuthorized`) was not verified by
  installing or running the package, only its Rust source and README, at commit `144048d`. No existing
  integration between cedar-wasm and any state-machine library was found beyond that source.
- XState v5's actor model beyond guards, persistence, and `@xstate/graph` (spawning, invoking
  services, actor systems, the full events/context typing surface) was not read, and no performance
  measurement was taken for the TypeScript side; "sub-millisecond, in-process" is a reasoning claim
  about relative cost, not a benchmark. Licensing terms for XState and cedar-wasm were not checked.
- The three validator implementations themselves (a real token linter, accessibility checker, story
  presence check) are out of scope; this file covers only the gate consuming their output, per
  `design-harness/CLAUDE.md`'s own inner-loop/outer-gate split. zygos ADR-0003 (the re-pin
  fidelity-review process) sits alongside the ADRs read here but was not opened; it was not named in
  this task's source list and nothing here depends on it.

## Sources

| Source | Pinned URL or local path | Read |
|---|---|---|
| zygos temper spec | `/Users/georgiospilitsoglou/Developer/projects/zygos/docs/research/harnesses/temper.md`, itself pinned to `github.com/nerdsane/temper@ff0774f572197a75987f3329b48553ae9f8b3c29` | Full |
| zygos-commons HarnessSpec IOA, CSDL, Cedar policy | `zygos/zygos-commons/specs/harness_spec.ioa.toml`, `specs/model.csdl.xml`, `policies/harness_spec.cedar` | Full |
| zygos-commons APP.md | `zygos/zygos-commons/APP.md` | Full |
| zygos ADR-0001 | `zygos/docs/adrs/0001-guards-check-presence-not-truth.md` | Full |
| zygos ADR-0002, including the Phase 1 result Addendum | `zygos/docs/adrs/0002-temper-backed-curation-pipeline.md` | Full |
| zygos ADR-0004 | `zygos/docs/adrs/0004-credential-bound-curation-auth.md` | Full |
| zygos tools/temper README | `zygos/tools/temper/README.md` | Full |
| design-harness operating rules and research index | `design-harness/CLAUDE.md`, `.worktrees/research-landscape/docs/research/README.md` | Full |
| design-harness README and ROADMAP | `.worktrees/research-landscape/README.md`, `docs/ROADMAP.md` | Part, repo context only |
| XState v5 persistence | https://stately.ai/docs/persistence, fetched 2026-09-03 | Part, persistence and rehydration section |
| XState v5 guards | https://stately.ai/docs/guards, fetched 2026-09-03 | Part, guard declaration and evaluation order |
| XState v5 graph / model-based testing | https://stately.ai/docs/graph (redirected from `/docs/xstate-graph`), fetched 2026-09-03 | Part, path generation and model-based testing sections |
| Cedar cedar-wasm README | https://raw.githubusercontent.com/cedar-policy/cedar/144048d44d9450106690a37b6a22203be2936d8c/cedar-wasm/README.md | Full |
| Cedar cedar-wasm source | https://raw.githubusercontent.com/cedar-policy/cedar/144048d44d9450106690a37b6a22203be2936d8c/cedar-wasm/src/lib.rs | Full |
| npm registry metadata for @cedar-policy/cedar-wasm | `npm view @cedar-policy/cedar-wasm`, fetched 2026-09-03 | Full, metadata only, no adoption figures used |
