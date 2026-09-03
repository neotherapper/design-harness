---
title: Deterministic validators
date: 2026-09-03
question: Which token, accessibility, type, story, and visual checks exist, and does each gate or only report?
---

## Summary

Every layer has at least one tool that can gate a CI run through a process exit code, but the
default configuration of almost every tool is REPORT, not GATE. Token conformance in plain CSS is the
most mature: stylelint's own severity and exit-code model (0 clean, 1 fatal error, 2 lint problem) is
a real gate, and a plugin rule like `scale-unlimited/declaration-strict-value` can force `var()` instead
of a literal color. The equivalent for TSX/JSX is thinner and mostly AST pattern-matching on JSX
attributes (MetaMask's `color-no-hex`) or on Tailwind class strings (`eslint-plugin-tailwindcss`
`no-arbitrary-value`), and both ship with non-blocking default severities. Accessibility splits cleanly
into two different mechanisms that are often confused: `eslint-plugin-jsx-a11y` is a static, lint-time
AST check, and Storybook's a11y addon is a render-time DOM audit built on `axe-core`; the addon only
fails a test run, in the CLI/CI sense, when a story's `parameters.a11y.test` is explicitly set to
`'error'` and it runs through the Vitest addon or the test-runner, otherwise it is panel-only. Types
(`tsc`) and lint (`eslint`) are the strongest deterministic gates in the whole stack: same input and
config reliably produce the same exit code, and both run without a browser in low single-digit seconds
on a component-sized diff. Story presence and interaction tests (`play` functions) are deterministic in
outcome but need a real browser (Playwright, via the Vitest addon or the Jest-based test-runner), so
they are an order of magnitude slower than `tsc`/`eslint`. Visual regression is the one category that is
structurally not just deterministic-vs-heuristic: Chromatic's pixel diff is deterministic, but its
gate is a human accept/deny of an intentional change, wired through a required PR status check, not
purely through the CLI exit code; Playwright's `toHaveScreenshot` is the same shape at unit scale, with
committed baseline images standing in for the human-reviewed baseline. No primary source found here
validates that a token reference in code actually resolves to an entry in a DTCG-format `tokens.json`;
the closest tool needs a pre-flattened custom-properties JSON, not the raw token tree.

## Token conformance (CSS and TSX)

**stylelint itself is the enforcement engine, not a rule.** Its CLI documents four exit codes: `1`
fatal error, `2` lint problem, `64` invalid CLI usage, `78` invalid configuration file (`docs/user-guide/cli.md`,
pinned to commit `ac19a3c04ca20ab57d637f81b57365539c1be84b`). Each rule has a `severity` of `"error"`
(default) or `"warning"`, configurable per rule or project-wide via `defaultSeverity`
(`docs/user-guide/configure.md`, same commit). A rule set to `"warning"` still reports but a reporter
can choose to treat it differently; the CLI's own exit-2 behavior is driven by whether any problem
(error or warning) was found at all, so the practical gate decision is "is this rule enabled," not just
its severity label.

**`stylelint-declaration-strict-value`** (the actively maintained fork lives at
`github.com/AndyOGo/stylelint-declaration-strict-value`, not the `YozhikM` org named in the research
brief, which no longer resolves) ships the rule `scale-unlimited/declaration-strict-value`. Its README,
pinned to commit `ba7401895ae82c904b6e5d0418237e1fa3fc12ee`, states it "enforces either variables
(`$sass`, `namespace.$sass`, `@less`, `var(--cssnext)`, `css-loader @value`), functions or custom CSS
values ... for CSS longhand and experimental shorthand properties." Configuring it on `color` makes
`a { color: #FFF; }` a violation and `a { color: var(--color-white); }` clean. It is deterministic
AST/value matching (PostCSS-based), not a heuristic, and it inherits stylelint's own gate: whatever
severity the rule is given, stylelint's exit code reflects it. The rule also supports `autoFixFunc`
for `--fix`, so it can rewrite a literal into a variable reference automatically in some cases.

