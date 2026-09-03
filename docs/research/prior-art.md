---
title: Prior art
date: 2026-09-03
question: What Kaelig's pipeline, anydesign, TypeUI skills, and Katagami actually enforce versus recommend.
---

## Summary

Across five pieces of prior art, only one (Katagami) has a mechanism that makes an incomplete
publish unreachable: a declarative guard on a state transition, checked by the runtime, not the
agent. Kaelig's eight-agent pipeline has real deterministic checks too (a grep for a forbidden
token prefix, `tsc`, lint, a scored threshold that hard-stops the pipeline), but they live inside a
pipeline the author wrote and controls, not a platform guarantee independent of agent cooperation.
anydesign and the TypeUI/awesome-design-skills registry are almost entirely prose: markers,
confidence levels, and "quality gates" that are checklist language inside a skill file, with no
build, hook, or transition that fails if the agent ignores them. Anthropic's `frontend-design` and
`webapp-testing` skills are pure instruction sets; the second gives an agent tools to write its own
checks but wires nothing itself. shadcn's MCP server is supply-side only: it lets an agent list and
install components, and constrains nothing about what the agent then builds. The pattern worth
carrying forward is Kaelig's: separate a rule an agent might skip from a script that verifies and
halts, and only call the second one enforcement.

## Kaelig's eight-agent pipeline

Source: Kaelig Deloumeau-Prigent, "Building design system components with agent teams,"
https://www.kaelig.fr/design-system-components-with-ai-agent-teams/, fetched 2026-09-03 in full
(raw HTML downloaded, quotes checked against the downloaded text, not against a summary).

Context, in the author's own words: a production Menu component for the Intuit Design System,
prompted by hand first, then generalized into an eight-agent pipeline across 16 versions.

### Agents and artifacts

Three phases, eight agents, each consuming only the artifacts named to it:

| Phase | Agent | Consumes | Produces | Exit criteria | Budget |
|---|---|---|---|---|---|
| Understand | Design Analyst | Figma file | `brief.md`, `figma-raw.json` | 12-point completeness checklist passes | 1 pass, flags gaps `[PENDING]` |
| Understand | Library Researcher | `brief.md`, dependencies | `component-rules.md` (CR-* mandatory, AR-* advisory) | all dependencies documented | 1 pass |
| Understand | Component Architect | `brief.md`, `component-rules.md` | `architecture.md` with handoff notes | all `[BLOCKING]` issues resolved via conversational gate | 1 pass plus conversational gate |
| Build | Code Writer | all three prior artifacts | TypeScript, CSS modules, barrel exports | compiles, zero `[TOKEN_MISMATCH]` | 1 pass plus compliance fix loop |
| Build | Accessibility Auditor | generated source | accessibility report (P1/P2/P3) | zero unresolved P1 | up to 3 remediation attempts |
| Build | Story Author | `brief.md`, `architecture.md`, a11y report, source | Storybook CSF stories with play functions | all variants covered, interaction tests pass | 1 pass plus up to 2 retries |
| Verify | Visual Reviewer | rendered stories, Figma reference | visual comparison report (9 dimensions) | all PASS or documented deviation | max 5 iterations, stops on diminishing returns |
| Verify | Quality Gate | all source files | quality gate report | TypeScript compile, lint, format all pass | 1 pass plus 1 retry |

Artifacts are files, not messages, and this is deliberate: "Agents don't talk to each other. They
read and write documents." A named, required file (`component-rules.md`, a mandatory pre-flight
load for Code Writer) fixed a documented failure where research the Library Researcher did
correctly never reached the Code Writer, because it lived only in a chat response the orchestrator
read but downstream agents never saw.

### Markers

`[PENDING]` and `[UNRESOLVED]` mark a data gap the Design Analyst will not guess at. `[BLOCKING]`
halts the pipeline for a human, for example "the spec lists checkmarks under both single-select and
multi-select sections, which behavior is intended?" `[CONCERN]` lets an agent proceed but records
its rationale. `[SUGGESTION]` is optional. `[HUMAN_GATE]` marks a class of decision the author
judges structurally resistant to automation (motion specs, screen-reader nuance). `[TOKEN_MISMATCH]`
and `[ARCH_DEVIATION]` are raised by the Code Writer's compliance check. `[UNRESOLVED_A11Y]` is what
the Accessibility Auditor writes instead of looping a third time on the same P1 signature.
`[DEGRADED_QUALITY]` marks output the pipeline let through below its quality threshold, with the
human's explicit consent.

