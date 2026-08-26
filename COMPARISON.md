# Comparison to existing concepts — Component Impact Discovery

> The landscape mapping of the prior form of this proposal is preserved. A new section at the end (§7) shows how this proposal differs from its own prior form (V1).

## 1. Project-level context files

### AGENTS.md (2025)
- **What it is:** A markdown file at the repo root giving AI coding agents project-wide instructions, conventions, and behavioral rules. Vendor-neutral, supported by Codex, Cursor, Claude Code, etc.
- **Overlap:** Same broad goal — give the agent context it cannot derive from code alone.
- **Key difference:** AGENTS.md is **one file per repo**, focused on behavioral rules. This proposal is about **one file per component**, focused on behavioral impact. The two compose: AGENTS.md (or equivalent) is the orchestration layer that tells the agent how to consume `COMPONENT_<name>.md` and when to invoke each of the six workflows. The orchestration layer is intentionally project-agnostic: it does not document the project itself or encode its architecture or business logic. Instead, it defines how an AI agent identifies the appropriate `COMPONENT_<name>.md` for the requested task and the execution loop it must follow (with Workflow 5 for edits and Workflow 6 for new components). That loop is executed against the selected `COMPONENT_<name>.md`, which remains the sole source of component-specific knowledge.
- **Caveat:** The arXiv paper *"Evaluating AGENTS.md"* (Feb 2026) found that context files can *reduce* task success rates compared to no context. This is a warning about design, not a verdict on the concept — but it is the strongest cautionary data point we have. This proposal's response to that warning is to make the per-component doc as small and behaviorally-grounded as possible: no static test inventory to drift, no `depends_on`/`used_by` to confuse with behavioral impact, no hand-authored prose to forget to update.

### `.cursorrules`, `.windsurfrules`
- Vendor-specific equivalents of AGENTS.md. Same granularity gap.

## 2. Auto-generated code graphs

### Aider Repository Map (2023)
- **What it is:** Tree-sitter + PageRank produces a directed graph of symbols and call edges, compressed into the LLM context.
- **Overlap:** Provides exactly the structural information that this proposal deliberately does **not** store in the doc (since structural information is recoverable on demand from the code itself, while behavioral information is not).
- **Key difference:** Aider's map is **global and auto-generated at query time**. This proposal's `Affects` and `Affected By` are **per-component and stored on disk**, produced by workflows that intentionally mutate components and observe which other components' tests fail. The mechanism is fundamentally different: Aider reads code; this proposal runs code under mutation.
- **Integration point:** Aider (or any tree-sitter-based code-graph tool) could be the static-analysis backend for the **pre-filter** in Workflow 1 and Workflow 2 (which components to consider as candidates for testing). The behavioral discovery still requires Workflow 3's AI-driven testing.

### Cursor semantic indexing + `@`-references
- Embeddings plus symbol-level references. Same mechanism as Aider, different UX. Same gap: structural only, not behavioral.

### CodeGraph / Depwire / CodeKnit / Pharaoh / Riftmap (MCP servers, 2025–2026)
- MCP servers that parse the codebase with tree-sitter, build a dependency graph, and expose blast-radius, dependency tracing, function search as tools the agent can call.
- **Overlap:** These give the agent the *content* of structural fields on demand — which is exactly what this proposal refuses to store in the doc.
- **Key difference:** They are **runtime tools**, not documentation. They cost context budget on every query. They also cannot, by construction, answer the behavioral question — they read code structure, not test behavior under mutation.
- **Integration point:** Same as Aider. An MCP code-graph server can be the pre-filter that Workflow 1/2 uses to narrow candidate components before dispatching Workflow 3.

## 3. Change impact analysis (research field)