**Cross-referencing against an actual token list.** `csstools/stylelint-value-no-unknown-custom-properties`
(commit `399282b3233429c9ee090f57f204e9ce6e9cc2b2`) goes one step further than "is this a `var()`": its
rule `csstools/value-no-unknown-custom-properties` fails when a `var(--brand-blue)` reference has no
matching `--brand-blue` declaration anywhere in scope. Custom properties can be supplied via
`importFrom`, including a JSON file shaped like `{ "custom-properties": { "--brand-blue": "#33f" } }`.
This is the closest primary-source match to "validate that the value exists in tokens.json," but the
shape it wants is a flat map of custom-property name to value, not a DTCG token tree with `$value`/`$type`
nodes and aliasing; a build step (for example, a Style Dictionary CSS-variables build) would have to sit
between the token source of truth and this rule's `importFrom` input. No tool was found that consumes a
DTCG `tokens.json` directly for this check.

**CSS-in-JS is a soft spot.** stylelint's `customSyntax` option (`docs/user-guide/options.md`, same
commit as above) lets stylelint parse styled-components/emotion template literals via a syntax module,
but the same page states plainly: "Stylelint can provide no guarantee that core rules work with custom
syntaxes." That caveat applies to plugin rules too; nothing in the primary sources checked here confirms
`declaration-strict-value` or the unknown-custom-properties rule works reliably against a CSS-in-JS
syntax module.

**TSX inline styles and Tailwind classnames.** `eslint-plugin-tailwindcss` (commit
`1ace2c54aa7c7d4b9590ffe02ebabdfe66bc2382`, default branch `v4`) ships `no-arbitrary-value`, described
as forbidding classnames like `w-[20rem]` in favor of a defined Tailwind theme value. Its own docs
header states it is "disabled in the recommended config." The companion `no-custom-classname` rule
flags any classname not recognized by Tailwind, and its docs header states it "warns in the recommended
config" (so it reports, non-blocking, out of the box; a project has to raise it to `"error"` to gate).
Both are deterministic string/AST matching against the resolved Tailwind config, not heuristics.

For inline `style={{ }}` objects and non-Tailwind CSS-in-JS, `@metamask/eslint-plugin-design-tokens`
(commit `daf6e7cd9bab3e9a47551c4cbb808c4e10695354`) is a real, in-production example (MetaMask's own
design tokens). Its `color-no-hex` rule doc gives exactly the shape this repo needs: `<div style={{ color: '#E06470' }}>` is flagged, `<div style={{ color: 'var(--color-error-default)' }}>` and
`<div style={{ color: theme.error.default }}>` are not. The example configuration in its own README
sets the rule to `"warn"`, so, like the Tailwind rules, it reports by default and a project must
explicitly choose `"error"` to make it a gate. It is deterministic (literal hex string match on a JSX
attribute value), and its own docs concede the obvious bypass: computed values, string concatenation,
or values coming from a non-literal expression are outside what a static AST rule can see.

## Accessibility

Two mechanisms are commonly conflated and are not the same thing.

**Lint-time, static.** `eslint-plugin-jsx-a11y` (commit `8f75961d965e47afb88854d324bd32fafde7acfe`)
"does a static evaluation of the JSX to spot accessibility issues in React apps," and its own README is
explicit about the boundary: "Because it only catches errors in static code, use it in combination with
[`@axe-core/react`] to test the accessibility of the rendered DOM." This runs at lint speed, needs no
browser, and gates exactly like any other ESLint rule (severity plus `--max-warnings`/exit code). It
cannot catch anything that depends on runtime DOM structure, computed styles, or actual contrast.

**Render-time, DOM-based.** `axe-core` (commit `540ecae993b5b0f40070c6b034843b0392956175`) audits the
rendered DOM against WCAG-derived rules and best practices. Its own README states it "returns zero false
positives (bugs notwithstanding)" but is explicit that some checks land in an "incomplete" bucket
requiring manual confirmation, and that `color-contrast` "is known not to work with JSDOM," so it needs
a real (or real-enough) browser environment to be trustworthy.

**Storybook's a11y addon** wraps `axe-core` and runs it against each story's rendered output
(`docs/writing-tests/accessibility-testing.mdx`, Storybook commit `4bd806b927f82b14057e0c143b064b89663f771b`).
By itself, opening a story runs the audit and shows results in an addon panel: this is REPORT only,
nothing fails. The doc's own test-behavior table is the key GATE/REPORT switch:

