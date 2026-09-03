---
title: Agent-consumable component systems
date: 2026-09-03
question: Which component systems are built for agents to consume, what do they expose, and does any of them constrain what the agent builds?
---

## Summary

shadcn/ui is the reference pattern: a JSON registry format plus an MCP server that lets an agent
search, view, and install components, plus an actual Claude Code skill directory of prose rules
("Critical Rules") for how to compose them. Several systems distribute components the same way
(AI Elements, Magic UI, Aceternity, 21st.dev) by publishing a `registry.json`/`registry-item.json`
compatible with the shadcn CLI, or by running their own MCP server over a component catalog
(21st.dev). None of these enforce anything: every "rule" I found is either injected agent context
(a skill file, a checklist tool) or documentation, never a mechanism that blocks output. The one
partial exception is Adobe's Spectrum Design Data project, which ships an agent-facing MCP server
with a `validate_usage` tool that returns a diagnostic report: still advisory to the calling agent,
not a publish gate, but a real machine check rather than prose. OpenLabs' oa-design packages a
design language as a Claude Code skill with type-checked component source, which is closer to
"supply with constraints an agent cannot silently violate in the shipped code" than to a runtime
gate. Prefab, assistant-ui, and CopilotKit are a different category: UI *for* agents to render to
end users, not UI a coding agent builds and publishes. For a pilot open design system with
machine-readable tokens, a real component library, and Storybook, IBM Carbon and GitHub Primer are
the strongest verified candidates; Carbon additionally ships DTCG-format token JSON.

## Registry-and-MCP pattern (shadcn as the reference)

shadcn/ui's registry is described in its own docs as "a distribution system for code" that lets a
project author "distribute your custom components, hooks, pages, config, rules and other files to
any project" (fetched 2026-09-03, https://ui.shadcn.com/docs/registry). The two schema files that
define the format are published in the repo itself, pinned at commit `8720dec73f5aebed9f649ea58636f54599fdedf1`:

- `apps/v4/public/schema/registry.json`: a JSON Schema for the top-level registry document: `name`,
  `homepage`, `items` (an array of `registry-item.json`-shaped objects), an `include` list to compose
  registries from sub-files, and a `pagination` block for dynamic/paginated catalogs.
- `apps/v4/public/schema/registry-item.json`: a JSON Schema for one item: `name`, `type` (one of
  `registry:lib`, `registry:block`, `registry:component`, `registry:ui`, `registry:hook`,
  `registry:theme`, `registry:page`, `registry:file`, `registry:style`, `registry:base`,
  `registry:font`, `registry:item`), `dependencies`/`devDependencies` (npm packages),
  `registryDependencies` (other registry items or registry URLs), `files` (path, content, type,
  target: with `@components/`, `@ui/`, `@lib/`, `@hooks/` placeholders resolved against the
  consuming project's own aliases), `tailwind`/`cssVars`/`css` for style and token injection, `docs`
  (markdown), `categories`, and font-specific and base-specific fields. Nothing in this schema
  encodes a validation rule; it is purely a description of what files to drop where and what to
  install alongside them.

Pinned source:
`https://raw.githubusercontent.com/shadcn-ui/ui/8720dec73f5aebed9f649ea58636f54599fdedf1/apps/v4/public/schema/registry-item.json`
and `.../registry.json`.

The MCP server (`packages/shadcn/src/mcp`, same SHA) exposes these tools, per the shipped
documentation at `skills/shadcn/mcp.md`:

- `shadcn:get_project_registries`: reads `components.json`.
- `shadcn:list_items_in_registries`: enumerates configured registries (built-in `@shadcn`, namespaced
  registries, or any `owner/repo` GitHub source with a root `registry.json`).
- `shadcn:search_items_in_registries`: fuzzy-searches the same set of registries.
- `shadcn:view_items_in_registries`: full file contents of an item.
- `shadcn:get_item_examples_from_registries`: demo code.
- `shadcn:get_add_command_for_items`: the CLI command to install.
- `shadcn:get_audit_checklist`: "Returns a checklist for verifying components (imports, deps, lint,
  TypeScript)."