### Software Change Impact Analysis (Lehnert et al., cited 243+)
- Mature research area. Dependency-graph rewriting, ripple-effect analysis, snapshot evolution.
- **Overlap:** Provides the theoretical basis for the mutation-driven impact discovery that Workflow 1 and Workflow 2 perform.
- **Key difference:** Academic techniques, not a documentation format. This proposal borrows the *content* (mutation testing is a well-established impact-analysis technique) and packages it as a per-component workflow-driven field. The novel move is using mutation testing as the **primary producer of an AI-consumed documentation layer**, not just as a research methodology or a code-coverage tool.

### Coderabbit "Security Blast Radius"
- PR-level tool that extends a diff to its likely affected surface.
- **Overlap:** Same idea, narrower scope (PRs, not components) and narrower consumer (security review, not general agent navigation).
- **Integration point:** Coderabbit flags a PR's blast radius at review time; this proposal's staleness check triggers Workflow 2 for components in the affected surface. The two compose naturally — Coderabbit is the per-PR trigger, Workflow 2 is the per-component response.

## 4. AI-oriented architecture concepts

### AI-Readable Architecture (Shumilov, *Contemporary*, May 2026)
- **The closest conceptual match.** Proposes a new abstraction layer where structural constraints of a software system are made machine-readable for AI-augmented development.
- **Overlap:** Same core argument — documentation should be reorganized around the AI consumer's constraints.
- **Key difference:** Shumilov frames it at the architecture level. This proposal narrows it to a **per-component schema with two specific behavioral fields** (`Affects` and `Affected By`), and adds a six-workflow validation model that produces, maintains, and consumes those fields through actual testing and runtime edit-verification.
- **Novelty over Shumilov:** The six-workflow lifecycle, the AI-driven dynamic test selection in Workflow 3, the runtime edit-verify loop in Workflow 5, the one-shot new-component bootstrap in Workflow 6, and the bidirectional `Affects`/`Affected By` representation. Shumilov's contribution is the architectural argument; this proposal's contribution is the per-component workflow mechanism that makes the argument operational.

### AI-Centric Software Architecture (academia.edu, 2026)
- Argues architecture should be designed for AI consumption, not just human consumption. Conceptual, not a format.

### Design-system-as-MCP (Design Systems Collective, 2026)
- Converts design-system components (props, tokens, usage patterns) into machine-readable metadata that AI tools can query.
- **Overlap:** A *domain-specific* version of this idea, applied to UI components. Worth studying as a precedent for what a generator looks like in practice.
- **Integration point:** A design-system-specific variant of Workflow 3 — where the AI tests "change this prop's type" or "remove this token" — would be a natural specialization.

## 5. Traditional structural documentation (precursors)

### Architectural Decision Records (ADRs)
- Per-decision records capturing context + consequences.
- **Difference:** Per-decision, not per-component. Missing the explicit behavioral-impact fields.

### C4 model, UML, dependency graphs, call graphs
- Visualization and analysis artifacts.
- **Difference:** Human-oriented, not agent-oriented. Not co-located with components. Describe structure, not behavior.

### Javadoc / Doxygen / Sourcery
- Extract structure from code, produce reference material.
- **Difference:** Reference material for humans, not navigation packets for agents. Describe *what* a component does, not *what it affects*.

## 6. Mutation testing as a field

### Mutation testing for test-suite quality (Stryker, Mutmut)
- **What they do:** Mutate the code under test, run the test suite, observe which mutations the tests catch. Produce a "mutation score" — the fraction of mutations the test suite detected. Used to assess test-suite quality.
- **Overlap with this proposal:** Same mechanism (intentional mutation, observe test behavior).
- **Key difference:** Different purpose. Stryker and Mutmut assess *the test suite*. This proposal uses mutation to discover *behavioral relationships* (Affects / Affected By). The two are conceptually similar but produce different artifacts: Stryker produces a coverage report for humans; this proposal produces `Affects`/`Affected By` lists for AI agents.
- **Honest caveat:** This proposal's application of mutation testing — to produce AI-consumed documentation rather than to assess test-suite quality — appears novel. We have not found prior art for this specific use, though the methodology is well-established.

## 7. This proposal vs its own prior form (V1)

