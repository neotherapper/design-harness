---
title: Token source of truth
date: 2026-09-03
question: DTCG, Style Dictionary, Figma variables and Code Connect via the Dev Mode MCP: what is machine-readable, what drifts?
---

## Summary

The chain from a designer's tokens to a component's code passes through at least three formats and two manual steps. Figma variables are the design-time source. The W3C Design Tokens Community Group (DTCG) format is the closest thing to a portable file format for them. Style Dictionary is the translation tool that turns a DTCG or Style Dictionary v3 JSON file into CSS custom properties or TypeScript. The DTCG Format Module (version 2025.10) is a stable, implementation-ready community group report, not a W3C standard, and Style Dictionary's own docs say it does not yet fully support that latest version. Figma's Dev Mode MCP server exposes a `get_variable_defs` tool, but Figma's own troubleshooting docs say the agent has to guess which tool to call and can return hard-coded code instead of a variable reference. Figma variables also carry an optional, hand-typed "Code Syntax" alias per platform that is not verified against any real codebase. None of DTCG, Style Dictionary, or the Figma MCP tools gate anything by themselves; they report design intent or translate a file format. The only artifact a validator can mechanically compare a component's values against is the built token output: a generated CSS or TS file, checked into the same repository as the component.

## DTCG format

Source: `design-tokens/community-group`, commit `16c902d9327c18290e956a21130c445f1b88c40f` (`gh api repos/design-tokens/community-group/commits/main --jq .sha`).

### Status

The repository's `main` branch is a work-in-progress preview. The format spec's own build config sets `specStatus: 'CG-DRAFT'` and `isPreview: true`, and its status section reads, verbatim:

> "This is a **preview draft** of in progress changes. Do not refer to this document directly, and do not implement anything in this document."

(`technical-reports/format/index.html`, around lines 78 to 93 at the pinned commit). That preview points to a published, dated version: `latestVersion: 'https://www.designtokens.org/TR/2025.10/format/'`. Fetching that published page (vendor doc, no commit SHA available for a rendered spec page; fetched 2026-09-03) shows a different status: "Design Tokens Format Module 2025.10", published 28 October 2025, with the stability statement "This specification is considered stable. Further updates will be provided in superseding specifications," issued as a Final Community Group Report under the W3C Community Final Specification Agreement, and explicitly "not a W3C Standard nor... on the W3C Standards Track." In short, 2025.10 is DTCG's current stable release; `main` on GitHub is already past it, in draft.

### File shape

Design token files are plain JSON (`technical-reports/format/file-format.md`). A token is any JSON object carrying a `$value` key; the parent key is the token's name. Minimal example from the spec (`design-token.md`):

```json
{
  "token name": {
    "$type": "color",
    "$value": { "colorSpace": "srgb", "components": [1, 0, 0] }
  }
}
```

`$value` and an unambiguous `$type` are the only required pieces. `$type` can be set directly on a token or inherited from an ancestor group, and a token with no resolvable type "MUST" be treated as invalid by conforming tools (`types.md`). Groups are simply any object without `$value`; they can carry `$description`, `$type` as a default for children, `$extends`, and `$deprecated` (`groups.md`). A group can also hold a `$root` token so it has both a base value and named variants without an ambiguous group/token reference (`groups.md`; the source does not say in which version this arrived).

References ("aliases") come in two forms (`aliases.md`):

- Curly-brace syntax, `{group.token}`, which always resolves to another token's whole `$value`.
- JSON Pointer syntax, `$ref: "#/group/token/$value/..."` (RFC 6901), for pointing at a specific property inside a value, which conforming tools "MUST support."

The spec's resolution algorithm requires a tool to look up the referenced token, validate that it has a `$value`, and check for cycles before substituting. It explicitly says tools "SHOULD preserve references and therefore only resolve them whenever the actual value needs to be retrieved," so premature flattening is discouraged but not forbidden.

### Types

`types.md` and `composite-types.md` at the pinned commit define seven primitive types (color, dimension, font family, font weight, duration, cubic Bezier, number) and six composite types built from them (stroke style, border, transition, shadow, gradient, typography). `dimension`, for example, must be an object with a numeric `value` and a `unit` restricted to `"px"` or `"rem"`; a `unit` of anything else, or a missing `unit` even when the value is `0`, is invalid. Font weight accepts either a number 1 to 1000 or a fixed table of string aliases (`"bold"` equals 700, and so on); anything else "MUST be rejected by tools." This precision is what a mechanical validator leans on for type-level checks, separate from whether a component actually used the token at all.

