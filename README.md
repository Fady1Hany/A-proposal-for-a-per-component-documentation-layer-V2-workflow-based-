# Component Impact Discovery

> A proposal for a per-component documentation layer in which the **behavioral relationships between components are discovered through controlled change and dynamic AI-driven testing**, rather than hand-authored or statically inferred from imports.
>
> The documentation layer is co-located with each component as a small structured Markdown file (`COMPONENT_<name>.md`). What makes this proposal different from prior per-component documentation formats is that the doc does **not** store test names, dependency lists, or human-authored intent. It stores only the **Affects** and **Affected By** relationships — and those relationships are produced by running the system's six workflows, not written by a developer.
>
> AI was used as a research and writing assistant in developing this proposal. The underlying idea, core concepts, and direction are the author's own.

## The core idea in one paragraph

Most component documentation tries to describe what a component is, what it imports, and which tests cover it. That information goes stale the moment a developer changes a related file without remembering to update the doc. This proposal inverts the contract. The doc stores only **behavioral relationships discovered through actual testing**: when component X is changed, which other components' tests break? That question is empirically answerable. A workflow intentionally mutates X, hands the suspected-affected components to an AI-driven testing workflow, observes which tests actually fail, and writes the resulting `Affects` list back into the component's Markdown. A second workflow keeps the reverse direction (`Affected By`) in sync. The result is a documentation layer that reflects observed behavior, not assumed structure, and that stays trustworthy because the workflows — not humans — keep it current.

## The problem

An AI coding agent does not have a persistent mental model of a codebase. Every interaction starts from near-zero context: it must rediscover where things are, what calls what, what breaks if it changes a given file, and which tests to run afterward. As a codebase grows, this rediscovery becomes expensive — in context budget, in time, and in reliability. A human developer who wrote the code carries most of this implicitly; the AI does not.

Existing approaches to closing this gap have a shared weakness. Project-level context files (AGENTS.md, `.cursorrules`) give the agent behavioral rules but no per-component map. Auto-generated code graphs (Aider repo map, Cursor semantic indexing, CodeGraph MCP servers) extract static structure at query time, but they describe **imports and call edges**, not behavioral dependencies — a component may import another and never break when it changes, or fail catastrophically when an unrelated-looking component changes. Hand-authored component docs go stale the moment a developer forgets to update them. None of these approaches captures the question the agent actually needs answered: *if I change this component, which other components will break, and how do I verify it?*

This proposal answers that question directly, by running the question rather than guessing at it.

## The proposal, in three sentences

For every component in a codebase, co-locate a small structured Markdown file — `COMPONENT_<name>.md` — that tells an AI agent exactly two things about that component: **what it affects**, and **what it is affected by**. Those two lists are produced by workflows that intentionally modify the component and observe, through actual AI-driven testing, which other components' behavior changes. The agent consumes the doc to decide what to test, and the workflows keep the doc current as the codebase evolves.

The doc does **not** contain a hand-authored dependency list, a curated list of test names, or prose about change impact. It contains component names only — the entries that testing proved to be related. If a relationship disappears, the next workflow run removes it. If a new relationship appears, the next workflow run adds it.

## The six workflows

The system is built around six workflows. Two of them drive impact discovery; one performs the actual testing; one keeps the documentation consistent in both directions and stamps the validation hash; one is the runtime edit-and-verify loop that consumes the relationships stored in the COMPONENT_*.md files during agent operation; one wraps code generation with the discovery + sync workflows for new components.