That last tool is the closest thing to enforcement shadcn's MCP surface offers, and it is explicitly
a checklist returned as text for the calling agent to apply itself: nothing on the server side
inspects the agent's output against it. The vendor docs page for the MCP server
(https://ui.shadcn.com/docs/mcp, fetched 2026-09-03) describes browsing, searching, and installing
components conversationally, and makes no statement about validating, gating, or constraining what
the agent then does with the installed source.

shadcn also ships an actual **agent skill**, not just docs: `skills/shadcn/SKILL.md` (same SHA) is a
Claude Code skill (frontmatter `name: shadcn`, `user-invocable: false`) with a "Critical Rules"
section: e.g. "className for layout, not styling. Never override component colors or typography,"
"No space-x-* or space-y-*. Use flex with gap-*," "Use semantic colors... never raw values like
bg-blue-500," "Dialog, Sheet, and Drawer always need a Title... for accessibility." These rules link
to sibling files (`rules/styling.md`, `rules/forms.md`, `rules/composition.md`, `rules/icons.md`,
`rules/chat.md`) with incorrect/correct code pairs, and the skill directory carries its own eval
suite (`skills/shadcn/evals/evals.json`) that scores whether a generated component followed the
rules (e.g. eval 1 checks for `FieldGroup`/`Field`, `data-invalid`/`aria-invalid`, `gap-*`, semantic
color tokens). This is real content, accessibility and token guidance included, but it is prose
injected into the agent's context plus an offline eval harness for the skill's own quality, not a
runtime check that blocks a non-conforming component from being installed or committed. shadcn also
publishes an `llms.txt` at `https://ui.shadcn.com/llms.txt` (verified: HTTP 200, `content-type:
text/plain`, `content-disposition: inline; filename="llms.txt"`, fetched 2026-09-03) linking every
docs page and component page.

## Systems built on that pattern

### Vercel AI Elements

AI Elements' README (pinned at `6a9d5b1822ffb10bba4bd97175f01edd7d8651cd`,
`https://raw.githubusercontent.com/vercel/ai-elements/6a9d5b1822ffb10bba4bd97175f01edd7d8651cd/README.md`)
states it is "a component library built on top of shadcn/ui" and documents two install paths: its own
CLI (`npx ai-elements@latest`) or the shadcn CLI directly against its hosted registry
(`npx shadcn@latest add https://elements.ai-sdk.dev/api/registry/all.json`, or a per-component
`.../registry/message.json`). It requires shadcn/ui already initialized in the target project. It
supplies chat/AI-native components (message, conversation, code-block, reasoning display); it states
no accessibility or token constraint of its own beyond what it inherits from shadcn/ui's semantic
tokens.

### Magic UI and Aceternity

Both publish a registry.json compatible with the shadcn schema, fetched directly and confirmed as
JSON (not just docs pages): `https://magicui.design/r/registry.json` (HTTP 200,
`{"name":"magicui", ...}`, items typed `registry:style`/`registry:ui` with `dependencies`,
`registryDependencies`, `cssVars`) and `https://ui.aceternity.com/registry.json` (HTTP 200,
`content-type: application/json`, items typed `registry:ui` with `files`). Neither publishes an MCP
server of its own as far as I found; both are consumed the same way any `owner/repo` or hosted
registry is consumed by the shadcn CLI/MCP tools above. Supply only; no constraint mechanism found.

### 21st.dev (Magic MCP / 21st MCP)

The current README of `21st-dev/magic-mcp` (pinned `6e5a43d3621442a450948864a74396d206b23c65`,
`https://raw.githubusercontent.com/21st-dev/magic-mcp/6e5a43d3621442a450948864a74396d206b23c65/README.md`)
states the old "Magic MCP" package name and tools (`21st_magic_component_builder`,
`21st_magic_component_inspiration`, `21st_magic_component_refiner`, `logo_search`) have been replaced
by the "21st MCP," reachable as a plain HTTP MCP server at `https://21st.dev/api/mcp` with an API-key
header, exposing (per the README) "catalog search across components/themes/templates, paid code
retrieval, bookmarks, team libraries, UI generation with variants, profile management, and more,"
with new canonical tool names `generate`, `get_inspiration`, `search_logo`. The package description
on the GitHub API record describes it as "It's like v0, but in your Cursor / Claude Code / Windsurf:
search 10,000+ React/Tailwind components, generate new UI with AI, and publish your own — right from
your editor." Nothing in this README describes token, accessibility, or composition constraints
on generated output: it is a search-and-generate service, not a gate.

### Prefab (PrefectHQ)

