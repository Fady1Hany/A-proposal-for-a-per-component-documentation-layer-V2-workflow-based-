# Sources — honest audit (Component Impact Discovery)

> This document exists because several claims in this repo are weaker than they first appeared. The audit from the prior form (V1) is preserved; this proposal's specific sources and claims are added in §4 onward. If you find a claim here that's overstated, please open an issue.

## 1. The four base sources (carried over from the prior form)

### 1.1 The arXiv paper on AGENTS.md
- **Title:** *"Evaluating AGENTS.md: Are Repository-Level Context Files Actually Useful?"*
- **Date:** February 2026.
- **Source:** arXiv:2602.11988v1
- **What it actually says (per abstract):** Across multiple coding agents and LLMs, repository-level context files tend to reduce task success rates compared to providing no repository context at all.
- **How the prior form used it:** As a cautionary data point that *poorly designed* agent context can hurt.
- **How this proposal uses it:** Same caution, applied even more aggressively. The prior form's argument was that its auto/authored field split was a "design compromise" the AGENTS.md paper warned about. This proposal removes that split's remaining residue and a second compromise the prior form retained — the static test inventory — and argues that the per-component doc should store only behaviorally-discovered relationships, not static structure or test names. The paper still doesn't directly test per-component docs or workflow-driven docs, so the caution applies indirectly.
- **Honest caveat:** The paper is about **project-level context files**. It is *not* a direct test of per-component structural documentation, mutation-driven docs, AI-driven test selection, or workflow-driven maintenance. The analogy is indirect.

### 1.2 The "30% increase in change-failure rates" claim
- **Source cited:** Reddit post linking to riftmap.dev blog.
- **What the blog says:** Cites "2026 data — Cortex, DORA, Amazon's memo" to claim AI coding agents have pushed change-failure rates up ~30% on cross-file changes.
- **Honest problem:** This is a **vendor blog** (riftmap sells blast-radius tooling) citing secondary sources we have not read. The specific "30%" figure is vendor-cited and should be treated as directional, not quantified.
- **Defensible version:** "AI coding agents have measurably higher failure rates on cross-file changes than on localized changes" — that part is plausible and consistent with how the tools behave.
- **Status:** **Low confidence.** Used as a directional indicator of problem size, not as a precise number. Unchanged from the prior form.

### 1.3 Cursor / Aider public claims about codebase indexing
- **Sources:** Aider documentation, Cursor marketing, Codegen.com.
- **Honest problem:** These are **descriptions of mechanisms** and **marketing claims**, not benchmarked outcome studies.
- **Status:** **Weak.** Unchanged from the prior form.
- **This proposal's relevance:** The proposal explicitly does not store structural information in the doc (because it's recoverable on demand from tools like these). The proposal therefore doesn't rely on these tools' efficacy claims, only on the existence of their mechanism.

### 1.4 First-principles reasoning
- **The argument:** arithmetic on context budget, file-read cost, and navigation savings.
- **Status:** **High confidence in the upper bound.** Unchanged from the prior form.
- **This proposal's relevance:** The cost model in [`COST.md`](./COST.md) extends this with concrete token/wall-time estimates for the full-system scan and single-component scan. The arithmetic is still arithmetic; the per-iteration token estimate is medium confidence.

## 2. Verified-to-exist concepts (carried over)

| Concept | Source | Status |
|---|---|---|
| AGENTS.md convention | agents.md, GitHub `agentsmd/agents.md` | Verified — open standard |
| Aider repo map | aider.chat/docs/repomap.html | Verified — well-documented mechanism |
| AI-Readable Architecture (Shumilov) | *Contemporary*, May 2026; ResearchGate 405269777 | Verified — peer-reviewed concept, closest match |
| CodeGraph MCP server | github.com/colbymchenry/codegraph | Verified — exists |
| Depwire MCP server | news.ycombinator.com/item?id=47169193 | Verified — exists |
| Software Change Impact Analysis (Lehnert) | d-nb.info review, cited 243+ | Verified — established research field |
| Architectural Decision Records | adr.github.io | Verified — established practice |
| Coderabbit Security Blast Radius | docs.coderabbit.ai | Verified — exists, narrower scope |