| # | Workflow | Purpose | When it runs | Cost shape |
|---|---|---|---|---|
| 1 | **Full-System Impact Scan** | Iterate every component in the project, mutate it, delegate testing of suspected-affected components to Workflow 3, and write the resulting `Affects` list back into each component's doc | Once, at initial setup; on demand for full re-scan | High — `O(N × M × C × T / P)` with parallel sub-agents reducing wall time |
| 2 | **Single-Component Impact Scan** | The same mutation + test-all operation as Workflow 1, minus the outer loop: it works on exactly one component you hand it. It mutates that component, tests **every** other component in the system, determines who the component affects and who it is affected by, and writes the component's MD file. If a `COMPONENT_<NAME>.md` already exists under that name, the fresh scan result **automatically replaces** it (previous version preserved in git history); otherwise W2 creates one. Then invokes W4. | Revalidation / on demand; new component via W6 | `O(M × C × T / P)` |
| 3 | **AI-Driven Testing** | Given a (component-under-test, mutation-context) task, dynamically determine and execute the appropriate tests for that component; return pass/fail verdicts and impact evidence | Invoked by Workflows 1, 2, and 5, in parallel by sub-agents | Proportional to the component's surface and the AI's selected test count |
| 4 | **Reverse Relationship Sync** | **Always a full nested loop over every `COMPONENT_<name>.md` file in the project**: pick a file, scan every other file's `Affects:` list, and rebuild that file's `Affected By:` list from scratch. No diff input, no edge-set argument — the algorithm is the same regardless of which workflow invoked it. W4 is also responsible for **stamping the validation hash into the MD file of every component** — it is the sole writer of `last_validated`. | After Workflows 1, 2, , 6 ( and also can 5 ) complete (always complete re-derivation + hash stamping) | `O(N² × L)` file reads where N = number of MD files and L = average `Affects` list length; pure file I/O, **zero LLM calls, zero test runs** |
| 5 | **Edit Component Loop** | Take an edit prompt for component X, back up X's source, apply the edit, fan out parallel sub-agents to Workflow 3 against every component under X's `Affects:` headline; if any reports `affected`, cancel the rest, restore the original, retry with a different edit; loop until safe or max-retries hit. Workflow 5 has **no relation to Workflow 2** — it never invokes it; verification goes through Workflow 3 only. | Whenever the agent modifies an existing component; on demand for edit verification | `O(R × \|Affects(X)\| × T / P)` where R = retries, P = sub-agent fan-out |
| 6 | **Create New Component** | Take a creation prompt, generate the new component's source code, write it to disk, then invoke Workflow 2 on the new component — no MD exists under its name yet, so W2 creates one (mutate the new component, test every other component in the system, write the MD). W2 internally triggers Workflow 4 to propagate reverse edges and stamp the validation hashes. | Whenever a new component is created; on demand for end-to-end new-component bootstrap | `O(codegen) + O(M × C × T / P)` (the Workflow 2 cost) |

Full algorithmic detail, the input/output contract for Workflow 3, the edit-loop contract for Workflow 5, edge cases, and the parallel sub-agent fan-out model are in [`WORKFLOWS.md`](./WORKFLOWS.md).

### Why six and not fewer

The split exists because the six operations have different triggers, different cost profiles, and different trust models. Workflow 1 and Workflow 2 run the same mutation + test-all algorithm — the only difference is that Workflow 1 wraps it in an outer loop over every component in the project, while Workflow 2 has no outer loop and works on the single component you hand it. Splitting them lets bootstrap be a deliberate, planned operation while single-component maintenance stays one invocation wide. Workflow 3 is split from 1 and 2 because the question *"which components should be tested?"* is logically distinct from *"how should each component be tested?"* — letting the AI decide the latter dynamically means the documentation never has to track a static test inventory that drifts every time a developer adds a test. Workflow 4 is split because reverse-edge propagation and validation-hash stamping are pure bookkeeping operations that should not block the more expensive forward-edge discovery.

Workflow 5 is split because it is a *runtime consumer* of the relationships stored in the COMPONENT_*.md files, not a *producer*. Workflows 1, 2, and 4 write to the docs; Workflow 5 reads from them to guarantee that an agent's edit doesn't silently break a downstream consumer. Folding W5 into the orchestration layer (as the prior form did) leaves the edit-verify loop unspecified — every runner implements it differently, the retry semantics drift, and there is no contract for what happens when a sub-agent reports `affected`. Promoting it to a numbered workflow fixes the contract: read the `Affects` list, back up the source, edit, fan out parallel W3, cancel-on-failure, restore, retry. Workflow 5 has no relation to Workflow 2. Workflow 6 is split because creating a new component is a non-trivial three-step operation (generate code → impact-scan → reverse-sync) that deserves its own named contract; without W6, the new-component lifecycle is implicit in W2 and the agent has no formal way to request "write this new component and document it" in one call.

A two-workflow design (one for "discover" and one for "sync") would either collapse the AI testing layer into the discovery workflow — making it impossible to parallelize testing across sub-agents — or omit the reverse-edge sync, leaving the relationships stored in the COMPONENT_*.md files readable in only one direction. A four-workflow design (no W5/W6) leaves the edit-verify loop and the new-component bootstrap unspecified. Both trade-offs are worse than the six-workflow split.

## The component documentation file

Each `COMPONENT_<name>.md` is small. It contains, at minimum:

1. **`component`** — the component's name, matching the filename.
2. **`location`** — where the component lives (file path or paths).
3. **`last_validated`** — the validation hash: the git commit and date stamped into the doc by Workflow 4. W4 is the sole writer of this field (see the freshness model below).
4. **`Affects`** — the list of components whose behavior was observed to change when this component was modified. Component names only.
5. **`Affected By`** — the list of components whose modification was observed to affect this component. Component names only. Auto-synced by Workflow 4.
6. **`provenance`** — workflow metadata for debugging and cost tracking.

What is **not** in the file: a list of test names, a manually authored `depends_on` / `used_by` list, prose descriptions of change impact, or a curated `targeted_verification` list. The system does not need these because the question "which tests should I run?" is answered dynamically by Workflow 3 at the time the agent actually modifies the component, not stored in advance as a stale snapshot.