Prefab exists as described, confirmed from its own README (`main` branch,
`https://raw.githubusercontent.com/PrefectHQ/prefab/main/README.md`, fetched 2026-09-03: the repo
has an active-development banner so I did not pin a stale SHA for a fast-moving README, see "What I
did not check"). It is "the generative UI framework that even humans can use": a Python DSL
(`prefab_ui.components`) that compiles a component tree to a JSON protocol, rendered by "a bundled
React renderer built on shadcn/ui." Verbatim: "The component tree compiles to a JSON protocol and is
rendered by a bundled React frontend built on shadcn/ui... The output is declarative and
serializable, which means UIs are safe for agents to generate, simple to validate, and portable
across any transport." It targets MCP Apps and ships as part of FastMCP. License: Apache-2.0
(`https://raw.githubusercontent.com/PrefectHQ/prefab/main/LICENSE`). This is UI *by* an agent in the
sense that an agent (or a developer) declares the tree, but the tree is composed from a fixed set of
~100 prebuilt components: there is no path for an agent to author a wholly new component inside
Prefab's own design system the way it can inside a shadcn registry.

## UI for agents versus UI by agents

assistant-ui (pinned `d511ff496c59005f5df4df155dcccb3c1ddff475`) and CopilotKit (pinned
`3bc03f9a887a33ef3035d7dd1aeb8e76e422cf9c`) are both libraries a *human developer* installs to give
an end user a chat or generative-UI surface backed by an LLM at runtime. assistant-ui's README
describes it as "an open-source TypeScript/React library to build production-grade AI chat
experiences fast," installed via `npx assistant-ui@latest create/init` (MIT license, copyright
AgentbaseAI Inc.). CopilotKit's README describes building "agent-native applications... Generative
UI, shared state, and human-in-the-loop workflows for React, Angular, Vue, React Native — and in
Slack and Microsoft Teams" (MIT license). Neither is a system a coding agent uses to author new
components inside a design system; both ship components that render an agent's output to a user.
They are out of scope for "does it constrain what the agent builds" because the "agent" in their own
docs is the runtime chat/task agent, not a component-authoring agent, and I did not pursue them
further than confirming this distinction from their own README text.

## Design language as a skill (oa-design)