### What is actually enforced versus instructed

Three things in this pipeline are deterministic checks with a real failure mode, verified directly
in the downloaded page text:

- A grep: "make the Quality Gate grep all CSS custom properties and immediately fail on any
  `--ids-` prefix," added after the author's account of 27 fabricated tokens shipping under a
  plausible-looking but nonexistent prefix.
- TypeScript compilation and lint/format, run by the Quality Gate, with one retry and a bail if the
  same error persists.
- A deterministic quality score: the author describes it as a deterministic penalty table producing
  the same score for the same input every time (REST-API fallback minus 0.30, each unresolved token
  minus 0.02), with no LLM judgment involved. Below 80 percent, "the pipeline hard-stops," and the
  post presents three options: fix the input, supply data manually, or proceed
  explicitly with `[DEGRADED_QUALITY]`. The scoring arithmetic is confirmed deterministic, but the
  post never states what performs the halt itself; it names "the orchestrator" only in a different
  context (arbitrating simultaneous writes to a state file), so the halt is enforcement as described,
  with the implementing mechanism unverified.

Everything else that reads as a rule is a prompt-layer instruction, with three tiers of durability
the author names explicitly: workarounds.md (an unvalidated observation log, "up to agent to read
and apply"), memory (standing instructions loaded into every agent's context, that an agent can
still ignore), and skills (agent-specific instructions the author calls "the hardened layer" since
they load into that agent's own system prompt before it can act, for example "MANDATORY: call
`mcp__ids__get_tokens` before writing any token reference"). This tracks how likely a rule is to be
followed, but none of the three tiers is a mechanism in the guard sense: nothing outside the model's
own context checks whether it complied before producing output. The author's own framing supports
this: he calls skill files "pre-flight checks," accurate for what they are but not evidence they are
enforced, since nothing external verifies the agent read them.

The conversational `[BLOCKING]` gate is a genuine human-in-the-loop stop, but it is invoked by agent
judgment (the Architect decides it cannot resolve something), not by a system that can detect every
case where the agent should have stopped and did not. Bail rules by agent are in the table above and
in the Table section below.

The author's own conclusion, quoted directly: "The pipeline is reliable – rules compound, handoffs
are structured, quality scores catch degraded input. But the breakthrough was making space for
humans." And on the limit of full autonomy: "The pipeline could follow rules but it couldn't
question."

## anydesign

Source: github.com/uxKero/anydesign, commit `1b478809c3ccfebae558d93e4cc719e27f18bc0a`
(`repos/uxKero/anydesign/commits/main`), `SKILL.md` and `README.md` read in full at that SHA.

anydesign works in the reverse direction: it does not build a component inside a design system, it
extracts a design system's spec from an image, a live site, or a Figma file, into `design.md` (a
7-section spec), `design-tokens.json` (W3C DTCG format, `$value`/`$type`), and an optional
`design-a11y.md` WCAG report.

The "don't invent tokens" rule is stated in the skill's Quality rules section as plain instruction:
"Invent tokens you didn't observe" is listed under a Don't heading, and Step 4's non-negotiable
output rules state "Inventing tokens is worse than saying 'not enough info.'" Confidence markers
(high/medium/low, shown as check, warning, and question marks in the source) are likewise an
instruction to the model about how to write its own output, not a check anything else evaluates.

One place comes closer to a mechanism: `scripts/lint_design_md.py`, a stdlib-only script the
SKILL.md instructs the agent to run against its own output. "After generating a design.md, ALWAYS
run the lint script before delivering," checking frontmatter completeness, that token references
resolve, that components declared in YAML have prose entries, and that Do's/Don'ts is non-empty.
This is a deterministic, scriptable check: if a real orchestrator ran it and refused to hand the
file to a human on failure, it would be enforcement. As written, invoking it is itself an
instruction ("ALWAYS run") the agent could skip, and nothing outside the conversation calls the
script or blocks delivery on its exit code. The same is true of `verify_design.py`, a genuinely
useful deterministic diff against a live URL, but one the agent is told to run, not one wired in.

## TypeUI / awesome-design-skills