Manual tests are not deleted by this system. Existing hand-authored tests remain part of the project and remain valuable as records of previously discovered bugs, business rules, and regression cases. The system simply does not store their names inside the component Markdown files. The AI in Workflow 3 is free to discover and execute them as part of its dynamic test selection.

## The full lifecycle

The system covers the full documentation lifecycle, end to end. None of these stages are optional — skipping any one breaks the trust model.

```mermaid
flowchart LR
    A[Component Creation Prompt] --> W6[W6: Create New Component<br/>invokes W2]
    W6 --> B[W2: Single-Component Scan<br/>mutate + test all other components]
    B --> C[W3: AI-Driven Testing]
    C --> D[Affects Discovered]
    D --> E[W4: Reverse Sync + hash stamping<br/>always full nested loop over all MD files]
    E --> F[Steady State]
    F -.agent edits X.-> W5[W5: Edit Component Loop<br/>verify via W3 and W4 ( optional )]
    W5 -.re-verifies against.-> C
    W5 -.safe edit applied.-> F
    F -.related change.-> B2[W2: Single-Component Scan<br/>mutate + test all, replace MD]
    B2 --> E
```

1. **Component Creation Prompt** — A new component is requested via Workflow 6.
2. **Create New Component** — Workflow 6 generates the source code, writes it, then invokes Workflow 2 — the new component has no MD under its name yet, so W2 creates one.
3. **Single-Component Scan** — Workflow 2 mutates the new component, dispatches testing of every other component in the system to Workflow 3, and writes the resulting `Affects` list into the new component's doc.
4. **AI-Driven Testing** — Workflow 3 dynamically determines and executes appropriate tests for each suspected-affected component. The verdicts (which components were actually affected) become the raw material for the `Affects` list.
5. **Affects Discovered** — The forward edges (X affects A, X affects C, …) are written into the new component's doc.
6. **Reverse Sync** — Workflow 4 runs its always-on nested loop over every MD file in the project: it reads each file, scans every other file's `Affects:` list, rebuilds that file's `Affected By:` list from scratch, and stamps the validation hash into the MD file of every component. A's doc gets `Affected By: X`; C's doc gets `Affected By: X`.
7. **Steady State** — The component is fully documented. When the agent later needs to edit X, Workflow 5 takes over: it reads the `Affects` list from X's MD, backs up the source, applies the edit, fans out W3 sub-agents against every component in `Affects`, restores on failure, retries with a different edit, and only commits the edit when all sub-agents report `not_affected`. Workflow 5 has **no relation to Workflow 2** — it verifies through Workflow 3 only. When any component in the impact group is later modified by a related change, the staleness check fires and re-triggers Workflow 2 with the path to the component's MD (mutate that component, test every other component, automatically replace its MD).

The same lifecycle applies when Workflow 1 is run against the entire project at initial setup; the only difference is that the outer loop iterates every component rather than one, and W1 invokes W4 (the full nested loop + hash stamping) once at the end.

## The change-aware freshness model

A time-based freshness threshold ("older than 90 days = outdated") is rejected. A component that hasn't been touched in 91 days but whose related components also haven't been touched is perfectly fine. A component whose doc was last validated yesterday but whose related component just had a breaking PR merged today is dangerously stale. Wall-clock time is the wrong axis.

The freshness model is change-aware:

1. **`last_validated`** in each `COMPONENT_<name>.md` is the validation hash: the git commit at which the doc's relationships were last confirmed. **Workflow 4 is the workflow responsible for putting this hash into the MD file of every component** — it stamps the hash during its reverse-sync pass, and no other workflow writes it.
2. **Validation group** — for any component X, the validation group is X itself plus `Affects(X) ∪ Affected_By(X)`. These are exactly the components whose behavior is coupled to X's; no separate field is needed.
3. **GitHub Action hash check** — a GitHub Action looks at the hash in each MD file and compares it against the latest commit made on the component that owns the file, or on any component in its `Affects` list, or on any component in its `Affected By` list.
4. **Comparison** — if the latest commit on any of those components is newer than the hash recorded in the MD file, the doc is flagged as inconsistent.
5. **Warning** — CI emits a change-aware staleness warning naming the related component and the commit that triggered the staleness. The warning can be configured to fail CI for `frozen`-stability components where staleness is unacceptable.

The hash in each `.md` is therefore **evidence of the last validation** (stamped by Workflow 4), while git history and the `Affects`/`Affected By` lists determine whether that validation is still trustworthy. The system is change-aware, not time-aware. Full details in [`SPEC.md` §7](./SPEC.md).

## What's in this repo