## Style Dictionary

Source: `style-dictionary/style-dictionary`, commit `29f1b25f3d05d8f264de7814b52919b3dd5dca96`. Note: `amzn/style-dictionary`, the name given in the research brief, now 301-redirects to `style-dictionary/style-dictionary` at the GitHub API level, confirmed with `curl -sI https://api.github.com/repos/amzn/style-dictionary`, which returned `location: https://api.github.com/repositories/75121110`. Both org names resolve to the same commit.

### DTCG support

`docs/src/content/docs/info/DTCG.mdx` states: "**As of version 4**, Style Dictionary has first-class support for the DTCG format," but immediately qualifies it:

> "Important note: the latest format 2025.10 does not have full support yet in Style Dictionary. This is a work in progress in v5"

with a link to `style-dictionary/style-dictionary#1590`. So the tool that turns tokens into code is, by its own documentation, behind the current stable spec version, a structural source of drift between "a spec-conformant token file" and "a file Style Dictionary can actually build correctly." `docs/src/content/docs/info/tokens.md` adds that Style Dictionary v4 accepts either its own legacy shape (`value`/`type`/`comment`) or the DTCG shape (`$value`/`$type`/`$description`), but the two "cannot be combined inside a single Style Dictionary instance," and its own docs still mostly show the legacy shape.

### What it generates

`docs/src/content/docs/reference/Hooks/Formats/predefined.md` lists Style Dictionary's built-in output formats. Confirmed present: `css/variables` ("Creates a CSS file with variable definitions"), and two TypeScript formats, `typescript/es6-declarations` and `typescript/module-declarations` ("Creates TypeScript declarations for ES6 modules" and "for module"), alongside SCSS, Less, Stylus, several JavaScript module shapes, Android, iOS/Swift, and Compose formats. `styledictionary.com` (fetched 2026-09-03) states the same intent on its homepage: "Export your Design Tokens to any platform - iOS, Android, CSS, JS, HTML" and calls itself "Forward-compatible with Design Tokens Community Group spec."

### How references resolve

`docs/src/content/docs/info/tokens.md` documents the same curly-brace alias syntax as DTCG (`"{size.font.medium}"`). What happens to that alias at build time is controlled per format by an `outputReferences` option, documented in `docs/src/content/docs/reference/Hooks/Formats/index.md`. With a chain `color.red` to `color.danger` to `color.error`, the doc's own example shows:

- `outputReferences: false`, the documented default: the alias chain is fully resolved, and the output CSS has three independent `--color-red`, `--color-danger`, `--color-error` variables, all set to the literal `#ff0000`. The reference is gone from the artifact.
- `outputReferences: true`: the output keeps the chain as `var()` calls, `--color-danger: var(--color-red);`, so the semantic link from `error` back to `red` survives into the built CSS.

The doc lists which formats even support the option (`css/variables`, `scss/variables`, `less/variables`, `android/resources`, `compose/object`, `ios-swift/class.swift`, `flutter/class.dart`, not plain JSON). The same DTCG source can therefore build into two structurally different artifacts depending on a config flag most component authors never see, and a validator that expects to trace `var(--color-error)` back to a primitive has to know which mode the build ran in.

## Figma variables, Dev Mode MCP, Code Connect

### Figma variables

`help.figma.com/hc/en-us/articles/15339657135383` (fetched 2026-09-03) describes variables as storing "reusable values that can be applied to all kinds of design properties," usable by "[a]nyone with access to a file," and publishable to team libraries only on paid plans.

`help.figma.com/hc/en-us/articles/15145852043927` (fetched 2026-09-03) documents **Code Syntax**: an optional, per-variable, per-platform alias. "You can create one name per platform, including Web, Android, and iOS... up to three code syntaxes per variable," set by hand: "From the Code syntax section of the Edit variable modal, click Add code syntax... enter a variable name." This is the mechanism meant to make a Figma variable's Dev Mode name match a real code identifier, for example a variable named "Extra Small" in Figma showing as `var(--extra-small)` in a CSS snippet. The same page states only that the syntax "appear[s] in code snippets in Dev Mode when inspecting elements"; nothing in this or any other fetched document says the string is checked against, or generated from, an actual codebase. It is free text a person types once.

### Dev Mode MCP server