This proposal supersedes one earlier form previously published by the same author. The comparison below is direct:

| Aspect | Prior form (V1) | This proposal |
|---|---|---|
| **Field ownership** | Split: auto-generated structural fields (`location`, `depends_on`, `used_by`) vs hand-authored intent fields (`change_impact`, `targeted_verification`, `notes`), with `targeted_verification` sourced from a static test inventory | All workflow-produced; humans review and edit after. The `Affected By` field is exclusively produced by Workflow 4. |
| **Primary relationship fields** | `depends_on` (imports) + `used_by` (importers) + `change_impact` (hand-authored prose, later augmented with mutation-derived entries) | `Affects` (mutation-derived) + `Affected By` (reverse-synced by Workflow 4). `depends_on`/`used_by`/`change_impact` are removed. |
| **Test information in doc** | `targeted_verification` — a list of test names taken from a static test inventory . and this List contains the specific tests or checks to run after modifying this component. This includes tests for components that may be affected by changes to this component, whether or not they are listed in depends_on . Prefer exact test names. | **None.** Workflow 3 dynamically selects tests at test time. No test names stored in the doc. The workflow (W3) can also be implemented by running static tests for testing the component_under_test. These tests are documented in Component_component_under_test.md under the `notes` field ( section ); however, this is an exceptional implementation.and if you will use the `notes` field to write static tests you can use it for that **under one condition** : any test in any `COMPONENT_<name>.md` must cover ( test ) **only** the Component whose name  matches the `<name>` part of that `COMPONENT_<name>.md` filename and must not cover any components related to it|
| **update md files** | 90 days from `last_validated` date  | `last_validated` validation hash **stamped exclusively by Workflow 4**, compared against the latest commit touching the owning component or any component in `Affects ∪ Affected By` (change-aware, validation group implicit, commit-hash based via a GitHub Action) |
| **Bootstrap** | create MD file for each component in your project| Workflow 1 (Full-System Scan): nested-loop mutation testing across all N components, delegates testing of candidate-affected components to Workflow 3 via parallel sub-agents |
| **Edit-and-verify loop** | Not specified — left to the orchestration layer (AGENTS.md) | **Workflow 5 (Edit Component Loop)** — take edit prompt, back up source, apply edit, fan out W3 sub-agents against every component under `Affects:`, cancel-on-first-failure, restore-and-retry until safe or max-retries hit. **Workflow 5 has no relation to Workflow 2** — it never invokes it, and W5 can invoke W4. |
| **New-component bootstrap** | create the new component by ai and then create the .md file for that component| **Workflow 6 (Create New Component)** — take creation prompt, generate source code, write to disk, invoke Workflow 2 on the new component — no MD exists under its name yet, so W2 creates one (it would automatically replace it if one existed). W2 internally triggers W4. One-shot new-component creation + documentation. |
| **Reverse sync** | Not specified — `used_by` was computed by inverting `depends_on` at workflow time, including the mutation-derived edges | **Workflow 4 (Reverse Relationship Sync)** — explicitly specified algorithm that is **always a full nested loop over every `COMPONENT_<name>.md` file in the project**: it picks a file, scans every other file's `Affects:` list, and rebuilds that file's `Affected By:` list from scratch. No diff input, no edge-set argument — the algorithm is the same regardless of which workflow invoked it. This eliminates drift in unrelated files and simplifies the caller contract (every caller just invokes W4 with the project root). Workflow 4 is also **the workflow responsible for stamping the validation hash (`last_validated`) into the MD file of every component** — it is the sole writer of that field, which is exactly what the CI staleness check compares against git history. Pure file I/O, `O(N² × L)` reads, zero LLM calls, zero tokens. |
| **Parallelism** | Not specified  | Explicit sub-agent fan-out model in Workflows 1, 2, and 5; testing tasks dispatched to parallel sub-agents that each invoke Workflow 3 |
| **Schema fields** | `component`, `location`, `last_validated`, `depends_on`, `used_by`, `change_impact`, `targeted_verification`, `validation_group`, `provenance`, plus optional `public_interface`, `stability`, `notes` | `component`, `location`, `last_validated`, `affects`, `affected_by`, `provenance`, plus optional `public_interface`, `stability`, `notes`, `spec_version`. Removed: `depends_on`, `used_by`, `change_impact`, `targeted_verification`, `validation_group`. |


