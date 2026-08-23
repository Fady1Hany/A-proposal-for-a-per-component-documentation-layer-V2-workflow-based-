# Tools (planned)

> Nothing in this directory is built yet. This repo is at the proposal / RFC stage; this file describes the tooling a compliant implementation would provide. See [`CONTRIBUTING.md`](../CONTRIBUTING.md) — runner implementations of Workflow 3, Workflow 5, and Workflow 6 are the most valuable contributions.

Any compliant runner (Claude Code custom commands, Codex workflows, a custom Python orchestrator driving an LLM API) that implements the algorithms in [`WORKFLOWS.md`](../WORKFLOWS.md) is acceptable. The planned tooling falls into four groups:

## 1. Workflow runners

- **W1 runner (Full-System Impact Scan)** — iterates every component in the project, mutates each one, fans out parallel Workflow 3 sub-agents, writes each component's `Affects` list, then invokes the reverse-sync tool. Long-running: must support checkpoint/resume via `progress.json` ([`WORKFLOWS.md` §1.10](../WORKFLOWS.md)).
- **W2 runner (Single-Component Impact Scan)** — works on exactly one component you hand it (no outer loop): mutates it, tests every other component in the system via Workflow 3 sub-agents, and writes the component's MD. If a `COMPONENT_<NAME>.md` already exists under that component's name, the fresh scan result **automatically replaces** it; otherwise the runner creates one. Invoked directly, by the W6 runner, and by the staleness checker on revalidation.
- **W3 runner (AI-Driven Testing)** — the linchpin. Given a `(component_under_test, mutation_context, baseline)` task, dynamically selects and executes appropriate tests and returns `(verdict, evidence)`. Input/output contract: [`WORKFLOWS.md` §3.3–3.4](../WORKFLOWS.md). Internals deliberately under-specified — that is the design.
- **W5 runner (Edit Component Loop)** — takes an edit prompt, backs up the source, applies the edit, fans out W3 sub-agents against every component under `Affects:`, cancels on the first `affected` verdict, restores and retries. **W5 has no relation to Workflow 2** — it never invokes it; verification goes through W3 only.
- **W6 runner (Create New Component)** — takes a creation prompt, generates the source, writes it, then invokes the W2 runner on the new component (W2 triggers the reverse-sync tool internally).

## 2. Reverse-sync + hash-stamping tool (Workflow 4)

Pure file I/O — zero LLM calls, zero test runs. Always a full nested loop over every `COMPONENT_<name>.md` file in the project:

1. Pick a target file, scan every other file's `Affects:` list, and rebuild the target's `Affected By:` list from scratch (no diff input, no edge-set argument).
2. **Stamp the validation hash** — write `last_validated := (commit C, current_date)` into the MD file of **every** component. Workflow 4 is the sole canonical producer of this field; no other workflow (and no human) writes it.

Cost: `O(N² × L)` file reads, seconds-to-minutes at typical project sizes. Uses skip-if-equal to leave untouched files alone, and a file-level lock to survive concurrent runs.

## 3. Staleness checker (GitHub Action)

The change-aware freshness check from [`SPEC.md` §7](../SPEC.md), packaged as a GitHub Action that runs on every push and on a schedule (default: daily). Its job:

1. **Look at the hash in each MD file** (`last_validated`, stamped by Workflow 4).
2. **Compare it against the latest commit made** on:
   - the component that owns the file (per its `location`), or
   - any component in its `Affects` list, or
   - any component in its `Affected By` list.
3. If that commit is newer than the recorded hash, emit a staleness warning naming the related component and the triggering commit. Optionally fail CI for `frozen`-stability components.
4. Optionally trigger the W2 runner for revalidation (which automatically replaces the stale MD and re-stamps hashes via Workflow 4).

No time thresholds anywhere — a quiet component with no related commits never fires, no matter how old its hash is.

## 4. Validation utilities

- **Schema validator** — parse each `COMPONENT_<name>.md` into its structured representation and validate it against [`schema/COMPONENT_NAME.schema.json`](../schema/COMPONENT_NAME.schema.json) (required fields, name/filename match, component-name-only entries in `Affects` / `Affected By`).
- **Migration script** — convert prior-form (V1) docs to this format; see the migration path in [`COMPARISON.md` §7](../COMPARISON.md).