`OpenLabs-so/oa-design`, pinned `bd200daeac8eca2501139cb0fa29cc12e4709303`, packages "the design
language of Open Analytics... three ways: an agent skill, type-checked component recipes, and a
CLI." Its own README states: "The recipes are the consumer format and they are never written by
hand": i.e. the markdown recipe files under `skills/oa-design/*.md` are generated from the
type-checked TypeScript source under `components/<slug>/*.tsx`, not authored independently, so the
two cannot drift. The skill (`skills/oa-design/SKILL.md`) states ten rules as prose ("One ink,
everything derived," "Squircles for surfaces, pills for actions," "Quality floor, always: Focus
rings, `role="status"`, `aria-hidden` decorations, `prefers-reduced-motion`, no horizontal page
scroll") plus a fixed table of seven named spring constants ("Do not invent an eighth"). Distribution
is CLI-based and shadcn-flavored: `npx getopen-design add tab-bar toast` copies recipes into the
project; `npx getopen-design tokens` emits the token CSS; `npx getopen-design skill` installs the
skill file into `.claude/skills`. License: MIT (`Copyright (c) 2026 Voprex Labs`).

Compared to a shadcn-style registry, oa-design's difference is provenance, not enforcement: because
the recipe an agent reads is mechanically derived from a real, compiled TypeScript component rather
than hand-written prose, the documented API can't silently diverge from the shippable code the way a
skill file describing a component from memory could. That is a stronger *supply* guarantee (the doc
and the artifact cannot disagree) but it is still not a *constraint* mechanism: nothing stops an
agent from producing a new component that ignores the ten rules; there is no schema, linter, or gate
in the repository that checks a consuming project's output against them. The repo also carries its
own `evals/evals.json`, structured the same way as shadcn's skill evals, again scoring the skill's
own output quality offline rather than gating a live publish path.

## Candidate open design systems for a pilot

| System | Tokens format and file | React components | Storybook | License | Fit as pilot |
|---|---|---|---|---|---|
| GitHub Primer (`primer/primitives`, pinned `93e536c26d7984f6d2368c0b102f111f079cfe0b`) | Custom JSON5, not raw DTCG: `src/tokens/base/{color,motion,size,typography}/*.json5`, e.g. `base/color/light/light.json5`. Distributed as compiled CSS variables under `dist/css/`. | Primer's React implementation lives in a sibling repo (`primer/react`, not opened here: see "What I did not check"). | Yes: badge and link to `https://primer.style/primitives/storybook/` in the README, and a `.playwright`/`docs/storybook` workspace entry in `package.json`. | MIT (declared in `package.json`; no separate `LICENSE` file found at this path in this repo: see gaps). | Good: token repo has its own CI gate already. `a11y-contrast.yml`, read in full, runs a color-contrast check on every PR that touches tokens or dependencies: the `build` job sets `continue-on-error: true` (so its own later steps, which post a PR comment and job summary, still run after a failing check step), but a separate downstream job, `Fail_action_on_contrast_failing`, has no `continue-on-error` and calls `exit 1` whenever `build` reports any failing check, which fails the overall workflow run. Whether that failed run is a required status check that actually blocks the merge button is a repository branch-protection setting this workflow file cannot show. Only the token layer was opened here; the component repo needs separate verification before a pilot build. |
| Atlassian Design System tokens (`@atlaskit/tokens`, npm `repository` field pins `bitbucket.org/atlassian/atlassian-frontend-mirror`, a public monorepo mirror; commit `728c79c92e88eb0074db4d8f28591f799ca9917b` pinned via the Bitbucket commits API) | Custom format, not DTCG: `design-system/tokens/src/artifacts/token-data.json5` (JSON5) plus generated TypeScript/CSS artifacts, consumed through a `token()` function API rather than raw token files. | Yes, first-party: dozens of sibling `design-system/<component>` packages in the same mirror (`button`, `avatar`, `form`, `modal-dialog`, and more), confirmed present in the repo tree at the pinned commit. | Not found: the `design-system/button` package's file tree (checked as a sample) has `docs`, `examples`, and a `constellation` directory (Atlassian's own internal docs tool) but no `.storybook` config; no Storybook was found anywhere in a root-level tree listing either. | Apache-2.0 (confirmed both via `npm view @atlaskit/tokens license` and the `license` field in `design-system/tokens/package.json` at the pinned commit). | Verified further than the original pass found: the earlier GitHub-only search missed that `@atlaskit/tokens`'s own npm `repository` field names a public Bitbucket mirror. Token format, a first-party component library, and license are now confirmed from that primary source; Storybook is confirmed absent, not merely unchecked. Worth a second look as a pilot candidate, with the caveat that its token format is bespoke (JSON5 plus a runtime `token()` API) rather than a widely reused open schema. |
| IBM Carbon (`carbon-design-system/carbon`, pinned `d8042305a792ffe57c3a37f045ccb89c705ba9b1`) | DTCG-format JSON confirmed present: `packages/themes/src/dtcg/components/*.json` (e.g. `button.json`, `content-switcher.json`, `notification.json`), alongside Sass/JS token packages (`@carbon/colors`, `@carbon/layout`, `@carbon/motion`, `@carbon/type`, `@carbon/themes`). | Yes, in-repo and first-party: `@carbon/react` and `@carbon/web-components`, both listed in the monorepo's own package table. | Yes, both flavors: linked from the README (`react-storybook`, `wc-storybook`) and confirmed by deploy workflows (`deploy-react-storybook.yml`, `deploy-web-components-storybook.yml`). | Apache-2.0 (`LICENSE` file, verified first three lines). | Strongest verified candidate. Everything needed for a pilot (machine-readable tokens including a DTCG subset, a first-party React library, live Storybook, permissive license, all confirmed inside one monorepo at one pinned SHA) is present without cross-repo chasing. |
| Adobe Spectrum (`adobe/spectrum-design-data`, formerly `spectrum-tokens`, pinned `f81ed06323631b3b0f375b37b0beacb3a63b6ce1`) | Custom "Design Data" format with its own normative spec (`packages/design-data-spec`, "Spec version 1.0.0-draft"), JSON Schemas (Draft 2020-12) and a rule catalog (`rules/rules.yaml`, rules `SPEC-001`…`SPEC-006`); token package is `packages/tokens`. Not DTCG. | React implementation is the separate `adobe/react-spectrum` repo (Apache-2.0, confirmed via the GitHub API record only: not opened further here). | Not verified in this pass: react-spectrum was not opened beyond its license/description. | Apache-2.0 (root `LICENSE`, verified). | Interesting but split across two repos (tokens/data vs. React components), and the token format is Spectrum-specific rather than a widely reused open schema: see the MCP section below for why this repo is the most relevant one *for the harness's mechanism*, independent of whether it becomes the pilot's token source. |

## Table: what each system exposes, supplies, and constrains