Source: github.com/bergside/awesome-design-skills, commit `f631a09b4fcc0166f2e2c1a8c81906ef680c57e8`
(`repos/bergside/awesome-design-skills/commits/main`), `README.md` read in full and
`skills/minimal/SKILL.md` read in full as the example, at that SHA.

This is a registry of design-language skill files (67 slugs: brutalism, glassmorphism, minimal,
a shadcn-styled one, and so on), each a folder of `SKILL.md` (agent instructions covering tokens,
component rules, accessibility constraints, quality gates) plus a companion `DESIGN.md`
(human-readable rationale and maintenance notes), the same spec-plus-companion split Katagami uses
for its own `DESIGN.md` next to its native spec. Distribution is a CLI, `npx typeui.sh pull <slug>`,
that writes the `SKILL.md` into a project's `.claude/` or `.cursor/skills/` directory.

`skills/minimal/SKILL.md`'s "Quality Gates" section, read in full, is four bullet points of prose:

> - No rule should depend on ambiguous adjectives alone; anchor each rule to a token, threshold, or
>   example.
> - Every accessibility statement must be testable in implementation.
> - Prefer system consistency over one-off local optimizations.
> - Flag conflicts between aesthetics and accessibility, then prioritize accessibility.

These are gates in name only. They instruct how the agent's own future output should read (favor
testable statements, avoid vague adjectives), not checks that run against a component the agent
produces. Nothing in the README or the CLI's documented behavior (pull, list, generate, `--dry-run`)
parses or lints a component against a skill's rules; the CLI's job ends at writing the skill file
into the target project. There is no CI hook, no build step, and no state machine anywhere in this
repository. All 67 skills follow the same generated template, visible from the
`TYPEUI_SH_MANAGED_START` marker in the one file read, so this finding likely generalizes across the
registry, though only one file was read in full.

## Katagami

