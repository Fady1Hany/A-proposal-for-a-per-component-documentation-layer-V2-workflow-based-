# Contributing — Component Impact Discovery

## What we want

- **Criticism** — especially "this workflow won't work because…" with a concrete reason. The six workflows in [`WORKFLOWS.md`](./WORKFLOWS.md) make strong claims about cost, reliability, and (in the case of Workflow 3) the reliability of AI-driven dynamic test selection, (in the case of Workflow 5) the cancel-on-first-failure semantics, and (in the case of Workflow 6) the reliability of LLM code generation on the first try; they deserve scrutiny.
- **Counterexamples** — existing concepts we missed in [`COMPARISON.md`](./COMPARISON.md), especially around: (a) AI-driven dynamic test selection (does prior art exist?), (b) reverse-edge sync as a separate workflow (does prior art exist?), (c) removal of test names from component docs (has anyone tried this?).
- **Pilot results** — if you try this proposal on a real codebase, positive *or* negative. The full-system scan cost, Workflow 3's `inconclusive` rate, Workflow 5's retry-exhaustion rate, and Workflow 6's codegen-failure rate are the four metrics most in need of empirical grounding (see [`COST.md` §14`](./COST.md)). **The single most important metric to publish is Workflow 3's false-positive and false-negative rates vs. a manual ground-truth check** — that is the empirical assumption on which the proposal's value proposition turns. The second most important is **Workflow 5's `verdict=unsafe` rate in production** — if it fires often, either the AI is bad at reformulating edits, or the Affects list is too restrictive; both outcomes are diagnostically valuable.
- **Workflow 3 runner implementations** — a Claude Code command, Codex workflow, or custom Python orchestrator implementing Workflow 3 (AI-Driven Testing) at the input/output contract specified in [`WORKFLOWS.md` §3.3–3.4`](./WORKFLOWS.md) would be the most valuable contribution. Workflow 3 is the linchpin; without a working implementation, the proposal cannot be piloted.
- **Workflow 5 runner implementations** — a Claude Code command or Python orchestrator implementing Workflow 5 (Edit Component Loop) is the second most valuable contribution. W5 is what every agent edit goes through in steady state; without a working W5, the proposal's runtime guarantee ("no edit silently breaks a downstream consumer") is untested.
- **Workflow 6 runner implementations** — a Claude Code command or Python orchestrator implementing Workflow 6 (Create New Component) is the third most valuable contribution. W6 is what every new-component creation goes through; without a working W6, the proposal's new-component bootstrap is untested.
- **Better sources** — if you have a real benchmark for AI-driven test selection, mutation testing cost, or workflow-driven doc maintenance, please share it. The [`SOURCES.md`](./SOURCES.md) file is honest about what's unverified.

## What we don't want

- **Tools before the spec is stable.** Tools lock in decisions. The spec is `v0.1`; a `1.0` release would follow a successful pilot.
- **Marketing language.** This is a proposal, not a product.
- **Over-engineering.** If a field doesn't directly serve the agent's navigation or verification needs, or if a workflow step doesn't directly serve impact discovery or reverse-edge sync, it probably doesn't belong.

## How to contribute

1. Open an issue first for discussion before opening a PR.
2. PRs that change the spec MUST bump `spec_version` in [`SPEC.md`](./SPEC.md) §10 and update [`schema/COMPONENT_NAME.schema.json`](./schema/COMPONENT_NAME.schema.json) accordingly.
3. PRs that change a workflow algorithm MUST update both [`WORKFLOWS.md`](./WORKFLOWS.md) and the cost model in [`COST.md`](./COST.md) if the change affects cost.
4. Be honest about confidence levels. If you add a claim, label its source per [`SOURCES.md`](./SOURCES.md).


## Code of conduct

Be honest about what's verified and what isn't. That's the whole code of conduct.
