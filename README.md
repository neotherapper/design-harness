# design-harness

A harness for AI agents that build and maintain reusable components inside a design system, where a
component cannot be published unless it uses the system's tokens and passes its accessibility and
guideline checks. The enforcement is structural: a non-conforming component is unreachable as a
published artifact, not merely discouraged.

This is the second application on [pavlos](https://github.com/neotherapper/pavlos), the runtime;
[zygos](https://github.com/neotherapper/zygos) is the first. Harness research records belong in zygos;
this repo holds the design-harness design, its research, and eventually its code.

Status: research. No implementation yet.

```
docs/
  ROADMAP.md        phases, what is committed and what is only direction
  research/         one file per research target, primary sources pinned to a commit
  decisions/        ADRs about this harness
  design/           architecture, the layer diagram, interface contracts
  features/         one file per feature once the design settles
```
