# Roadmap

## Phase 0 — research (committed, in progress)

Five research files under `docs/research/`, one per target, each from primary sources pinned to a
commit SHA and each ending with "What I did not check." See `docs/research/README.md` for the
targets and their status.

Exit criterion: all five files merged after review, and `docs/design/architecture.md` can be written
from them without a new fetch.

## Phase 1 — architecture (committed, after Phase 0)

One diagram and one document: the layers (supply, generate, validate, gate, publish, evolve), which
tool sits in which layer, and where Temper or a TypeScript equivalent holds state and gates
transitions. The artifact a designer and a developer can both read.

Also in Phase 1: the surface a designer and a developer actually use. Working hypothesis, to be
tested against the research: Storybook is the surface, since it already shows the component, its
stories, and the accessibility panel; the harness adds the gate status and the validator findings
beside them rather than a new UI. Whether the harness is a plugin on an existing agent harness or
a standalone pipeline is a second Phase 1 decision.

Decision to record as an ADR at the end of Phase 1: Temper as the gate substrate, or a
TypeScript reimplementation of the guarded state machine.

## Later — direction, not scope

- A pilot: one `Component` entity, one deterministic token validator, one publish gate, against an
  open design system.
- Accessibility validator and story presence as further gated flags.
- A review job for judgment items.
- Evolution of agent guidance from trajectories, if and only if the gates are stable first.
