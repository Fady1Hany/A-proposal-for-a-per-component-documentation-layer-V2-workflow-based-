# Contributing — Component Impact Discovery

## Stage

This repo is at the **proposal / RFC stage**. We are not building production tooling yet. The goal is to refine the spec, refine the workflow algorithms, gather counterexamples, and — if the ideas survive scrutiny — encourage someone to pilot the proposal on a real codebase.

This proposal supersedes one earlier form (V1) previously published by the same author. The files at the repo root supersede it. The lifecycle expands from the prior form's four workflows to six: W1 is retained; W2 is the same mutation + test-all algorithm **minus the outer loop** — it works on exactly one component you hand it, tests every other component in the system, and writes the component's MD file, **automatically replacing** any existing MD under that component's name (no mode flags); W3 is repurposed from static test enumeration into AI-driven dynamic testing; W4 is repurposed into reverse-edge sync **plus validation-hash stamping** — always a full nested loop over every `COMPONENT_<name>.md` file in the project, no diff input, and the sole writer of the `last_validated` hash that the CI staleness check (a GitHub Action comparing the hash in each MD file against the latest commit on the owning component or its `Affects` / `Affected By` components) relies on; W5 (Edit Component Loop) formalizes what the prior form left to the orchestration layer and has **no relation to Workflow 2** (doc refresh after edits is left to the staleness check); W6 (Create New Component) wraps code generation with a W2 invocation (plus the W4 that W2 triggers).

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
- **Conflating the prior form with this proposal.** The two are different proposals. If you want to discuss the prior form (V1), refer to its published description; if you want to discuss this proposal (six workflows, Affects/Affected-By only), use the root files. Don't propose changes that mix the two without being explicit about the migration path.

## How to contribute

1. Open an issue first for discussion before opening a PR.
2. PRs that change the spec MUST bump `spec_version` in [`SPEC.md`](./SPEC.md) §10 and update [`schema/COMPONENT_NAME.schema.json`](./schema/COMPONENT_NAME.schema.json) accordingly.
3. PRs that change a workflow algorithm MUST update both [`WORKFLOWS.md`](./WORKFLOWS.md) and the cost model in [`COST.md`](./COST.md) if the change affects cost.
4. Be honest about confidence levels. If you add a claim, label its source per [`SOURCES.md`](./SOURCES.md).

## Migration from the prior form

If you have a project documented in the prior form (V1) and want to migrate to this proposal:

1. Run a migration script that:
   - Drops `depends_on`, `used_by`, `change_impact`, `targeted_verification`, `validation_group` from each `COMPONENT_<name>.md`.
   - Keeps `last_validated` as-is (data-compatible — it is already `{commit, date}`).
   - Constructs `affects` from the components that appeared in the prior form's `change_impact` entries (manual prose and machine-generated entries alike).
   - Leaves `affected_by` empty — populated by the first Workflow 4 run after migration.
   - Adds an empty `provenance` field (to be populated by the first Workflow 1 or Workflow 2 run).
   - Sets `spec_version` to `v0.1`.
2. Run Workflow 1 (Full-System Scan) to fully regenerate the docs with the new semantics. The migration script's output is a valid doc, but it lacks the AI-driven testing signal — only Workflow 1 can produce that.
3. Configure the staleness checker as a GitHub Action. From this point, the change-aware freshness model is in effect, with the validation group computed implicitly from `Affects ∪ Affected By`.

## Code of conduct

Be honest about what's verified and what isn't. That's the whole code of conduct.