| `parameters.a11y.test` value | Behavior |
|---|---|
| `'off'` | Do not run accessibility tests (manual check only via the panel) |
| `'todo'` | Run tests; violations return a warning in the Storybook UI |
| `'error'` | Run tests; violations return a failing test in the Storybook UI **and CLI/CI** |

The docs state directly: "Accessibility tests will only produce errors in CI if you have set
`parameters.a11y.test` to `'error'`. If you set it to `'todo'`, there will be no accessibility-related
errors, warnings, or output in CI." So the gate is opt-in per story/component/project, not a property of
installing the addon. When it is set to `'error'`, the tests execute through either the Vitest addon
(`vitest` CLI, standard non-zero exit on failure) or the Jest/Playwright-based test-runner, both of
which need a real browser (Playwright's Chromium, launched by Vitest browser mode, or by
`jest-playwright` under the test-runner), so this is not a "seconds, no browser" inner-loop check; it is
closer to an integration-test-speed step.

## Types, lint, formatting

**TypeScript (`tsc`).** Pinned to the `v7.0.2` tag (commit `1e4744d68260a7cb91b62b12edc3f6a2187faaf1`,
the last tag whose source tree still has the classic TypeScript-implemented compiler under `src/`; the
`main` branch has since become a from-scratch Go reimplementation and was not used for this claim), the
compiler's own `ExitStatus` enum in `src/compiler/types.ts` is:

```
Success = 0
DiagnosticsPresent_OutputsSkipped = 1
DiagnosticsPresent_OutputsGenerated = 2
```

Both `1` and `2` are non-zero, so `tsc` gates on type errors by exit code even in its default
configuration, where it emits JS output alongside the errors (`OutputsGenerated`, code `2`); the
`--noEmitOnError`/`noEmitOnError` compiler option changes only whether files are written, not whether
the process exits non-zero. This is fully deterministic given a fixed `tsconfig.json` and source tree,
needs no browser, and on a component-sized change is fast enough for an agent's edit loop. Its
limitation is scope: `tsc` type-checks; it has no concept of a design token, an ARIA attribute, or a
story.

**ESLint.** Commit `1696682791661c13167eb905da2f38d1b8f4a3bf`, `docs/src/use/command-line-interface.md`:
exit code `0` when linting succeeds and warnings are within `--max-warnings`; `1` when there is at least
one error or more warnings than `--max-warnings` allows; `2` on a configuration or internal error. This
is the actual gate mechanism behind every ESLint-based rule discussed above (Tailwind, MetaMask design
tokens, jsx-a11y): the rule can flag a violation, but whether that flag becomes a failing build is a
function of the rule's configured severity plus this exit-code table, not something the rule itself
decides. Fast, no browser, deterministic given a fixed config and ESLint/plugin version.

**Prettier**, as the formatting baseline: `prettier --check` "will return exit code `1`" when files are
not formatted according to its rules (`docs/cli.md`, commit `338310e438be8bac2efcb1db4da313c046d45eef`),
and `2` if something is wrong with Prettier itself. This is a real gate in `--check` mode, deterministic,
fast, and orthogonal to `tsc`/`eslint`: it checks formatting, not types or semantics, and does not know
what a design token is.

## Stories and interaction tests

A Storybook story is a precondition for every other test type discussed here (a11y, visual, interaction)
rather than a check in itself, but Storybook does run one check automatically: a **render test**, "a
simple version of an interaction test that only tests the ability of a component to render successfully
in a given state" (`docs/writing-tests/interaction-testing.mdx`, same Storybook commit as above). This
means "story exists and renders without throwing" is itself a deterministic, gate-capable check once run
through the Vitest addon or the test-runner.

**Interaction tests (`play` functions).** A story's `play` function simulates user behavior via Testing
Library-style queries (`getByRole`, `findByText`, and so on) against a `canvas` scoped to that story, and
makes assertions with `expect`. The Vitest addon's "How it works" section states: "Stories are tested in
two ways: a smoke test to ensure it renders and, if a play function is defined, that function is run and
any assertions made within it are validated." Both are deterministic pass/fail outcomes.