## 3. Mutation testing as a field (carried over)

| Concept | Source | Status |
|---|---|---|
| Mutation testing as a methodology | Jia & Harman, *"An Analysis and Survey of the Development of Mutation Testing"* (IEEE TSE, 2011, cited 2000+) | Verified — established research field |
| Mutation testing for code coverage | Stryker (stryker-mutator.io), Mutmut (github.com/boxed/mutmut) | Verified — production tools exist |
| Mutation testing for AI-assisted impact analysis | (no specific source found) | **Not verified** — application of mutation testing to produce AI-consumed documentation appears novel; we have not found prior art |
| Cost of mutation testing on large codebases | Andrews et al., *"Is Mutation an Appropriate Tool for Testing Experiments?"* (ISSRE 2005) — argues mutation testing is feasible with subset selection | Verified — the cost concern is real and studied; subset selection (e.g. M ≈ 5 per component) is a recognized mitigation |

**Honest caveat:** The prior form used mutation testing as the primary impact-discovery mechanism. This proposal retains that choice. The application of mutation testing to produce AI-consumed documentation remains, to our knowledge, novel but unvalidated.

## 4. This proposal's specific sources and claims

### 4.1 AI-driven dynamic test selection

| Concept | Source | Status |
|---|---|---|
| LLM-based test generation | Multiple academic papers on LLM-for-test, e.g. *"Large Language Models are Few-Shot Testers"* (Schäfer et al., 2023) and follow-ups | Verified — active research area, but quality of generated tests is mixed and varies by codebase |
| LLM-based test selection (picking which existing tests to run) | (no specific source found) | **Not verified** — the specific application of LLMs to *select* tests at scan time, given a component and mutation context, appears novel. Related work exists in regression-test selection (e.g. EPA-based selection), but that work is structural, not LLM-driven. |
| Reliability of LLM test selection at scale | (no specific source found) | **Not verified** — this is the central empirical assumption this proposal raises. The pilot's primary metric is to measure this. |
| Sub-agent dispatch patterns for parallel AI workflows | Anthropic's Claude Code sub-agent documentation; OpenAI's Codex agent docs; general multi-agent orchestration literature | Verified — sub-agent dispatch is a well-known pattern in 2026 AI orchestration; reliability at the scale this proposal assumes (P=8–16 parallel sub-agents across N=500+ components) is not directly benchmarked. |

**Honest caveat:** This proposal's central design choice — letting the AI dynamically select tests at scan time, rather than maintaining a static test inventory — rests on an empirical assumption (current AI can reliably select appropriate tests given a component and mutation context) that has not been directly tested. The proposal flags this explicitly in [`WORKFLOWS.md` §9 open questions](./WORKFLOWS.md) and in [`COST.md` §11.2](./COST.md). The pilot's primary job is to measure this assumption.

### 4.2 Bidirectional documentation sync

| Concept | Source | Status |
|---|---|---|
| Bidirectional relationship fields in documentation | Common pattern (graph databases, ontology design); not novel | Verified — established pattern |
| Reverse-edge propagation as a separate workflow step | (no specific source found) | **Not verified** — the specific composition of "produce forward edges in one workflow, propagate reverse edges in a separate workflow" appears novel in the AI-documentation context. In graph database contexts, reverse edges are typically maintained transactionally, not via a separate workflow. |
| Idempotent reverse-edge sync with conflict handling on human edits | General pattern in CRDT and version-control literature | Verified — the techniques Workflow 4 uses (idempotent add/remove, file-level locking, human-edit reconciliation) are well-established |

### 4.3 Change-aware freshness via the implicit validation group