| File | Purpose |
|---|---|
| [`README.md`](./README.md) | This file — overview, six workflows at a glance, lifecycle, freshness model |
| [`SPEC.md`](./SPEC.md) | Formal spec for the `COMPONENT_<name>.md` format, the required and optional fields, validation model, and lifecycle rules |
| [`WORKFLOWS.md`](./WORKFLOWS.md) | Algorithmic specification of the six workflows — inputs, outputs, edge cases, parallel sub-agent fan-out, edit-loop contract (W5), failure modes |
| [`RATIONALE.md`](./RATIONALE.md) | Why this exists — context-window asymmetry, why observed-impact beats static-structure, why test names don't belong in component docs |
| [`COMPARISON.md`](./COMPARISON.md) | Landscape vs AGENTS.md, Aider repo map, MCP code-graph servers, ADRs, Shumilov, and the prior form (V1) of this proposal |
| [`COST.md`](./COST.md) | Order-of-magnitude cost model for the full-system scan vs the single-component scan, with hypotheses to measure on a pilot |
| [`SOURCES.md`](./SOURCES.md) | Honest audit of the evidence base — what's verified, what's reasoned, what's vendor-cited |
| [`examples/COMPONENT_authentication_service.md`](./examples/COMPONENT_authentication_service.md) | Worked example in the new format |
| [`examples/COMPONENT_TEMPLATE.md`](./examples/COMPONENT_TEMPLATE.md) | Blank template to copy |
| [`schema/COMPONENT_NAME.schema.json`](./schema/COMPONENT_NAME.schema.json) | JSON Schema for the structured representation of the format |
| [`tools/Readme.md`](./tools/Readme.md) | Planned tooling — workflow runners, reverse-sync tool, staleness checker |
| [`CONTRIBUTING.md`](./CONTRIBUTING.md) | How to contribute / discuss |

## Relationship to the prior V1 proposal

This proposal supersedes one earlier form of the same idea, previously published by the same author. The files at the repo root are the canonical statement of the architecture going forward.

The prior form (V1) split every `COMPONENT_<name>.md` into auto-generated structural fields (`location`, `depends_on`, `used_by`) and hand-authored intent fields (`change_impact`, `targeted_verification`, `notes`), kept a static test inventory as the source of test names, maintained an explicit `validation_group` field under a change-aware freshness model, and defined a four-workflow lifecycle built around mutation testing with statically enumerated tests. This proposal removes the static test inventory (replaced by AI-driven dynamic test selection in Workflow 3) and the `depends_on` / `used_by` / `change_impact` / `targeted_verification` / `validation_group` fields (replaced by `Affects` and `Affected By`, which store component names only, with the validation group derived implicitly). The change-aware freshness model is retained, now built on the validation hash that Workflow 4 stamps into every component's MD and that a GitHub Action compares against the latest commit on the owning component or its `Affects`/`Affected By` components. The lifecycle is expanded from four workflows to six: Workflow 1 and Workflow 2 run the same mutation + test-all algorithm (W2 has no outer loop — it works on the single component you hand it, and automatically replaces the component's MD if one already exists under its name), Workflow 3 becomes the AI testing engine, Workflow 4 becomes the reverse-edge sync and the sole stamper of the validation hash (always a full nested loop over every MD file in the project, no diff input), Workflow 5 is the runtime edit-and-verify loop (formalizing what the prior form left to the orchestration layer; it has no relation to Workflow 2), and Workflow 6 wraps code generation with a W2 invocation plus W4 for new-component bootstrap.

## Status

**Stage: proposal / RFC.** Not a tool. Not a standard. Not validated on a real codebase yet.

This is a substantive revision, not a refinement of the prior form. The defining design choices of the prior form — a static test-inventory workflow, curated `targeted_verification` lists, the `depends_on`/`used_by` structural fields — are removed. In their place is a smaller doc format (component names only) and an AI-driven testing workflow that decides dynamically what to test. The trade-off is explicit: the system makes stronger assumptions about the AI's ability to reliably select and execute appropriate tests for a component without a pre-enumerated inventory, and it assumes an AI orchestration layer (Claude Code, Codex, similar) that can dispatch parallel sub-agents and reconcile their results.

The intent of publishing is to invite discussion specifically on the six-workflow structure, the AI-driven testing contract in Workflow 3, the edit-and-verify contract in Workflow 5, and the implications of removing test names from the component doc. If the ideas survive scrutiny, the next step is a pilot: a real 200+ component codebase, instrumented before/after, with full-system scan cost measured directly. See [`COST.md` §"Recommended pilot"](./COST.md) for the measurement plan.

## License

MIT. See [`LICENSE`](./LICENSE).

## Contributing

Open an issue with criticism, counterexamples, or pilot results. Pull requests that improve the spec, refine the workflow algorithms, or add runner implementations are welcome. See [`CONTRIBUTING.md`](./CONTRIBUTING.md).