### Migration path from the prior form (V1)

A migration script can convert prior-form docs to this proposal's format:

1. Drop `depends_on`, `used_by`, `change_impact`, `targeted_verification`, `validation_group` from each `COMPONENT_<name>.md`.
2. Construct `affects` from the union of components that appeared in the prior form's `change_impact` entries.
3. Leave `affected_by` empty — populated by the first Workflow 4 run after migration.
4. Keep `last_validated`, `provenance`, `public_interface`, `stability`, `notes` as-is (the `provenance.workflow_id` enum now includes 4, 5, and 6 for the new workflows' runs).
5. Set `spec_version` to `v0.1`.
6. Run Workflow 1 (Full-System Scan) to fully regenerate the docs with the new semantics. The migration script's output is a valid doc, but it lacks the AI-driven testing signal — only Workflow 1 can produce that.

## What is genuinely novel in this proposal

The prior form claimed several novel things — the workflow lifecycle, mutation-driven impact discovery, the change-aware freshness model. This proposal claims four (overlapping) novel things:

1. **The six-workflow lifecycle, with Workflow 2 as the outer-loop-free single-component scanner (one algorithm, no mode flags, automatic MD replacement when a doc already exists under the component's name), Workflow 3 as an AI-driven testing engine, Workflow 4 as the always-nested-loop reverse-edge sync **and the sole stamper of the validation hash**, Workflow 5 as the runtime edit-verify loop (with no relation to Workflow 2), and Workflow 6 as the new-component bootstrap (which uses Workflow 2).** The prior form's workflow structure had testing as a static-enumeration step. This proposal makes testing dynamic and AI-selected at test time. Workflow 2 always runs the full mutate + test-all algorithm on the single component it is handed — the difference from Workflow 1 is exactly the missing outer loop — and an existing MD is automatically replaced by the fresh scan result. The W4 redesign (always a full nested loop, no diff input, plus hash stamping) simplifies the caller contract, eliminates drift in unrelated files, and gives the freshness model a single authoritative hash writer. The specific composition — Workflow 1 (outer loop over all components) and Workflow 2 (no outer loop, one component at a time) as impact discovery at different scopes, Workflow 3 as AI-driven dynamic testing, Workflow 4 as a single-algorithm reverse-edge sync + validation-hash stamper, Workflow 5 as the runtime edit-and-verify loop (which never invokes W2), and Workflow 6 as new-component creation + W2 scan + reverse-sync in one call — is, as far as we can tell, new.
2. **Mutation testing as the impact-discovery mechanism for AI docs.** Mutation testing is well-established in research (Lehnert, Jia/Harman) but has not, to our knowledge, been packaged as the *primary producer* of an AI-consumed documentation layer. This proposal retains the prior form's claim here.
3. **Change-aware freshness via the implicit `Affects ∪ Affected By` validation group.** The prior form used a separate `validation_group` field. This proposal computes the validation group implicitly from the two behavioral fields — the structural simplification is small but the conceptual cleaning is meaningful.
4. **Removal of test names from the component doc.** This is the most novel and riskiest design choice. To our knowledge, no prior AI-oriented component documentation format has explicitly refused to store test names. The argument is that test selection is a dynamic question best answered by the AI at test time, not a static question best answered by a curated list. The trade-off (assumes AI can reliably select appropriate tests) is explicit.

## See also

- [`SOURCES.md`](./SOURCES.md) for the honest audit of which of these claims are verified vs. vendor-cited vs. reasoned.
- [`RATIONALE.md`](./RATIONALE.md) §"Part 4 — Why test names do not belong in component docs" for the argument against the static test list.