**Running them.** The Vitest addon transforms stories into Vitest tests via "portable stories" and runs
them "in browser mode, using Playwright's Chromium browser" (`docs/writing-tests/integrations/vitest-addon/index.mdx`).
The CLI form is a normal `vitest` invocation (for example `vitest --project=storybook`), which gates
through Vitest's own standard exit code. For Webpack-based Storybooks without Vite, the older
**test-runner** does the same job, "powered by [Jest] and [Playwright]" (`docs/writing-tests/integrations/test-runner.mdx`);
its docs mark it as superseded by the Vitest addon for Vite projects, but it is still the documented path
for Webpack builds and gates the same way Jest does (non-zero exit on any failing test), with a `--ci`
flag that turns implicit new-snapshot writes into failures instead. Neither path is a "no browser,
seconds" check: both launch a real Playwright browser, so they belong in a slower, integration-test tier
of the loop rather than the tightest inner loop with `tsc`/`eslint`.

## Visual regression

**Chromatic** is the tool Storybook's own docs point to for cross-browser visual testing
(`docs/writing-tests/visual-testing.mdx`, same Storybook commit). Its CLI's exit-code table (from
`chromatic.com/docs/cli/`, fetched 2026-09-03) is:

| Code | Key | Meaning |
|---|---|---|
| 0 | `OK` | Exited successfully |
| 1 | `BUILD_HAS_CHANGES` | Chromatic build has (visual) changes |
| 2 | `BUILD_HAS_ERRORS` | Chromatic build has component errors |
| 3 | `BUILD_FAILED` | Chromatic build failed due to system error |
| 4 | `BUILD_NO_STORIES` | Chromatic build failed because it contained no stories |
| 101 | `GIT_NOT_CLEAN` | Git repository workspace not clean |
| 102 | `GIT_OUT_OF_DATE` | Git repository not up-to-date with remote |

(shortened; the full table also lists quota, payment, Storybook-build, and network-error codes). So the
CLI itself does surface a non-zero exit on any pixel-level change, deterministic pixel diff, not a
heuristic. But Storybook's own guidance describes the actual team-level gate differently: "Once you
successfully set up Chromatic in CI, your pull/merge requests will be badged with a UI Tests check. ...
Make the check required in your Git provider to prevent accidental UI bugs from being merged." That is a
required PR status check, and the workflow explicitly includes a human step: "If the changes are
intentional, accept them as baselines locally. If the changes aren't intentional, fix the story and
rerun the tests." An `autoAcceptChanges` option can auto-approve changes on chosen branches (for example
`main`), and `exitOnceUploaded` can make the CLI return before results are known, both of which remove
the "block until approved" property if enabled. So Chromatic is deterministic at the pixel-diff layer and
gated by a human-in-the-loop status check at the merge layer, not purely by the CLI's own exit code.

**Playwright's `toHaveScreenshot`** (`playwright.dev/docs/test-snapshots`, fetched 2026-09-03) is the
same shape at the level of a single Playwright test: "On first execution, Playwright test will generate
reference screenshots. Subsequent runs will compare against the reference." The comparison "uses the
pixelmatch library," configurable via `maxDiffPixels` (no default value is stated on the page for that
option; an unconfigured project effectively compares for an exact or near-exact match per pixelmatch's
own defaults). A failing comparison fails the Playwright test like any other assertion, giving a normal
non-zero exit code, so this is a real CLI/CI gate, not merely a report. The doc also flags an inherent
limitation: "Browser rendering can vary based on the host OS, version, settings, hardware ... For
consistent screenshots, run tests in the same environment where the baseline screenshots were
generated." Baseline images are meant to be committed to version control and reviewed like code, which
is the same "pixel-deterministic diff, human-reviewed baseline" pattern Chromatic uses, just without a
hosted review UI.

## Table