`developers.figma.com/docs/figma-mcp-server/tools-and-prompts/` (fetched 2026-09-03) lists the tool `get_variable_defs`, described in one line as "Variables and styles used in a selection," alongside the separate tool `get_code_connect_map`, "Retrieves node ID to code component mappings." These are distinct tools: one surfaces variable and token data, the other surfaces which code component a Figma node maps to.

Whether an agent actually calls `get_variable_defs` is not guaranteed by the protocol. Figma's own troubleshooting page, `developers.figma.com/docs/figma-mcp-server/variables-vs-code/` (fetched 2026-09-03), exists specifically because of this failure mode:

> "If you intended to get variable names and values (like colors or spacing tokens), but your AI assistant returned code instead, it probably didn't trigger the correct MCP tool... By default, the agent chooses which tool to call based on your prompt. But sometimes it guesses wrong."

Its fix is to make the prompt more explicit ("Get the variable names and values for this selection"), which is a workflow instruction, not a protocol guarantee. The presence of `get_variable_defs` in the tool list does not by itself mean any given generated snippet used it; that has to be checked after the fact, in the generated code, not assumed from the tool's existence.

`help.figma.com/hc/en-us/articles/32132100833559` (fetched 2026-09-03) describes the server in looser terms: it lets the client "[p]ull in variables, components, and layout data directly into your IDE," and separately states of Code Connect: "Boost output quality by reusing your actual components. Code Connect keeps your generated code consistent with your codebase." Neither fetched Figma page states that the name returned by `get_variable_defs` is guaranteed identical to the identifier used in the codebase; the closest thing to that guarantee is the manually typed Code Syntax field above, and Code Connect's mapping is about components and props, not variable names, as the next section shows.

### Figma Code Connect

Source: `figma/code-connect`, commit `f55fc3a8d9392df0dd80a9e859ffdddbd5c50019`. The repository README states its purpose plainly: "Code Connect is a tool for connecting your design system components in code with your design system in Figma. When using Code Connect, Figma's Dev Mode will display true-to-production code snippets from your design system instead of autogenerated code examples." Framework-specific parsers are deprecated in favor of "template files," per a warning banner at the top of the README pointing to a migration guide.