Source: github.com/arni-labs/katagami, commit `5366ab8bbe7012a9768a37a12f869d7c9a0a3681` (default
branch is `master`, not `main`). `README.md` read in full; `design_language.ioa.toml` read directly
for the guard definitions (primary source for the mechanism claim, not the README's paraphrase);
`.agents/behaviors/review-agent/BEHAVIOR.md` read in part for how the review role is defined.

Katagami is a library of complete design languages (philosophy, tokens, rules, layout, guidance,
plus a rendered embodiment of about 15 canonical UI elements), built on Temper, described in the
README as "a policy-driven runtime where all state is expressed as communicating state machines."
The `DesignLanguage` entity's lifecycle is `Draft -> UnderReview -> Published -> Archived`.

### The mechanism: a declarative transition guard

Read directly from `design_language.ioa.toml`, the `SubmitForReview` action (`Draft -> UnderReview`)
carries a `guard` array of boolean and cross-entity checks, a structured list the runtime evaluates,
not prose. Reformatted below for readability (line-wrapped and reflowed from the source's single
long line; elided entries are named, not paraphrased):

```
guard = [
  { type = "is_true", var = "has_philosophy" },
  { type = "is_true", var = "has_tokens" },
  { type = "is_true", var = "has_rules" },
  { type = "is_true", var = "has_layout" },
  { type = "is_true", var = "has_guidance" },
  { type = "is_true", var = "has_embodiment" },
  ... has_compositions, has_thumbnail, has_design_md, has_valid_design_md,
      has_shadcn_export, has_shadcn_component_spec, has_shadcn_preview_shots,
      has_default_art_style ...
  { type = "cross_entity_state", entity_type = "File", entity_id_source = "embodiment_file_id",
    required_status = ["Ready", "Locked"] },
  ... four more File-readiness checks and one ArtStyle-status check ...
]
```

Every one of these is presence and shape, not truth: `has_tokens` is a boolean the pipeline sets
when a tokens section exists, not a check that the tokens are good tokens. This matches the pattern
zygos's own ADR-0001 names ("guards check presence, not truth") and this repo's CLAUDE.md states as
a working principle. If any listed condition is false, the transition is rejected outright by the
runtime; the agent's Python client calling `temper.action(..., 'SubmitForReview', {})` gets a
rejection, not a warning. This is enforcement in the strict sense the two-axis framework asks for:
a check that fails, on a transition, evaluated by something other than the agent.

The `Publish` action (`UnderReview -> Published`) repeats the same completeness guard and adds two
more: `is_true, quality_review_passed` and `is_true, has_published_assets`. `quality_review_passed`
is a boolean the guard trusts as an input; the guard does not evaluate quality itself.

### The review-quality job: enforced to happen, not enforced to be right

Who sets `quality_review_passed` is a separate agent job, described in the README's job table as
`quality_review`, skill `review-quality`, "Reviews and fixes embodiment against spec," and, in more
procedural detail, in `.agents/behaviors/review-agent/BEHAVIOR.md`: a reviewer that must record which
submission it is reviewing, examine artifacts directly rather than their metadata, record findings
with location and severity, and rule exactly once (pass, revise, or reject), running under its own
credential, "never a contributor's," per the file's "Never rule on your own work" section: "a
principal that authored the submission MUST NOT record findings or a verdict on it." This is
judgment, carried out by prose instructions, with no deterministic check that the verdict is correct.
What is structural is that the separation of roles is real: the reviewer's verdict is what the
`Publish` guard reads, and nothing else can set `quality_review_passed` to true. So the review's
occurrence and authority to gate publish are enforced; its correctness is not. This is the
"recover, not prevent" half of Axis A, resting on role separation rather than a truth check.

Embodiment rendering (the synthesize agent renders the language's roughly 15 canonical elements in a
cloud sandbox and visually verifies them at three viewport sizes) is agent work, not a mechanism;
what is mechanical is that `has_embodiment` and the paired File's `Ready`/`Locked` status must both
be true before `SubmitForReview` succeeds. `DESIGN.md` export is gated the same way: presence and,
separately, a validity flag are both booleans the guard checks, so an agent cannot submit a language
with a missing or structurally invalid `DESIGN.md`, but nothing in the guard confirms the
`DESIGN.md` accurately describes the language.

## Anthropic skills

Source: github.com/anthropics/skills, commit `41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f`, both
`skills/frontend-design/SKILL.md` and `skills/webapp-testing/SKILL.md` read in full at that SHA.

`frontend-design` is pure instruction to a model about aesthetic judgment: it asks the agent to name
the subject matter before designing, warns against five named "tells" of generated design (a
specific cream-and-terracotta palette, a near-black-plus-accent palette, a broadsheet-with-hairlines
look, the "SaaS-card kit," and template chrome such as tracked-out eyebrows or middle-dot-joined
meta strings), and prescribes a two-pass process: plan a token system, review the plan against the
brief for genericness, then build. It ends with a self-critique instruction rather than any check.
Nothing in the file names a script, a test, or a build step; it is entirely recommend, with zero
enforce.

`webapp-testing` is a toolkit for the agent to write and run its own Playwright scripts against a
local app: a decision tree (static HTML vs. dynamic, server already running or not), a helper script
`with_server.py` to manage server lifecycle, and a reconnaissance-then-action pattern (screenshot,
identify selectors, act). It gives the agent the means to build a real deterministic check (a
Playwright assertion that fails), but the skill itself wires no check: whether the agent runs the
script, what it asserts, and whether a failure blocks anything downstream are all left to the agent
and to whatever harness invokes this skill. Read together, these two skills sit at opposite ends of
the same axis as the other sources: one is prose about taste, the other is a toolkit for building a
mechanism, and neither is itself a mechanism.

## shadcn registry and MCP

Source: https://ui.shadcn.com/docs/mcp, fetched 2026-09-03 (page content only, the underlying
`components.json` spec and MCP server source not opened). The MCP server lets an agent browse
available components, search for specific ones, and install them directly from any registry
following the shadcn registry specification, including a private or internal one configured as a
URL in `components.json`. This is purely supply-side: it changes what an agent can fetch, not what
it is allowed to build with what it fetches. The page describes no validation, lint, or
accessibility check gating registry access or code written with an installed component; the only
constraints named are transport-level (URL correctness, auth for private registries). Kept short as
instructed.

## Table