| check | tool | deterministic or heuristic | gates or reports (how) | runs in an agent edit loop | what it cannot catch |
|---|---|---|---|---|---|
| No literal color/value in plain CSS | stylelint + `scale-unlimited/declaration-strict-value` | deterministic (PostCSS AST) | GATE if rule severity is `error` (default); stylelint CLI exits 1/2 on any problem | yes, seconds, no browser | CSS-in-JS syntaxes (stylelint disclaims rule support there) |
| `var()` reference resolves to a known custom property | csstools `value-no-unknown-custom-properties` | deterministic | GATE via stylelint exit code, if `importFrom` is wired to a flattened custom-properties list | yes, seconds, no browser | raw DTCG `tokens.json`; needs a pre-flattened JSON/CSS list |
| No arbitrary Tailwind values (`w-[20rem]`) | `eslint-plugin-tailwindcss` `no-arbitrary-value` | deterministic (resolved against Tailwind config) | REPORT by default (disabled in `recommended`); GATE only if raised to `error` | yes, seconds, no browser | non-Tailwind classnames, inline styles |
| No unrecognized Tailwind classnames | `eslint-plugin-tailwindcss` `no-custom-classname` | deterministic | REPORT by default (`warn` in `recommended`); GATE if raised to `error` and `--max-warnings 0` | yes, seconds, no browser | Tailwind-valid classnames that still bypass the design system's intent |
| No hex color literal in JSX inline style | `@metamask/eslint-plugin-design-tokens` `color-no-hex` | deterministic (AST literal match) | REPORT by default (sample config uses `warn`); GATE if raised to `error` | yes, seconds, no browser | computed/concatenated values, non-literal expressions |
| Static JSX accessibility issues (alt text, invalid ARIA, roles) | `eslint-plugin-jsx-a11y` | deterministic | GATE via ESLint's own exit code, if severities are `error` | yes, seconds, no browser | anything depending on rendered DOM or computed style |
| Rendered-DOM accessibility violations | Storybook a11y addon (`axe-core`) | deterministic given a fixed DOM state; partial rule coverage by axe's own account | REPORT (panel) by default; GATE only when `parameters.a11y.test = 'error'` and run through Vitest addon or test-runner | no, needs a real/real-enough browser; integration-test speed | contrast under JSDOM (documented axe-core gap); anything in the "incomplete" bucket needing manual review |
| Type correctness | `tsc` | deterministic | GATE always: exit code 1 or 2 on any type error, per `ExitStatus` enum | yes, seconds on component-sized diffs, no browser | design tokens, accessibility, visual appearance, story presence |
| Lint rules generally | `eslint` | deterministic given fixed config/version | GATE via exit codes 0/1/2, tunable by severity and `--max-warnings` | yes, seconds, no browser | anything not expressible as a static rule |
| Formatting | `prettier --check` | deterministic | GATE: exit code 1 on any unformatted file | yes, seconds, no browser | semantics, types, design intent |
| Story renders without error | Storybook (Vitest addon or test-runner "smoke test") | deterministic | GATE via the runner's own exit code (Vitest or Jest) | no, needs a real browser (Playwright) | interaction correctness, appearance |
| Interaction tests (`play` functions) | Storybook Vitest addon / test-runner | deterministic pass/fail | GATE via Vitest/Jest exit code | no, needs a real browser | anything the play function doesn't explicitly assert |
| Visual regression, cross-browser | Chromatic | deterministic pixel diff, human-gated acceptance | Both: CLI exits non-zero (`BUILD_HAS_CHANGES`) on any diff; the actual merge gate is a required PR status check that a human must clear (or `autoAcceptChanges`) | no, cloud round-trip, needs a browser render | whether an accepted diff is actually correct; that judgment is human |
| Visual regression, per-test snapshot | Playwright `toHaveScreenshot` | deterministic pixel diff (pixelmatch), threshold configurable | GATE: fails the test (non-zero exit) on any diff over threshold | no, needs a real browser | cross-machine/cross-OS render differences (documented Playwright caveat); correctness of the committed baseline itself |

## Gaps

- **No tool validates a code-level token reference against a DTCG `tokens.json` directly.** The nearest
  primitive, `csstools/stylelint-value-no-unknown-custom-properties`, needs a flat
  `{ "custom-properties": { "--name": "value" } }` JSON, which is a build artifact (for example, from
  Style Dictionary), not the raw `$value`/`$type`/alias token tree DTCG defines. Whether that build step
  keeps the flattened list honestly in sync with the token source is outside what any linter checked here
  verifies; the harness would have to guarantee that separately.