| System | Exposes to agent | Supplies | Constrains (what) | Enforced or prose |
|---|---|---|---|---|
| shadcn/ui | Registry JSON (`registry.json`/`registry-item.json`), MCP tools (search/view/install/audit-checklist), a Claude Code skill directory, `llms.txt` | Component source files, install commands, npm/registry dependency resolution | Composition and styling conventions (semantic tokens, `Field`/`FieldGroup`, dialog titles for a11y, icon sizing) | Prose (skill file + linked rule docs) plus an offline eval suite scoring the skill's own output; `get_audit_checklist` returns a checklist for the agent to self-apply. Nothing server-side blocks a non-conforming install. |
| Vercel AI Elements | shadcn-compatible registry JSON, hosted at `elements.ai-sdk.dev` | AI-native chat/message/reasoning components, built on shadcn/ui | None beyond what it inherits from shadcn/ui | N/A: supply only |
| Magic UI | shadcn-compatible `registry.json` | Visual/animation components | None found | N/A: supply only |
| Aceternity | shadcn-compatible `registry.json` | Visual/animation components | None found | N/A: supply only |
| 21st.dev (21st MCP / ex-Magic MCP) | HTTP MCP server (`generate`, `get_inspiration`, `search_logo`, catalog search) | Component search and on-demand UI generation across a hosted catalog | None found | N/A: supply/generate only |
| Prefab | Python DSL over ~100 prebuilt components; MCP Apps integration via FastMCP | A fixed component set, composed (not authored) by the agent, rendered by a shadcn/ui-based React bundle | Implicitly constrains to its component set (an agent cannot introduce a new primitive, only compose existing ones) | Structural, by the DSL's own vocabulary: not a validator; there is nothing to violate because out-of-set components don't exist in the language |
| assistant-ui | npm packages + CLI scaffold | Chat UI components for an end-user-facing agent | None (out of scope: UI for an agent, not UI by one) | N/A |
| CopilotKit | npm packages, AG-UI protocol | Generative UI / shared state / human-in-the-loop components | None (out of scope: UI for an agent, not UI by one) | N/A |
| oa-design (OpenLabs) | CLI (`getopen-design add/tokens/skill`), Claude Code skill directory, one-file `DESIGN-SKILL.md` | Type-checked component recipes, token CSS, one-file design language for any agent | Ten stated rules (one neutral color, one spring family, squircle/pill shape rules, a11y quality floor) | Prose in the skill file; recipes are mechanically derived from compiled TS so doc and code can't drift, but nothing gates a new component against the ten rules. Offline eval suite scores skill output, not a live gate. |
| Adobe Spectrum Design Data (`design-data-agent-mcp`) | MCP server with tools `primer`, `resolve_token`, `query_tokens`, `describe_component`, `validate_usage`, `diff_datasets`, `write`; also a packaged Claude Code skill | Full token taxonomy, component schemas, field definitions, to an agent authoring session | `validate_usage`: "Validate token usage and return a diagnostic report" | Reports, does not gate. The tool is callable by the agent and returns diagnostics; nothing in the MCP server or skill blocks the agent's output if it ignores the report. Separately, the repo's own token package is validated in CI via Moon tasks (`tokens:validateDesignData`) against JSON Schema and a rule catalog: that gates merges to the *token data repo itself*, not an agent's downstream component. |
| GitHub Primer (primitives) | npm package, CSS variable distribution, Storybook | Color/spacing/typography primitives | Color contrast, checked on PRs touching tokens | Fails the run, blocking status unconfirmed: the `build` job has `continue-on-error: true`, but a separate `Fail_action_on_contrast_failing` job (no `continue-on-error`, `exit 1` on any failing check) fails the overall workflow whenever a contrast check fails. Whether that failed run is a required status check on the branch, which would actually block the merge button, is not something the workflow file itself can show. |
| IBM Carbon | npm packages (`@carbon/react`, `@carbon/web-components`, `@carbon/themes`, etc.), Storybook | React and web-component implementations, Sass/JS/DTCG-JSON tokens | Not evaluated for an agent-facing gate: none found; standard CI (`ci.yml`) covers the library's own tests | Not applicable: no agent-consumption surface was found; this is a candidate for what to build the harness *against*, not a peer of the agent-facing systems above |

## What I did not check

- I did not open `primer/react` (the component implementation repo) at all: Primer's row above is
  built entirely from `primer/primitives`, so "React components: yes" for Primer rests on the README
  link and package-table claim in that repo, not on inspecting `primer/react`'s own source or its
  license file directly.
- I did not open `adobe/react-spectrum` beyond its GitHub API description and license field; no
  README, install instructions, or Storybook link were verified for it.