| source | artifact contracts defined | enforced by mechanism (which) | recommended in prose (which) | bail/stop rule | human gate |
|---|---|---|---|---|---|
| Kaelig's pipeline | `brief.md`, `figma-raw.json`, `component-rules.md`, `architecture.md`, source files, a11y report, Storybook stories, visual report, quality report | grep for forbidden token prefix; `tsc`/lint/format; deterministic quality-score threshold (orchestrator hard-stop below 80%) | skill "pre-flight" instructions; memory (standing instructions); workarounds log; conversational `[BLOCKING]` judgment | a11y: same P1 twice; Quality Gate: same error after 1 retry; Visual Reviewer: 5 iterations or under 2% gain; quality score under 80% | yes, `[BLOCKING]` conversational gate, presented as risk tradeoffs, resumes only on explicit human "proceed" |
| anydesign | `design.md` (7 sections), `design-tokens.json` (DTCG), optional `design-a11y.md` | none automatically invoked; `lint_design_md.py` and `verify_design.py` are real scripts but agent-invoked, not hooked | "don't invent tokens," confidence markers, mandatory Open Questions and Do's/Don'ts sections, "ALWAYS run the lint script" | none stated beyond telling the user clearly on capture failure | no formal gate; user reviews output and chooses next step |
| TypeUI / awesome-design-skills | `SKILL.md` plus `DESIGN.md` per style, `index.json` registry manifest | none found in README or the one `SKILL.md` read in full | entire "Quality Gates" section is prose guidance for how the agent's own rules should read | none | none; CLI only writes files, no review step described |
| Katagami | `DesignLanguage` spec (5 sections plus embodiment, compositions, thumbnail, `DESIGN.md`, shadcn export artifacts) | `SubmitForReview` and `Publish` guards on the Temper state machine: presence/shape booleans plus cross-entity File-readiness checks, read directly from `design_language.ioa.toml` | the review-quality agent's judgment of whether the embodiment matches the spec (`BEHAVIOR.md`); the synthesize agent's writing of each spec section | transition rejected outright if any guard condition is false; no retry loop described in README | yes, structurally: a separate credentialed review role must set `quality_review_passed` before `Publish` can succeed; README also describes a human curator approving publish |
| Anthropic `frontend-design` | none (guidance only, no named output file) | none | entire skill: named anti-patterns, two-pass plan-then-build process, self-critique instruction | none | none |
| Anthropic `webapp-testing` | none (a toolkit, not a spec format) | none wired by the skill itself; gives the agent means to build a Playwright check | decision tree for choosing an approach; run helper scripts with `--help` first | none | none |
| shadcn MCP/registry | registry item schema (not examined beyond the docs page) | none described | none beyond correct configuration | none | none |

## Patterns worth carrying forward

- Separate a script's exit code from an instruction to run it. Kaelig's grep-for-forbidden-prefix
  and `tsc` are real gates because Quality Gate's own budget logic halts on failure; anydesign's
  `lint_design_md.py` is the same kind of script but is not wired to anything that blocks delivery.
  The inner loop should make validators fire somewhere the agent cannot skip, not merely tell the
  agent to run them. (Kaelig, anydesign)
- A guard should check presence and cross-entity readiness, not truth, and should say so. Katagami's
  `design_language.ioa.toml` guard is a clean worked example of the pattern this repo's own
  CLAUDE.md already commits to, worth reading as a template for the `Component.Publish` guard's
  shape. (Katagami)
- Separate the role that produces from the role that reviews, and let the guard read only the
  reviewer's verdict. Katagami's review-agent BEHAVIOR.md ("Never rule on your own work") plus the
  `Publish` guard's `quality_review_passed` check prevents an unreviewed publish structurally,
  while leaving review quality itself to the separate role's judgment. (Katagami)
- Route agent-to-agent knowledge through named required files, not chat. Kaelig's fix for
  cross-agent drift, `component-rules.md` as a mandatory pre-flight load, generalizes past this one
  pipeline: downstream agents in fresh contexts need an explicit artifact contract. (Kaelig)
- A halt should state what degraded and what it costs, with named options, not just fail, the way
  Kaelig's quality-score stop message names the score, the missing data, and three options. (Kaelig)
- Confidence-marking and an explicit "don't invent" instruction are cheap and worth copying even as
  prose; they lower the rate of the failure a downstream mechanism would otherwise have to catch.
  (anydesign)

## What I did not check

- I did not open Temper's own source (linked from Katagami's README as `nerdsane/temper`) to confirm
  how a guard's `cross_entity_state` or `is_true` check is actually evaluated at runtime, or what
  happens on a rejected transition beyond what the spec file and README state. The guard was read
  directly from `.ioa.toml`; its execution semantics were not verified against Temper's engine code.
  Similarly, Kaelig's post never names what software performs the quality-score hard stop; only the
  scoring arithmetic itself is confirmed deterministic, with no LLM judgment involved.