- **Judgment items have no deterministic validator, by design.** "Matches the Figma comp," "follows the
  spacing rhythm," and similar are exactly the kind of claim none of the tools above attempt: axe-core
  and jsx-a11y check rules, not intent; Chromatic and `toHaveScreenshot` catch that *something* changed
  pixel-for-pixel, not whether the change is *right*. Chromatic's "accept as baseline" step is a cheap way
  to invoke a human judgment call per diff, not a replacement for one.
- **CSS-in-JS token enforcement is unverified.** stylelint's own docs disclaim that core (and by
  extension plugin) rules are guaranteed to work through a `customSyntax` parser for styled-components or
  emotion. No primary source checked here confirms a token-literal rule actually fires correctly against
  a styled-components template literal.
- **A rule flagging a hardcoded value is not the same as a rule that fails the build.** Every AST-based
  design-token rule found (Tailwind's, MetaMask's) ships at `warn` or disabled by default in its
  documented example or recommended config. Presence of the rule in a project's dependency tree proves
  nothing about whether it gates; only the configured severity plus `--max-warnings`/CI settings do.
- **No tool found that checks which spacing/sizing scale step is "correct" for a given layout,** only
  whether a raw literal was used versus a variable/token reference at all. Enforcing *that a variable was
  used* is a materially weaker guarantee than enforcing *that the right variable was used*.

## What I did not check

- Did not run any of these tools against real code; every claim above comes from reading documentation,
  READMEs, and, for `tsc`, the compiler's own source enum, not from executing the linters/compilers and
  observing output.
- Did not verify that `AndyOGo/stylelint-declaration-strict-value` is the exact successor to whatever
  package the `YozhikM` org previously published under the same npm name; I confirmed the npm package
  name matches this repo's `package.json` description but did not trace the org-transfer history.
- Did not read `eslint-plugin-tailwindcss`'s rule implementation source, only its documentation pages, so
  the exact AST-matching logic (as opposed to the documented input/output examples) is unverified.
- Did not verify the actual runtime speed of the Vitest addon or test-runner in a real project; the
  "needs a browser, integration-test speed" characterization is inferred from the documented architecture
  (Playwright browser mode / jest-playwright), not from a timed run.
- Did not check `@metamask/eslint-plugin-design-tokens` for evidence of it running in MetaMask's actual
  CI pipeline (a required check versus advisory); only its own README and rule docs were read.
- Did not check axe-core's `doc/rule-descriptions.md` in full for a complete list of what is and is not
  covered; only the top-level README claims about coverage and the JSDOM/`color-contrast` caveat were
  read.