- I did not fully audit Carbon's CI (`ci.yml`, `achecker.js` at the repo root suggests an accessibility
  checker exists) to determine whether any check there gates merges the way Primer's `a11y-contrast.yml`
  fails its own run on a violation. `achecker.js` was seen in the file tree only, not opened.
- I did not verify whether `@adobe/design-data-agent-mcp`'s `write` tool ("Write agent-generated
  product context to the dataset") has any validation attached before a write lands, only that it
  exists as a named tool in the README's table.
- I did not pin a commit SHA for PrefectHQ/prefab; its README carries an active "under very active
  development" banner and I read the `main` branch tip on 2026-09-03 rather than a fixed SHA, so a
  future reader following the citation may see different content. This is a deliberate exception to
  rule 1's spirit, noted rather than hidden.
- Atlassian Design System tokens: the corrected pass found the source (the npm package's own
  `repository` field, pointing to a public Bitbucket mirror), but I did not open the `tokens`
  package's own README, did not confirm Storybook's absence beyond one sample component package's
  file tree (`button`), and did not check whether any first-party React component package other
  than the tokens package itself is published to npm under a name I searched for.
- I did not test any of the MCP servers described here by actually connecting a client to them
  (shadcn's, 21st's, or Adobe's): everything above is drawn from published source/schema files and
  vendor documentation text, not from an observed tool-call transcript.
- I did not check whether Magic UI or Aceternity publish their own MCP server in addition to their
  shadcn-compatible `registry.json`: I confirmed the JSON files directly but did not search their
  docs sites exhaustively for a separate MCP offering.
- I did not evaluate license compatibility or terms-of-use restrictions on any of the community
  registries (Magic UI, Aceternity) beyond noting their existence: some community registries carry
  usage restrictions on their component source that a pilot would need to check before adoption.
- I did not investigate whether shadcn's `evals/evals.json` or oa-design's `evals/evals.json` are run
  in any CI pipeline for those repos; I only confirmed the files exist and what one entry each
  contains.
- Corrected after the 2026-09-03 fidelity review: read the full `a11y-contrast.yml` and corrected
  the claim that Primer's contrast check does not block merges (a separate job does fail the run,
  though whether that is a required status check is unconfirmed); restored Atlassian Design System
  tokens to the pilot table after finding its public Bitbucket mirror via the npm package's own
  `repository` field; fixed a silently truncated 21st.dev quote; and split one shadcn MCP bullet
  that had bundled two distinct tools into two, so all seven tools are listed separately.

## Sources