| Concept | Source | Status |
|---|---|---|
| Git archaeology / change-frequency analysis | Bird et al., *"Mining Version Control Archives"* (MSR 2005, cited 1000+) | Verified — established research field |
| Using commit history to identify related components | Zimmermann et al., *"Mining Version Histories to Guide Software Changes"* (IEEE TSE 2005, cited 800+) | Verified — established technique |
| Time-based staleness in documentation | Renovate, Dependabot, GitHub's stale-issue bot | Verified — common pattern, but time-based, not change-aware |
| Change-aware (hash-based) staleness for AI docs | (no specific source found) | **Not verified** — the specific model of "Workflow 4 stamps a validation hash into the MD file of every component; a GitHub Action compares that hash against the latest commit touching the owning component or any component in `Affects ∪ Affected By`" appears novel in combination, though the pieces are well-established |

**Honest caveat:** This proposal simplifies the prior form by deriving the validation group implicitly from `Affects` and `Affected By` rather than storing it as a separate field. The simplification is structural, not methodological — the staleness check algorithm is essentially unchanged from the prior form.

### 4.4 Removal of test names from component docs

| Concept | Source | Status |
|---|---|---|
| Component docs that omit test names | (no specific source found) | **Not verified** — to our knowledge, no prior AI-oriented component documentation format has explicitly refused to store test names. The argument for doing so (drift, invisible staleness, the question is dynamic not static) is presented in [`RATIONALE.md` §Part 4](./RATIONALE.md) but is not directly supported by prior literature we could find. |
| Trade-offs of static vs. dynamic test selection | Test selection literature (e.g. Stryker's mutation-coverage approach, EPA-based regression selection) | Verified — the trade-offs are well-studied; the specific trade-off of "AI-driven dynamic selection at scan time vs. AI-driven static enumeration in a stored inventory" is novel in this combination |

### 4.5 W2 single-algorithm scan with automatic MD replacement, and W4 always-nested-loop + hash stamping

These are the structural revisions introduced in this revision of the proposal. Both are internal architectural choices rather than empirical claims; their justification is design-driven rather than evidence-driven.

| Concept | Source | Status |
|---|---|---|
| Single-component impact scan with automatic replacement (one algorithm, no mode flags; an existing MD is fully replaced by the fresh scan result) | (no specific source found) | **Not verified** — we have not found prior art for a single-component impact scanner that uniformly creates-or-replaces its own documentation artifact on every run. The closest analogue is incremental vs. full test selection in regression-test selection literature (e.g. EPA-based selection), but those techniques are structural, not LLM-driven, and they do not manage a per-component documentation file. |
| Always-full-nested-loop reverse-edge sync (re-derive every `Affected By:` from scratch every time) | (no specific source found) | **Not verified** — graph-database literature typically maintains reverse edges transactionally at write time, not via a separate full-rescan workflow. The "always re-derive from scratch" pattern is closer to a build-system / linting model (re-derive everything for correctness, with skip-if-equal as an optimization) than to a graph-database model. The specific choice — making the simpler full-nested-loop algorithm the default and treating incremental mode as an opt-in optimization for very large projects (flagged in [`WORKFLOWS.md` §9](./WORKFLOWS.md)) — appears novel in this combination. |
| Validation-hash stamping by the reverse-sync workflow (W4 as sole writer of `last_validated`) | (no specific source found) | **Not verified** — assigning the freshness stamp to the bookkeeping workflow rather than to the discovery workflows appears novel; build systems typically stamp artifacts at production time, not in a separate reconciliation pass. |
| The composition W6 → W2 → W4 (and W5 never invoking W2) | (no specific source found) | **Not verified** — the specific composition where the new-component workflow invokes the single-component discovery workflow (which triggers the reverse-sync + hash-stamping workflow), while the edit loop never invokes the discovery workflow, appears novel. |

**Honest caveat:** These revisions are introduced as design simplifications, not as empirical claims supported by external evidence. The W2 design — one algorithm, no outer loop, automatic replacement — is justified by simplicity: the difference from Workflow 1 is exactly the missing outer loop, and the create-or-replace branch point needs no caller-supplied mode flags. The W4 design (always re-derive + stamp the validation hash into every component's MD file) is justified by drift-proofing and by making W4 the single authority for the freshness contract that the CI staleness check relies on. Both choices are reversible — a future revision could add an incremental mode to W4 for very large projects, or surface the old-vs-new Affects delta during replacement — without changing the rest of the proposal. The pilot's job is not to validate these design choices directly, but to surface whether they introduce any unexpected failure modes in practice.

## 5. Confidence summary for this proposal

| Claim | Confidence | Reason |
|---|---|---|
| Mutation testing is a valid technique for impact discovery | Medium-high (methodology valid), Low (specific application to AI docs novel) | Methodology is established; the specific application is unvalidated |
| AI-driven dynamic test selection can replace a static test inventory | **Low — this is the central unverified assumption** | No direct evidence; the pilot's primary job is to measure this |
| The six-workflow lifecycle (full-system scan, single-component scan, AI-driven testing, reverse sync, edit loop, new-component bootstrap) is novel in combination | Medium | Negative result from search; could exist under a name we didn't find |
| Change-aware staleness with implicit validation group is novel | Medium | Negative result from search; specific combination appears new |
| Full-system scan cost is `O(N × M × C × T / P)` | High (arithmetic) | Direct from algorithm |
| Full-system scan token cost for a 500-component project is ~110M tokens | Medium | Per-iteration token count is a reasoned estimate; the dominant cost is per-candidate Workflow 3 calls |
| Single-component scan cost is `O(M × C × T / P)` | High (arithmetic) | Direct from algorithm |
| This proposal's docs are more accurate than the prior form | **Low — depends on AI test-selection quality** | The prior form stored tests statically (deterministic but stale-prone); this proposal selects dynamically (fresh but AI-quality-dependent). The trade is the empirical question. |
| Cross-file edit failure rate drops 50–80% | Low | Carried over from the prior form; this proposal's specific validation not done |
| Navigation cost reduction 40–70% | Medium-high | Arithmetic, slightly better than the prior form due to smaller doc schema |
| Verification loop time reduction 60–90% | Medium (lower than the prior form) | Depends on Workflow 3's reliability; range is wider |

## 6. What we did not verify

- We did not read the underlying Cortex, DORA, or Amazon sources that riftmap.dev cites. (Carried over from the prior form.)
- We did not run any benchmarks on this proposal's workflow algorithms — none exist. This is a proposal, not a pilot.
- We did not validate the [`COST.md`](./COST.md) numbers against a real codebase — no pilot has been run.
- We did not exhaustively search for this proposal's specific combination of (mutation testing for impact discovery) + (AI-driven dynamic test selection) + (reverse-edge sync workflow) + (change-aware implicit validation group) under every possible name. There may be prior art we missed.
- We did not verify that AI orchestration layers (Claude Code, Codex, etc.) can execute Workflow 1 reliably at the proposed scale (N=500+ component full-system scan with checkpoint/resume and P=8+ parallel sub-agents). This is an assumption based on observed 2026 agent capabilities; it has not been tested at the proposed scale.
- **Critically: we did not verify that current AI can reliably select appropriate tests for a component given a mutation context.** This is the central empirical assumption this proposal makes. The pilot's primary job is to measure this. If the assumption fails, the system degrades gracefully (more `inconclusive` verdicts, more human-flagged entries in `notes`) but the cost-benefit shifts.

## 7. How to cite this repo

If you build on this proposal, cite the repo directly. Do not cite the cost estimates as if they were benchmarks. If you run a pilot and get different numbers — especially for the full-system scan cost, the revalidation-trigger frequency, or Workflow 3's `inconclusive` rate — please publish them and open an issue. Negative results are especially welcome. The single most valuable contribution a pilot can make is to settle the question of whether AI-driven dynamic test selection is reliable enough to replace a static test inventory — that is the empirical assumption on which the proposal's value proposition turns.