- I did not read Katagami's `katagami-curation` agent skills (`research-direction`,
  `synthesize-language`, `organize-taxonomy`) beyond the README's one-line descriptions, and read
  only part of the review-agent `BEHAVIOR.md` (identity, evidence, ruling, non-self-review). I did
  not read the curator-agent or human-curator behavior files, did not check how
  `quality_review_passed` is written back to the entity or which action call performs that write,
  and did not check `design_element.ioa.toml` or `element_manifest`, so the per-element guard
  structure is unverified.
- I did not open anydesign's `scripts/lint_design_md.py` or `scripts/verify_design.py` source; what
  each checks is taken from the SKILL.md's prose description, not from reading the script logic.
- I did not read any of the other 66 `awesome-design-skills` SKILL.md files beyond `minimal`, and
  did not check the `typeui.sh` CLI's own source (a separate repository, only linked from the
  README); the claim that all 67 skills share one template rests on the managed-block marker in the
  one file read and the repeated file-tree structure, not on reading each file or the CLI's code.
- I did not check the shadcn registry-item JSON schema or the MCP server's own source; the
  no-constraint finding rests on the docs page only, as the task allowed for this source. I also did
  not check Anthropic's other named skills (`brand-guidelines`, `theme-factory`, `canvas-design`,
  `web-artifacts-builder`) from the starting map; only `frontend-design` and `webapp-testing` were
  in scope here.
- I did not verify the starting map's claim about social engagement on Kaelig's post, and omit it as
  not load-bearing here. Star counts and adoption figures from the starting map are likewise omitted,
  per this repo's format rule. I did not check `google-labs-code/design.md`, the format Katagami's
  `DESIGN.md` export targets compatibility with; that claim rests on Katagami's own
  `has_valid_design_md` guard flag and README description, not on reading Google's spec directly.

## Sources

| source | pinned URL | read |
|---|---|---|
| Kaelig Deloumeau-Prigent blog post | https://www.kaelig.fr/design-system-components-with-ai-agent-teams/ (fetched 2026-09-03) | full |
| anydesign `SKILL.md` | https://raw.githubusercontent.com/uxKero/anydesign/1b478809c3ccfebae558d93e4cc719e27f18bc0a/SKILL.md | full |
| anydesign `README.md` | https://raw.githubusercontent.com/uxKero/anydesign/1b478809c3ccfebae558d93e4cc719e27f18bc0a/README.md | full |
| awesome-design-skills `README.md` | https://raw.githubusercontent.com/bergside/awesome-design-skills/f631a09b4fcc0166f2e2c1a8c81906ef680c57e8/README.md | full |
| awesome-design-skills `skills/minimal/SKILL.md` | https://raw.githubusercontent.com/bergside/awesome-design-skills/f631a09b4fcc0166f2e2c1a8c81906ef680c57e8/skills/minimal/SKILL.md | full |
| Katagami `README.md` | https://raw.githubusercontent.com/arni-labs/katagami/5366ab8bbe7012a9768a37a12f869d7c9a0a3681/README.md | full |
| Katagami `katagami-commons/specs/design_language.ioa.toml` | https://raw.githubusercontent.com/arni-labs/katagami/5366ab8bbe7012a9768a37a12f869d7c9a0a3681/katagami-commons/specs/design_language.ioa.toml | part (state variables and the `SubmitForReview`/`Publish`/`Revise`/`Archive` actions) |
| Katagami `.agents/behaviors/review-agent/BEHAVIOR.md` | https://raw.githubusercontent.com/arni-labs/katagami/5366ab8bbe7012a9768a37a12f869d7c9a0a3681/.agents/behaviors/review-agent/BEHAVIOR.md | part (first ~100 lines) |
| Anthropic skills `skills/frontend-design/SKILL.md` | https://raw.githubusercontent.com/anthropics/skills/41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f/skills/frontend-design/SKILL.md | full |
| Anthropic skills `skills/webapp-testing/SKILL.md` | https://raw.githubusercontent.com/anthropics/skills/41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f/skills/webapp-testing/SKILL.md | full |
| shadcn MCP docs | https://ui.shadcn.com/docs/mcp (fetched 2026-09-03) | part (page content only) |