The hosted docs page for the React integration, `developers.figma.com/docs/code-connect/react` (fetched 2026-09-03; the repo's own `docs/react.md` at the pinned commit just redirects readers to this URL), shows the mapping shape:

```javascript
figma.connect(Button, 'https://...', {
  props: {
    label: figma.string('Text Content'),
    disabled: figma.boolean('Disabled'),
    type: figma.enum('Type', { Primary: 'primary', Secondary: 'secondary' }),
  },
  example: ({ disabled, label, type }) => (
    <Button disabled={disabled} type={type}>{label}</Button>
  ),
})
```

The example above is reformatted for width and is not byte-exact to the page.

A Code Connect mapping declares which Figma component URL corresponds to which code component, a `props` object translating Figma's design properties (variant names, boolean layer properties, text content) into that component's code props, and an `example` function showing the actual import and usage shape Dev Mode should display. This is a component-and-prop mapping, authored and published by hand and checked in like source code, per the repo's CLI. It does not declare a mapping from Figma variable names to design-token identifiers in code; that is the separate, weaker Code Syntax mechanism described above.

### Tokens Studio for Figma

Checked only lightly, per the rule to skip anything not quickly verifiable. `docs.tokens.studio` (fetched 2026-09-03) states that it is "a Figma Plugin allowing you to integrate Tokens into your Figma designs," that it supports "Token Format - W3C DTCG vs Legacy," and that it integrates with "Style Dictionary + SD Transforms" for developer handoff. Two follow-up fetches to guess deeper pages about whether Tokens Studio tokens two-way sync with Figma's native Variables feature (`docs.tokens.studio/sync/figma`, `docs.tokens.studio/figma-variables/overview`) both 404'd, so whether a Tokens Studio token is the same underlying object as a native Figma variable, or a parallel system that has to be pushed or pulled, is left as **not checked** here rather than guessed at.

## The chain, end to end

```
Figma variable (designer edits)
   |  [manual]  optional "Code Syntax" alias typed once per platform, never re-verified
   v
Dev Mode inspect panel / MCP get_variable_defs
   |  [semi-automated] tool call happens IF the agent's prompt makes the intent explicit;
   |                   otherwise the agent may emit hard-coded values instead (Figma's own
   |                   "variables-vs-code" troubleshooting doc exists because of this)
   v
Exported token file: DTCG $value/$type JSON, or a plugin export (e.g. Tokens Studio), or
hand-copied values
   |  [manual export step] no fetched source shows this step happening automatically inside
   |                   plain Figma; it is a plugin action or a person's copy-paste
   v
tokens.json (source of truth committed to the repo)
   |  [automated, once configured] `style-dictionary build`, IF the file's DTCG version is one
   |                   Style Dictionary v4/v5 actually supports (2025.10 is explicitly not yet
   |                   fully supported per Style Dictionary's own docs)
   v
CSS custom properties (css/variables) / TypeScript declarations (typescript/*-declarations)
   |  [config-dependent] `outputReferences` true keeps the alias chain as var() references;
   |                   false, the documented default, flattens every alias to its literal value
   v
Component code (imports the generated CSS var or TS constant, or a person/agent hand-types a
literal that happens to match)
```

Every arrow not marked `[automated]` is a place where the string naming a token in one artifact can silently stop being the same string, or the same token, by the time it reaches the next artifact.

## Drift table

| Drift kind | Where it enters | Mechanically detectable (how) or needs a human |
|---|---|---|
| Figma variable renamed, or its Code Syntax alias edited, without updating the generated code token | Designer edits the variable's name or its hand-typed Code Syntax field; nothing re-checks it | Needs a human, unless the harness also pulls current Figma variable and Code Syntax metadata via the Figma API and diffs it against the generated CSS/TS names in the same run. That integration is not shown by any fetched Figma doc to exist out of the box |
| Variable exists in a Figma file but was never published to the library it feeds | Design time; "Anyone with can edit access... can create and manage variables" but publishing is a separate, plan-gated action | Not detectable from the codebase side at all; needs either a Figma API check against published-library variables, or a human confirming publish state |
| Agent calls a code-generation tool without triggering `get_variable_defs`, so it emits a hard-coded literal instead of a variable reference | Generation time, inside the Dev Mode MCP flow; documented by Figma itself as a frequent failure ("it probably didn't trigger the correct MCP tool") | Mechanically detectable after the fact: a token-conformance linter scans the generated component's literal values (colors, dimensions, etc.) against the built token manifest's known values and names, and flags any raw literal that is not a token reference |
| Agent fabricates a plausible-sounding but nonexistent variable or token name | Generation time; models can hallucinate identifiers that look like real tokens | Mechanically detectable: validator resolves every token reference in the component against the generated manifest (Style Dictionary output or the DTCG source); an unresolvable name fails |
| DTCG source uses a 2025.10 feature Style Dictionary does not fully support yet (per Style Dictionary's own DTCG.mdx) | Build step | Partially detectable: outright unsupported constructs should fail or warn at build time, which is detectable as a build error; silent mis-transformation of a construct Style Dictionary accepts but interprets differently needs a human diff of source semantics against build output |
| `outputReferences: false`, the documented default, flattens an alias chain to a literal in the built CSS/TS, discarding the semantic link from a derived token back to its primitive | Style Dictionary build config, per platform and file | Mechanically detectable that it happened, by inspecting the platform config for the flag or diffing whether the built file contains `var(...)` chains or only literals; the design intent behind the alias chain, whether `error` should really track `red`, needs a human |
| Hard-coded value in component source that happens to equal a token's current value | Component authoring, by a human or an agent, at any time | Fully mechanically detectable: this is the class of check the design-harness's validator exists to run, comparing every literal value in the component against the token manifest's value set and flagging matches or near-matches that are not spelled as a reference |
| Code Connect mapping goes stale, the component's props changed but the `figma.connect` file was not updated | Code Connect file authored by hand, published via CLI, not regenerated automatically on every commit | Mechanically detectable only if CI runs the Code Connect CLI's own validation or publish step against the current component source; otherwise needs a human to notice Dev Mode showing an outdated snippet |

## What a validator compares against

The one artifact that is both machine-readable and lives in the same repository as the component is the **built token output**: a Style Dictionary-generated CSS custom properties file and/or its TypeScript declaration counterpart, built from a committed DTCG (or Style Dictionary v3-shape) `tokens.json`. That pair, source JSON plus generated CSS/TS, is what a deterministic validator can actually open at lint time, without needing live Figma credentials.

- **Presence and shape check (inner loop):** for every value in a component's source that looks like a color, dimension, font, duration, or other DTCG primitive or composite type, the validator checks whether that value is expressed as a reference to a name present in the generated token file, a `var(--x)` in CSS or an imported constant from the generated TypeScript module, rather than a raw literal. This is a presence-and-shape check, not a truth check: it can tell you a component used *some* token, not that it used the *right* one for this context (zygos ADR-0001's guard framing applies the same way here).
- **Resolvability check:** any token name a component references must exist as a key in the built manifest. An agent-fabricated or renamed-and-not-rebuilt name fails this check immediately.
- **What it cannot check from the codebase alone:** whether the *design* still matches what a designer currently has in Figma, renames, unpublished variables, whether the Code Syntax alias is still accurate. Those require either a live Figma API round trip wired into the same validation step, or a human review, per the design-harness's own "Two layers" split between deterministic inner-loop checks and judgment sent to a review job.

In short, DTCG and Style Dictionary together can produce the comparison target. The Figma-side tools (variables, Code Syntax, Dev Mode MCP, Code Connect) can produce or assist in producing the *source* of that target, but none of them, by themselves, gate a component's publication; they report design intent or translate a file format. Gating is something the harness's own outer-gate guard has to do, by refusing to flip a `Component`'s publish flag until this comparison has run and passed.

## What I did not check

- Corrected after the 2026-09-03 fidelity review: a version claim about `$root` that no source dated, an invented list of Figma variable types (removed rather than re-sourced), and three quotes that had drifted from their sources by a dash, a case change, and reformatting.

- The Dev Mode MCP server's exact JSON response shape for `get_variable_defs`: field names, whether it returns DTCG-style `$type`/`$value` or a Figma-internal shape. No fetched document shows a sample response body, only a one-line tool description.
- Whether Tokens Studio's tokens are the same underlying object as native Figma Variables (two-way synced) or a separate system requiring an explicit push/pull. Two guessed doc URLs 404'd; left as not checked rather than dug for further, per the instruction to skip what isn't quickly verifiable.
- Figma's REST/Variables API directly. Only help-center and developer-docs pages were fetched, so claims about "published" versus "local" variable visibility rest on that wording, not the API contract.
- Style Dictionary v5's actual DTCG 2025.10 support status beyond the one linked GitHub issue (`style-dictionary/style-dictionary#1590`), which itself was not opened, only referenced from `DTCG.mdx`. Also not opened: the `cli/` directory's source, so the Code Connect CLI's publish and validate behavior in CI, and whether it can fail a pipeline on a stale mapping, is unverified.
- HTML, SwiftUI, and Jetpack Compose Code Connect integrations, and the DTCG `color/` and `resolver/` technical-report directories beyond the Format Module: all listed in their repos but not opened.
- Figma pricing and plan boundaries beyond the one quoted line about variables in libraries requiring a paid plan; not checked against a pricing page.

## Sources

| Source | Pinned URL | Read: full/part |
|---|---|---|
| DTCG repo root README | `https://raw.githubusercontent.com/design-tokens/community-group/16c902d9327c18290e956a21130c445f1b88c40f/README.md` | part |
| DTCG technical-reports README (build/deploy notes, draft URL) | `https://raw.githubusercontent.com/design-tokens/community-group/16c902d9327c18290e956a21130c445f1b88c40f/technical-reports/README.md` | full |
| DTCG format spec build config and SOTD (`index.html`) | `https://raw.githubusercontent.com/design-tokens/community-group/16c902d9327c18290e956a21130c445f1b88c40f/technical-reports/format/index.html` | part |
| DTCG design-token.md ($value, $description, naming) | `https://raw.githubusercontent.com/design-tokens/community-group/16c902d9327c18290e956a21130c445f1b88c40f/technical-reports/format/design-token.md` | full |
| DTCG file-format.md (JSON, MIME type, extensions) | `https://raw.githubusercontent.com/design-tokens/community-group/16c902d9327c18290e956a21130c445f1b88c40f/technical-reports/format/file-format.md` | full |
| DTCG aliases.md (curly-brace and JSON Pointer refs) | `https://raw.githubusercontent.com/design-tokens/community-group/16c902d9327c18290e956a21130c445f1b88c40f/technical-reports/format/aliases.md` | full |
| DTCG types.md (primitive types) | `https://raw.githubusercontent.com/design-tokens/community-group/16c902d9327c18290e956a21130c445f1b88c40f/technical-reports/format/types.md` | part |
| DTCG composite-types.md (composite types) | `https://raw.githubusercontent.com/design-tokens/community-group/16c902d9327c18290e956a21130c445f1b88c40f/technical-reports/format/composite-types.md` | part |
| DTCG groups.md ($root, group properties) | `https://raw.githubusercontent.com/design-tokens/community-group/16c902d9327c18290e956a21130c445f1b88c40f/technical-reports/format/groups.md` | full |
| DTCG terminology.md (glossary, tool categories) | `https://raw.githubusercontent.com/design-tokens/community-group/16c902d9327c18290e956a21130c445f1b88c40f/technical-reports/format/terminology.md` | part |
| DTCG Format Module 2025.10, published (stability statement) | `https://www.designtokens.org/TR/2025.10/format/`, vendor doc, fetched 2026-09-03 | part |
| Style Dictionary repo listing (confirms `amzn/style-dictionary` redirect) | `https://api.github.com/repos/style-dictionary/style-dictionary` at commit `29f1b25f3d05d8f264de7814b52919b3dd5dca96` | part |
| Style Dictionary DTCG.mdx (support status, v5 gap) | `https://raw.githubusercontent.com/style-dictionary/style-dictionary/29f1b25f3d05d8f264de7814b52919b3dd5dca96/docs/src/content/docs/info/DTCG.mdx` | full |
| Style Dictionary tokens.md (file shape, aliasing, legacy vs DTCG) | `https://raw.githubusercontent.com/style-dictionary/style-dictionary/29f1b25f3d05d8f264de7814b52919b3dd5dca96/docs/src/content/docs/info/tokens.md` | part |
| Style Dictionary predefined formats (css/variables, typescript/*-declarations) | `https://raw.githubusercontent.com/style-dictionary/style-dictionary/29f1b25f3d05d8f264de7814b52919b3dd5dca96/docs/src/content/docs/reference/Hooks/Formats/predefined.md` | part |
| Style Dictionary formats index (`outputReferences` behavior) | `https://raw.githubusercontent.com/style-dictionary/style-dictionary/29f1b25f3d05d8f264de7814b52919b3dd5dca96/docs/src/content/docs/reference/Hooks/Formats/index.md` | part |
| styledictionary.com homepage | `https://styledictionary.com/`, vendor doc, fetched 2026-09-03 | part |
| Figma Code Connect repo README | `https://raw.githubusercontent.com/figma/code-connect/f55fc3a8d9392df0dd80a9e859ffdddbd5c50019/README.md` | full |
| Figma Code Connect React docs pointer (`docs/react.md` at pinned commit) | `https://raw.githubusercontent.com/figma/code-connect/f55fc3a8d9392df0dd80a9e859ffdddbd5c50019/docs/react.md` | full (one line, redirects elsewhere) |
| Figma Code Connect React mapping example (hosted docs) | `https://developers.figma.com/docs/code-connect/react`, vendor doc, fetched 2026-09-03 | part |
| Figma help center: Dev Mode MCP server guide | `https://help.figma.com/hc/en-us/articles/32132100833559-Guide-to-the-Dev-Mode-MCP-Server`, vendor doc, fetched 2026-09-03 | part |
| Figma help center: guide to variables | `https://help.figma.com/hc/en-us/articles/15339657135383-Guide-to-variables-in-Figma`, vendor doc, fetched 2026-09-03 | part |
| Figma help center: overview of variables, collections, and modes (Code Syntax) | `https://help.figma.com/hc/en-us/articles/15145852043927-Overview-of-variables-collections-and-modes`, vendor doc, fetched 2026-09-03 | part |
| Figma developer docs: MCP tools and prompts (tool list) | `https://developers.figma.com/docs/figma-mcp-server/tools-and-prompts/`, vendor doc, fetched 2026-09-03 | part |
| Figma developer docs: Code Connect integration with MCP | `https://developers.figma.com/docs/figma-mcp-server/code-connect-integration/`, vendor doc, fetched 2026-09-03 | part |
| Figma developer docs: "Tried to fetch variables, but got code instead" | `https://developers.figma.com/docs/figma-mcp-server/variables-vs-code/`, vendor doc, fetched 2026-09-03 | full (short page) |
| Tokens Studio docs homepage | `https://docs.tokens.studio/`, vendor doc, fetched 2026-09-03 | part |