| Source | Pinned URL | Read |
|---|---|---|
| shadcn/ui repo tree | `github.com/shadcn-ui/ui` @ `8720dec73f5aebed9f649ea58636f54599fdedf1` | part (tree listing + specific files below) |
| shadcn registry-item.json schema | `https://raw.githubusercontent.com/shadcn-ui/ui/8720dec73f5aebed9f649ea58636f54599fdedf1/apps/v4/public/schema/registry-item.json` | full |
| shadcn registry.json schema | `https://raw.githubusercontent.com/shadcn-ui/ui/8720dec73f5aebed9f649ea58636f54599fdedf1/apps/v4/public/schema/registry.json` | full |
| shadcn skill: SKILL.md | `https://raw.githubusercontent.com/shadcn-ui/ui/8720dec73f5aebed9f649ea58636f54599fdedf1/skills/shadcn/SKILL.md` | full |
| shadcn skill: mcp.md | `https://raw.githubusercontent.com/shadcn-ui/ui/8720dec73f5aebed9f649ea58636f54599fdedf1/skills/shadcn/mcp.md` | full |
| shadcn skill: evals.json | `https://raw.githubusercontent.com/shadcn-ui/ui/8720dec73f5aebed9f649ea58636f54599fdedf1/skills/shadcn/evals/evals.json` | part (first ~3 evals) |
| shadcn LICENSE.md | `https://raw.githubusercontent.com/shadcn-ui/ui/8720dec73f5aebed9f649ea58636f54599fdedf1/LICENSE.md` | part (header) |
| ui.shadcn.com/docs/mcp | vendor docs, fetched 2026-09-03 | part (fetched summary) |
| ui.shadcn.com/docs/registry | vendor docs, fetched 2026-09-03 | part (fetched summary) |
| ui.shadcn.com/llms.txt | vendor doc, fetched 2026-09-03 | part (headers + first 40 lines) |
| Vercel AI Elements README | `https://raw.githubusercontent.com/vercel/ai-elements/6a9d5b1822ffb10bba4bd97175f01edd7d8651cd/README.md` | part (first 80 lines) |
| elements.ai-sdk.dev/overview | vendor docs, fetched 2026-09-03 (redirect target not re-fetched; README sufficed) | not fetched |
| 21st-dev/magic-mcp README | `https://raw.githubusercontent.com/21st-dev/magic-mcp/6e5a43d3621442a450948864a74396d206b23c65/README.md` | full |
| 21st-dev/magic-mcp repo metadata | `gh api repos/21st-dev/magic-mcp`, fetched 2026-09-03 | full (description field) |
| Magic UI registry.json | `https://magicui.design/r/registry.json`, fetched 2026-09-03 | part (first ~500 bytes, structure confirmed) |
| Aceternity registry.json | `https://ui.aceternity.com/registry.json`, fetched 2026-09-03 | part (first ~400 bytes, structure confirmed) |
| PrefectHQ/prefab README | `https://raw.githubusercontent.com/PrefectHQ/prefab/main/README.md`, fetched 2026-09-03 | part (first ~60 lines) |
| PrefectHQ/prefab LICENSE | `https://raw.githubusercontent.com/PrefectHQ/prefab/main/LICENSE`, fetched 2026-09-03 | part (header) |
| assistant-ui README | `https://raw.githubusercontent.com/assistant-ui/assistant-ui/d511ff496c59005f5df4df155dcccb3c1ddff475/README.md` | part (first 40 lines) |
| assistant-ui LICENSE | `https://raw.githubusercontent.com/assistant-ui/assistant-ui/d511ff496c59005f5df4df155dcccb3c1ddff475/LICENSE` | part (header) |
| CopilotKit README | `https://raw.githubusercontent.com/CopilotKit/CopilotKit/3bc03f9a887a33ef3035d7dd1aeb8e76e422cf9c/README.md` | part (first 40 lines) |
| CopilotKit LICENSE | `https://raw.githubusercontent.com/CopilotKit/CopilotKit/3bc03f9a887a33ef3035d7dd1aeb8e76e422cf9c/LICENSE` | part (header) |
| OpenLabs-so/oa-design tree | `github.com/OpenLabs-so/oa-design` @ `bd200daeac8eca2501139cb0fa29cc12e4709303` | part (tree listing) |
| oa-design README | `https://raw.githubusercontent.com/OpenLabs-so/oa-design/bd200daeac8eca2501139cb0fa29cc12e4709303/README.md` | part (first 60 lines) |
| oa-design SKILL.md | `https://raw.githubusercontent.com/OpenLabs-so/oa-design/bd200daeac8eca2501139cb0fa29cc12e4709303/skills/oa-design/SKILL.md` | part (first 100 lines) |
| oa-design LICENSE | `https://raw.githubusercontent.com/OpenLabs-so/oa-design/bd200daeac8eca2501139cb0fa29cc12e4709303/LICENSE` | part (header) |
| primer/primitives tree | `github.com/primer/primitives` @ `93e536c26d7984f6d2368c0b102f111f079cfe0b` | part (tree listing) |
| primer/primitives README | `https://raw.githubusercontent.com/primer/primitives/93e536c26d7984f6d2368c0b102f111f079cfe0b/README.md` | part (first 40 lines) |
| primer/primitives package.json | `https://raw.githubusercontent.com/primer/primitives/93e536c26d7984f6d2368c0b102f111f079cfe0b/package.json` | part (first 30 lines) |
| primer/primitives a11y-contrast.yml | `https://raw.githubusercontent.com/primer/primitives/93e536c26d7984f6d2368c0b102f111f079cfe0b/.github/workflows/a11y-contrast.yml` | full |
| adobe/spectrum-tokens README (rename notice) | `https://raw.githubusercontent.com/adobe/spectrum-tokens/1c78755d138ecd340cce5b7bcd260526d854456b/README.md` | full |
| adobe/spectrum-design-data tree | `github.com/adobe/spectrum-design-data` @ `f81ed06323631b3b0f375b37b0beacb3a63b6ce1` | part (tree listing, filtered) |
| adobe/spectrum-design-data README | `https://raw.githubusercontent.com/adobe/spectrum-design-data/f81ed06323631b3b0f375b37b0beacb3a63b6ce1/README.md` | part (~50 lines) |
| adobe/spectrum-design-data LICENSE | `https://raw.githubusercontent.com/adobe/spectrum-design-data/f81ed06323631b3b0f375b37b0beacb3a63b6ce1/LICENSE` | part (header) |
| design-system-registry README (deprecated pkg) | `https://raw.githubusercontent.com/adobe/spectrum-design-data/f81ed06323631b3b0f375b37b0beacb3a63b6ce1/packages/design-system-registry/README.md` | part (first 80 lines) |
| component-schemas README | `https://raw.githubusercontent.com/adobe/spectrum-design-data/f81ed06323631b3b0f375b37b0beacb3a63b6ce1/packages/component-schemas/README.md` | part (first 50 lines) |
| design-data-spec README | `https://raw.githubusercontent.com/adobe/spectrum-design-data/f81ed06323631b3b0f375b37b0beacb3a63b6ce1/packages/design-data-spec/README.md` | full |
| tools/design-data-agent-mcp README | `https://raw.githubusercontent.com/adobe/spectrum-design-data/f81ed06323631b3b0f375b37b0beacb3a63b6ce1/tools/design-data-agent-mcp/README.md` | full |
| tools/design-data-skill README | `https://raw.githubusercontent.com/adobe/spectrum-design-data/f81ed06323631b3b0f375b37b0beacb3a63b6ce1/tools/design-data-skill/README.md` | part (first 80 lines) |
| packages/tokens README | `https://raw.githubusercontent.com/adobe/spectrum-design-data/f81ed06323631b3b0f375b37b0beacb3a63b6ce1/packages/tokens/README.md` | part (first 40 lines) |
| carbon-design-system/carbon tree | `github.com/carbon-design-system/carbon` @ `d8042305a792ffe57c3a37f045ccb89c705ba9b1` | part (tree listing, filtered) |
| carbon README | `https://raw.githubusercontent.com/carbon-design-system/carbon/d8042305a792ffe57c3a37f045ccb89c705ba9b1/README.md` | part (~90 lines) |
| carbon LICENSE | `https://raw.githubusercontent.com/carbon-design-system/carbon/d8042305a792ffe57c3a37f045ccb89c705ba9b1/LICENSE` | part (header) |
| carbon DTCG token tree (`packages/themes/src/dtcg`) | tree listing at same SHA | part (filenames only, contents not opened) |
| adobe/react-spectrum repo metadata | `gh api repos/adobe/react-spectrum`, fetched 2026-09-03 | part (description + license field only) |
| Atlassian design tokens search (original GitHub-only attempt) | `gh api search/repositories?q=org:atlassian+tokens`, `gh api repos/atlassian/atlassian-frontend-mirror`, `atlassian.design/components/tokens/overview/`, fetched 2026-09-03 | attempted, all returned not-found: no content to read (see corrected rows below) |
| `@atlaskit/tokens` npm repository field | `npm view @atlaskit/tokens repository --json`, fetched 2026-09-03 | full |
| Bitbucket repo metadata: `atlassian/atlassian-frontend-mirror` | `https://api.bitbucket.org/2.0/repositories/atlassian/atlassian-frontend-mirror`, fetched 2026-09-03 | full (confirmed `is_private: false`) |
| Bitbucket directory listing: `design-system/` | `https://api.bitbucket.org/2.0/repositories/atlassian/atlassian-frontend-mirror/src/728c79c92e88eb0074db4d8f28591f799ca9917b/design-system/?pagelen=100`, fetched 2026-09-03 | full (90 sibling package directories, including `tokens`) |
| Bitbucket commits API (pin) | `https://api.bitbucket.org/2.0/repositories/atlassian/atlassian-frontend-mirror/commits?pagelen=1`, fetched 2026-09-03 | full (top commit `728c79c92e88eb0074db4d8f28591f799ca9917b`) |
| `design-system/tokens/package.json` at pinned commit | `https://api.bitbucket.org/2.0/repositories/atlassian/atlassian-frontend-mirror/src/728c79c92e88eb0074db4d8f28591f799ca9917b/design-system/tokens/package.json` | full |
| `design-system/tokens/src/artifacts/` listing at pinned commit | `https://api.bitbucket.org/2.0/repositories/atlassian/atlassian-frontend-mirror/src/728c79c92e88eb0074db4d8f28591f799ca9917b/design-system/tokens/src/artifacts/?pagelen=100` | full (filenames only, `token-data.json5` confirmed present) |
| `design-system/button/` listing (Storybook check) | `https://api.bitbucket.org/2.0/repositories/atlassian/atlassian-frontend-mirror/src/728c79c92e88eb0074db4d8f28591f799ca9917b/design-system/button/?pagelen=100`, fetched 2026-09-03 | full (no `.storybook` directory found) |
