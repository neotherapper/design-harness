# CLAUDE.md — operating rules for design-harness

Read [`README.md`](README.md) first, then [`docs/ROADMAP.md`](docs/ROADMAP.md).

## What this repo is

The design and, later, the implementation of a harness for AI agents that create and maintain
components inside a design system. The must-have property is enforcement: an agent's component
cannot become published unless it uses the system's design tokens and passes the system's
accessibility and guideline checks.

This is an application on pavlos, the runtime, alongside zygos. It follows the same research
discipline as both. It is public.

## The five rules

1. **Read the primary source.** Never cite something you did not open. Pin every fetched file to a
   commit SHA, not a branch ref. A `main` link still resolves after its content changes.
2. **Every research file ends with "What I did not check."** A file that cannot name its own gaps is
   not finished. Say which documents were read in full, which in part, and which not at all.
3. **Record what a mechanism gates separately from what it reports.** A linter whose output nothing
   acts on is telemetry, not enforcement. Say which it is.
4. **Omit weak figures rather than sourcing them weakly.** Star counts, adoption figures, and vendor
   accuracy percentages are excluded by default. A precise number with no primary source reads as
   verification and is not.
5. **Nothing private, ever.** No client or employer design system enters this repo: not token names,
   not component rules, not screenshots, not distinctive phrasing. Examples use an open design system
   or a synthetic one. Restate findings in fresh words rather than editing originals down.

## Two layers, keep them apart

- **Inner loop, deterministic validators.** Token lint, accessibility checks, type and story checks.
  Run inside the agent's loop so a non-conforming draft is corrected fast.
- **Outer gate, guarded state machine.** A `Component` entity whose `Publish` transition requires
  validator flags that only validator credentials can set. This is what makes non-conformance
  unreachable at publish. It does not stop an agent from writing a bad draft; only the inner loop
  reduces those.

A guard checks presence and shape, never truth (zygos ADR-0001). Judgment items such as "matches the
Figma comp" or "follows the spacing rhythm" go to a review job, never to a guard. Do not write a
design that implies otherwise.

## Working style

Answer first, one decision at a time, short blocks. Put durable detail in files and give the path.
Ask the one question that blocks the next step.

## Boundary with sibling repos

pavlos and zygos are read-only from here. Harness specifications belong in zygos; runtime decisions
belong in pavlos. If a finding here changes either, open an issue there.