- Did not investigate Style Dictionary or other DTCG build tooling in depth; that is explicitly the scope
  of `docs/research/token-source-of-truth.md` (target 3 in this repo's research plan), not this file.
- Did not check for any private, enterprise, or paid design-system linting product beyond what surfaced
  in open GitHub search; the "no tool found" conclusions above are bounded by what a public GitHub/docs
  search returned on 2026-09-03.
- Did not check ESLint's or stylelint's behavior across major version boundaries; all claims are pinned
  to the specific commits/tags cited and may not hold for older or future major versions.
- Did not check whether Prettier's formatting rules can conflict with or mask a token-literal violation
  (for example, reformatting a value in a way that changes whether a regex-based rule matches it).

## Sources

| source | pinned URL | read |
|---|---|---|
| stylelint CLI docs | https://raw.githubusercontent.com/stylelint/stylelint/ac19a3c04ca20ab57d637f81b57365539c1be84b/docs/user-guide/cli.md | full |
| stylelint configure docs (severity) | https://raw.githubusercontent.com/stylelint/stylelint/ac19a3c04ca20ab57d637f81b57365539c1be84b/docs/user-guide/configure.md | part |
| stylelint options docs (customSyntax) | https://raw.githubusercontent.com/stylelint/stylelint/ac19a3c04ca20ab57d637f81b57365539c1be84b/docs/user-guide/options.md | part |
| stylelint-declaration-strict-value README | https://raw.githubusercontent.com/AndyOGo/stylelint-declaration-strict-value/ba7401895ae82c904b6e5d0418237e1fa3fc12ee/README.md | full |
| stylelint-value-no-unknown-custom-properties README | https://raw.githubusercontent.com/csstools/stylelint-value-no-unknown-custom-properties/399282b3233429c9ee090f57f204e9ce6e9cc2b2/README.md | full |
| eslint-plugin-tailwindcss README | https://raw.githubusercontent.com/francoismassart/eslint-plugin-tailwindcss/1ace2c54aa7c7d4b9590ffe02ebabdfe66bc2382/README.md | part |
| eslint-plugin-tailwindcss `no-arbitrary-value` doc | https://raw.githubusercontent.com/francoismassart/eslint-plugin-tailwindcss/1ace2c54aa7c7d4b9590ffe02ebabdfe66bc2382/docs/rules/no-arbitrary-value.md | full |
| eslint-plugin-tailwindcss `no-custom-classname` doc | https://raw.githubusercontent.com/francoismassart/eslint-plugin-tailwindcss/1ace2c54aa7c7d4b9590ffe02ebabdfe66bc2382/docs/rules/no-custom-classname.md | full |
| MetaMask eslint-plugin-design-tokens README | https://raw.githubusercontent.com/MetaMask/eslint-plugin-design-tokens/daf6e7cd9bab3e9a47551c4cbb808c4e10695354/README.md | full |
| MetaMask eslint-plugin-design-tokens `color-no-hex` doc | https://raw.githubusercontent.com/MetaMask/eslint-plugin-design-tokens/daf6e7cd9bab3e9a47551c4cbb808c4e10695354/docs/rules/color-no-hex.md | full |
| eslint-plugin-jsx-a11y README | https://raw.githubusercontent.com/jsx-eslint/eslint-plugin-jsx-a11y/8f75961d965e47afb88854d324bd32fafde7acfe/README.md | part |
| axe-core README | https://raw.githubusercontent.com/dequelabs/axe-core/540ecae993b5b0f40070c6b034843b0392956175/README.md | part |
| Storybook accessibility-testing doc | https://raw.githubusercontent.com/storybookjs/storybook/4bd806b927f82b14057e0c143b064b89663f771b/docs/writing-tests/accessibility-testing.mdx | full |
| Storybook Vitest addon doc | https://raw.githubusercontent.com/storybookjs/storybook/4bd806b927f82b14057e0c143b064b89663f771b/docs/writing-tests/integrations/vitest-addon/index.mdx | full |
| Storybook test-runner doc | https://raw.githubusercontent.com/storybookjs/storybook/4bd806b927f82b14057e0c143b064b89663f771b/docs/writing-tests/integrations/test-runner.mdx | part |
| Storybook interaction-testing doc | https://raw.githubusercontent.com/storybookjs/storybook/4bd806b927f82b14057e0c143b064b89663f771b/docs/writing-tests/interaction-testing.mdx | part |
| Storybook visual-testing doc | https://raw.githubusercontent.com/storybookjs/storybook/4bd806b927f82b14057e0c143b064b89663f771b/docs/writing-tests/visual-testing.mdx | full |
| ESLint command-line-interface doc | https://raw.githubusercontent.com/eslint/eslint/1696682791661c13167eb905da2f38d1b8f4a3bf/docs/src/use/command-line-interface.md | part |
| TypeScript `ExitStatus` enum (source, tag v7.0.2) | https://raw.githubusercontent.com/microsoft/TypeScript/1e4744d68260a7cb91b62b12edc3f6a2187faaf1/src/compiler/types.ts | part |
| TypeScript `executeCommandLine.ts` (exit call sites) | https://raw.githubusercontent.com/microsoft/TypeScript/main/tsc/testdata/fixtures/compiler/executeCommandLine.ts | part |
| Prettier CLI doc (exit codes) | https://raw.githubusercontent.com/prettier/prettier/338310e438be8bac2efcb1db4da313c046d45eef/docs/cli.md | part |
| Chromatic CLI doc (exit codes) | https://www.chromatic.com/docs/cli/ (fetched 2026-09-03) | part |
| Playwright visual comparisons doc | https://playwright.dev/docs/test-snapshots (fetched 2026-09-03) | full |
