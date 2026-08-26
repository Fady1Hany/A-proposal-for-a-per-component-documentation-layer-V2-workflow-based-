# Cost model — Component Impact Discovery

> The cost model is structural rather than empirical: it describes *what* determines cost and *how* the costs scale, with order-of-magnitude estimates. **No one has measured this proposal on a real codebase.** The numbers below are reasoned hypotheses, not benchmarks.

## How to read this document

Each cost claim has a **confidence level**:

- **High** — supported by arithmetic or directly observable behavior of test suites and LLM APIs.
- **Medium** — direction is well-supported, magnitude is a reasoned guess based on analogy.
- **Low** — single source, vendor-cited, or composite of weaker claims.

Treat all numbers as **hypotheses to measure on your own codebase**, not as predictions.

## 1. The cost regimes

The proposal has four cost regimes, deliberately separated by the workflow split.

| Regime | Triggered by | Workflow | Cost shape |
|---|---|---|---|
| **Full-System Scan** | Initial setup on an existing project; on-demand full re-scan | Workflow 1 | `O(N × M × C × T / P)` — high, one-time or rare |
| **Single-Component Scan** | New component creation (via W6); revalidation trigger (via CI); on-demand re-validation or re-scan | Workflow 2 | `O(M × C × T / P)` — medium, per-component (no outer loop; automatic MD replacement adds no cost) |
| **Edit Component Loop** | Agent edits an existing component | Workflow 5 (verify loop only — W5 has no relation to W2 and can invoke W4 )| `O(R × |Affects(X)| × T / P)` — low, per-edit |
| **Create New Component** | Agent creates a new component | Workflow 6 | `O(codegen) + O(M × C × T / P)` (the Workflow 2 cost) |

Reverse Sync (Workflow 4) is essentially free in tokens (pure file I/O, no LLM calls) but is no longer `O(E)` — it is now `O(N² × L)` file reads because W4 always runs as a full nested loop over every MD file, plus one hash-stamping pass that writes `last_validated` into every file. It is included in the cost model in §4 below. At typical project sizes (N ≤ 2000) the wall-time cost is seconds-to-minutes, which is negligible next to the LLM-driven workflows.

Where:
- `N` = number of components in the project.
- `M` = average number of mutation operators applied per component (typically 4–10; see [`WORKFLOWS.md` §1.5](./WORKFLOWS.md)).
- `C` = average number of candidate components tested per mutation (after the optional pre-filter; typically 5–50 for a single-component scan, and N×0.1 for a full-system scan with shared-imports pre-filter). Before pre-filtering, C = N − 1 — Workflow 2 tests **every** other component in the system.
- `T` = AI-driven testing time per component — the time Workflow 3 takes to dynamically select and execute appropriate tests for one candidate, given a mutation context.
- `P` = parallel sub-agent fan-out factor (typically 4–16, bounded by LLM API rate limits and test-runner concurrency limits).
- `R` = retry count in the Workflow 5 edit loop (default 3).
- `|Affects(X)|` = number of components listed under X's `Affects:` headline (typically 1–10).
- `L` = average length of an `Affects` list (typically 1–10). Used in the W4 cost model.

The cost asymmetry across regimes is the entire justification for the workflow split. The full-system scan is expensive enough that it should be a deliberate, planned operation. The single-component scan is the same algorithm minus the outer loop — one invocation wide — so it is cheap enough to run on every new component and every revalidation trigger. The edit-component loop is cheaper still — bounded by the existing Affects list size — and is what runs on every agent edit. The create-new-component workflow is the most expensive per-event (because it includes code generation plus a full W2 scan) but is rare enough to be acceptable.

## 2. Full-System Scan cost (Workflow 1)

### 2.1 Wall-time cost

For a project with N components, M mutations per component, C candidates per mutation (after pre-filter), T seconds per AI-driven test (per candidate), and P parallel sub-agents:

```
full_system_scan_wall_time = N × M × C × T / P
```

**Worked example:**

| Project size | N | M | C | T (sec) | P | Full-system scan wall time |
|---|---|---|---|---|---|---|
| Small (50 components, fast suite) | 50 | 5 | 10 | 5 | 4 | 52 minutes |
| Medium (200 components) | 200 | 5 | 30 | 10 | 8 | 10.4 hours |
| Large (500 components) | 500 | 5 | 50 | 10 | 8 | 43.4 hours |
| Very large (2000 components) | 2000 | 8 | 80 | 15 | 16 | 333 hours |

**Confidence: High.** This is arithmetic. The main uncertainties are:
- Whether the assumed M (mutations per component) holds — for components with large public surfaces, M may be 10–20, multiplying cost 2–4×.
- Whether the pre-filter (shared imports, existing `Affects` entries) reduces C as assumed. A poorly-filtered candidate set increases cost linearly.
- Whether Workflow 3's per-test time T scales as assumed. AI-driven test selection involves an LLM call per candidate, which dominates the wall time for fast test suites.

### 2.2 Token cost

Each mutation iteration (one mutation of one component) involves:
1. Reading the mutated component's source (1–4k tokens depending on file size).
2. Generating the mutation (LLM call, 0.5–2k tokens).
3. Dispatching C candidate testing tasks to Workflow 3 sub-agents.
4. Each sub-agent: reads the candidate's source (1–4k tokens), reasons about test selection (1–3k tokens), runs the tests (no LLM tokens, but wall time), reads the test output (1–3k tokens), returns a verdict (0.2–1k tokens). Per sub-agent: 3–11k tokens.
5. Merging sub-agent verdicts (small LLM call, 0.5–1k tokens).
6. After all mutations, generating the doc itself (1–3k tokens).

Per component per mutation: roughly `3–11k × C / P` tokens for the sub-agents, plus `3–10k` of orchestration overhead (reading the mutated source, generating the mutation, merging verdicts; the doc-generation pass adds 1–3k once per component). For N=500, M=5, C=50 (pre-filtered), P=8, and 7k tokens per sub-agent: `500 × 5 × (7k × 50 / 8) ≈ 500 × 5 × 44k ≈ 110M` tokens for the full-system scan, plus roughly 8–25M of orchestration overhead across all 2,500 mutations. This matches the Large row in the worked-example table below.

**Worked example (token cost):**

| Project size | N | M | C | Tokens per sub-agent | Tokens per mutation (subtotal) | Full-system scan tokens |
|---|---|---|---|---|---|---|
| Small (50 components) | 50 | 5 | 10 | 5k | 12.5k | ~3M |
| Medium (200 components) | 200 | 5 | 30 | 7k | 26k | ~26M |
| Large (500 components) | 500 | 5 | 50 | 7k | 44k | ~110M |
| Very large (2000 components) | 2000 | 8 | 80 | 10k | 50k | ~800M |

*Per-mutation subtotal = tokens per sub-agent × C / P; it excludes the small per-mutation orchestration overhead (~3–10k tokens per mutation), which adds roughly 10–20% on top of the totals above.*

**Confidence: Medium.** Per-iteration token count is a reasoned estimate based on typical LLM API usage in agent workflows. The dominant cost is the per-sub-agent token usage in Workflow 3, which is bounded by the candidate's source size and the AI's selected test count. Real values will vary by LLM provider, code language, and test verbosity. The very-large case (~800M tokens) is high enough that practical deployment would likely require shrinking M or C, or accepting partial coverage.

### 2.3 The "intentional high cost" framing

The proposal explicitly frames the full-system scan as a **high-cost, one-time operation**. This is a design choice, not a defect:

- The cost is paid once per project. After the scan, the relationships stored in the COMPONENT_*.md files and docs exist; incremental maintenance is cheap.
- The cost is bounded and predictable (arithmetic on N, M, C, T, P). It is not open-ended.
- The cost is parallelizable. Adding sub-agents reduces wall time linearly (up to API rate limits).
- The cost is resumable. Workflow 1 supports checkpoint and resume ([`WORKFLOWS.md` §1.10](./WORKFLOWS.md)), so a long scan that gets interrupted doesn't have to restart from zero.

The honest comparison: the alternative to the full-system scan is "the agent rediscovers the relationships stored in the COMPONENT_*.md files from scratch on every task." Over a year of agent use on a 500-component codebase, the cumulative token cost of repeated rediscovery is much higher than the one-time full-system scan cost. The scan amortizes.

## 3. Single-Component Scan cost (Workflow 2)

Workflow 2 runs the same mutation + test-all algorithm as Workflow 1, minus the outer loop: it works on exactly one component you hand it. If an MD already exists under that component's name, the scan result automatically replaces it — replacement is not a separate mode and costs nothing extra, because the same scan runs either way.

### 3.1 Wall-time cost

For a single component with M mutations, C candidates per mutation (C = N − 1 before the optional pre-filter), T seconds per AI-driven test, and P parallel sub-agents:

```
single_component_scan_wall_time = M × C × T / P
```

**Worked example:**

| Project size | M | C | T (sec) | P | Single-component scan wall time |
|---|---|---|---|---|---|
| Small | 5 | 5 | 5 | 4 | 31 seconds |
| Medium | 5 | 10 | 10 | 4 | 2.1 minutes |
| Large | 5 | 20 | 10 | 4 | 4.2 minutes |
| Very large | 8 | 30 | 15 | 8 | 7.5 minutes |

**Confidence: High.** Arithmetic. The candidate count C is the main uncertainty; C starts at N − 1 (every other component in the system) and the optional pre-filter typically shrinks it to 1–50 depending on the component's connectivity. For a broadly-connected component (e.g. a new shared utility), C can spike to 30+.

### 3.2 Token cost

Per single-component scan: roughly `M × (sub-agent token cost × C / P + per-mutation overhead)`. With M=5, C=20, P=4, and 7k tokens per sub-agent: `5 × (7k × 20 / 4 + 10k) = 5 × 45k = 225k` tokens.

**Worked example (token cost):**

| Project size | Tokens per single-component scan |
|---|---|
| Small | 25k–50k |
| Medium | 50k–150k |
| Large | 150k–300k |
| Very large | 300k–600k |

**Confidence: Medium.** Per-iteration token count is a reasoned estimate based on typical LLM API usage in agent workflows. The dominant cost is the per-sub-agent token usage in Workflow 3, which is bounded by the candidate's source size and the AI's selected test count. Automatically replacing an existing MD (vs. creating a new one) adds no cost — the same scan runs either way.

### 3.3 Revalidation-triggered single-component scans

The CI staleness check (see [`SPEC.md` §7](./SPEC.md)) triggers Workflow 2 (with the path to the component's MD, which W2 automatically replaces) when it detects that the hash in the component's MD file is older than the latest commit touching the owning component or any component in its `Affects` / `Affected By` lists. The frequency of these triggers depends on the project's commit velocity:

| Commit velocity | Revalidation triggers per week | Weekly token cost (large project, ~150–300k per scan per §3.2) |
|---|---|---|
| Low (1–5 commits touching related components/week) | 1–3 | 150k–900k |
| Medium (5–20 /week) | 3–10 | 450k–3M |
| High (20+ /week) | 10–30 | 1.5M–9M |

Smaller projects scale down proportionally: a small project (25k–50k per scan per §3.2) with 1–3 triggers/week spends 25k–150k tokens/week, and a very large project (300k–600k per scan) with 10–30 triggers/week can reach 3M–18M tokens/week.

**Confidence: Low.** The mapping from commit velocity to revalidation triggers depends on how often commits touch components in the same validation group, which is highly project-specific. The numbers above are reasoned from typical project structures; real projects will vary. Revalidation uses the same full single-component scan as new-component bootstrap — W2 has one algorithm, and the hash comparison in CI is what decides when it runs.

## 4. Reverse Sync cost (Workflow 4)

### 4.1 Wall-time and token cost

Workflow 4 is now **always a full nested loop over every `COMPONENT_<name>.md` file in the project**: it picks a file, scans every other file's `Affects:` list, and rebuilds that file's `Affected By:` list from scratch. No diff input, no edge-set argument. The algorithm is the same regardless of which workflow invoked it.

This means W4's cost is no longer `O(E)` where E is the diff size. It is now `O(N² × L)` file reads, where N is the number of MD files and L is the average `Affects` list length. The good news: it is still pure file I/O — **zero LLM calls, zero test runs, zero tokens**.

```
reverse_sync_wall_time = O(N² × L × file_io_time)
reverse_sync_token_cost = 0
```

**Worked example (wall time, full nested loop):**

| Project size | N (MD files) | L (avg Affects length) | Total file reads (N × L) | Wall time (sequential, 1ms/read) | Wall time (parallel, 8 threads, 1ms/read) |
|---|---|---|---|---|---|
| Small (50 components) | 50 | 3 | 150 | ~0.15s | <0.1s |
| Medium (200 components) | 200 | 5 | 1,000 | ~1s | <0.2s |
| Large (500 components) | 500 | 5 | 2,500 | ~2.5s | ~0.4s |
| Very large (2000 components) | 2000 | 5 | 10,000 | ~10s | ~1.3s |
| Monorepo (10000 components) | 10000 | 7 | 70,000 | ~70s | ~9s |

The writes are also bounded: at most N writes per W4 invocation (one per file), but in steady state most files' `Affected By:` lists do not change, so the skip-if-equal optimization (§4.5 steps 2.4 and 3.1.3 in [`WORKFLOWS.md`](./WORKFLOWS.md)) keeps actual writes close to the diff size.

**Confidence: High.** This is just I/O. The numbers above assume sequential 1ms-per-read file I/O; real filesystems and OS-level caches make typical numbers lower. For monorepo-scale projects (10000+ components), the W4 nested loop becomes seconds-to-minutes of wall time — still negligible next to the LLM-driven workflows (W1 alone is 43 hours for a 500-component project), but worth noting. An optional incremental mode (re-deriving only the `Affected By` lists of components whose `Affects` changed) is flagged as an open question in [`WORKFLOWS.md` §9](./WORKFLOWS.md).

## 5. Edit Component Loop cost (Workflow 5)

W5 has **one cost component**: the verify loop (the per-retry W3 fan-out with cancel-on-first-failure). Workflow 5 has no relation to Workflow 2 — it can invoke W4 .

### 5.1 Verify-loop wall-time cost

For one edit attempt against a component X with `|Affects(X)|` consumers, `P`-way parallelism, and `T` per-test time:

```
edit_attempt_wall_time = |Affects(X)| / P × T
```

Across `R` retries (worst case — no retry succeeds until the last):

```
edit_loop_wall_time_worst = R × |Affects(X)| / P × T
```

In practice, the expected wall time is much lower because of the **cancel-on-first-failure** semantics: as soon as any sub-agent reports `affected`, the remaining sub-agents are cancelled and the next retry begins. So the *expected* wall time per failing retry is closer to `(expected_failure_position / P) × T`, and the full `|Affects(X)| / P × T` is only incurred on the final successful retry.

**Worked example (verify loop):**

| Component fan-out | R | T (sec) | P | Worst-case wall time | Expected wall time (with cancel) |
|---|---|---|---|---|---|
| Tiny (1 affected) | 3 | 5 | 1 | 15s | ~5s |
| Small (5 affected) | 3 | 10 | 4 | 37.5s | ~12s |
| Medium (10 affected) | 3 | 10 | 4 | 75s | ~25s |
| Large (20 affected) | 3 | 15 | 8 | 112.5s | ~40s |

**Confidence: High on the arithmetic, Medium on the expected-vs-worst gap.** The arithmetic is direct from the algorithm. The "expected wall time" column assumes a roughly uniform distribution of failure positions across retries; real distributions will vary.

### 5.2 Verify-loop token cost

Per edit attempt: `|Affects(X)| × per_sub_agent_token_cost / P + per_attempt_overhead`. With 5 affected consumers, P=4, and 7k tokens per sub-agent: `5 × 7k / 4 + 5k ≈ 14k` tokens per attempt. Across R=3 retries: `~42k` tokens per edit.

**Worked example (verify loop, tokens):**

| Component fan-out | R | Tokens per sub-agent | Tokens per edit (worst case) |
|---|---|---|---|
| Tiny (1) | 3 | 5k | 15k |
| Small (5) | 3 | 7k | 42k |
| Medium (10) | 3 | 7k | 60k |
| Large (20) | 3 | 10k | 150k |

**Confidence: Medium.** Per-sub-agent token cost is a reasoned estimate. The cancel-on-first-failure semantics reduces expected token cost (failed retries don't run all sub-agents to completion), so real averages should sit below the worst-case numbers.

### 5.3 No post-edit scan cost

Unlike a design where the edit loop also triggers a single-component scan after every safe edit, W5 attaches **zero** scan cost to an edit: it never invoke workflow 2 and it can Workflow 4.

### 5.4 Frequency-driven total cost

W5 runs on every agent edit. For a project with E edits per week, most of which produce a safe edit (the common case — `verdict=unsafe` is rare in practice):

| Edit velocity | Edits/week | Average verify-loop cost per edit | Weekly W5 token cost |
|---|---|---|---|
| Low (1–5 edits/week) | 3 | ~30k | 90k |
| Medium (5–20 edits/week) | 10 | ~50k | 500k |
| High (20+ edits/week) | 30 | ~80k | 2.4M |

To this, add the revalidation-triggered W2 costs from §3.3, which are driven by commit velocity, not edit velocity — the two regimes are fully decoupled because W5 never invokes W2.

**Confidence: Low.** The edit-velocity → W5-cost mapping is highly project-specific and depends on the average `|Affects(X)|` per edit.

## 6. Create New Component cost (Workflow 6)

### 6.1 Wall-time and token cost

W6 = (code generation) + (W2 cost — the full single-component scan) + (W4 cost, which is essentially free). The dominant cost is the W2 invocation; code generation adds a one-time LLM call.

```
w6_wall_time = codegen_time + w2_wall_time
w6_token_cost = codegen_tokens + w2_token_cost
```

For a typical new component:
- Code generation: 10–30 seconds, 5–20k tokens (depending on component size and complexity).
- Workflow 2: 2–8 minutes, 50–300k tokens (per §3 — the full single-component scan). W6 invokes W2 on the new component; no MD exists under its name yet, so W2 creates one (it would automatically replace it if one existed).
- Workflow 4: seconds of file I/O, 0 tokens (plus the hash-stamping pass, still 0 tokens).

**Worked example:**

| New component complexity | Codegen tokens | W2 tokens | Total W6 tokens | W6 wall time |
|---|---|---|---|---|
| Simple utility (1 symbol) | 5k | 50k | 55k | ~2 min |
| Medium service (5–10 symbols) | 12k | 150k | 162k | ~5 min |
| Complex component (15+ symbols) | 25k | 300k | 325k | ~10 min |
| Codegen retries (broken first attempt) | 30k × 2 | 0 (baseline failed, no W2) | 60k | ~5 min (just codegen retries) |

**Confidence: Medium on the codegen half, High on the W2 half.** The W2 numbers come directly from §3. The codegen numbers are reasoned estimates based on typical LLM API usage for source-code generation; real values will vary by LLM provider, language, and component complexity.

### 6.2 When W6 is expensive

W6's expensive case is **codegen failure with retries**. If the LLM cannot produce a working component on the first try, W6's step 5 baseline-test path fires, the file is deleted, and the agent (or a human) reformulates the creation prompt and retries. Each retry costs ~10–30k tokens of codegen, plus the baseline test run. If the LLM cannot produce working code after several retries, W6 returns `verdict=codegen_failed` and the agent must either choose a different creation strategy or escalate to a human.

This is the W6-specific analogue of W3's central empirical question: can current AI reliably produce a working new component from a creation prompt on the first try? If not, W6's value proposition (one-shot new-component creation + documentation) weakens, and the workflow becomes a multi-retry loop. Pilot data would measure this directly.

## 7. Where this proposal costs MORE than the prior form (V1)

The proposal's costs are higher than the prior form's in two specific places:

1. **Per-candidate token cost.** The prior form ran the project's existing test suite against the mutated component and observed which tests failed — no AI involvement per candidate. This proposal dispatches a Workflow 3 sub-agent per candidate, which makes an LLM call to select tests, runs them, and reports a verdict. The per-candidate cost is therefore higher — but the trade is that the proposal does not maintain a static test inventory, which the prior form did (and which itself had a maintenance cost via the prior form's nightly testing workflow).
2. **AI test-selection reliability.** The prior form's `targeted_verification` list was deterministic; the runner just executed the named tests. This proposal's test selection is non-deterministic — the same candidate, same mutation, same baseline can yield different verdicts across runs. The mitigation (re-dispatch when the first verdict is `affected`) doubles the cost for edges that turn out to be stable; the trade is fewer false positives.

These are not regressions; they are **costs shifted from static infrastructure (test inventory maintenance) to dynamic operation (per-candidate AI calls)**. The question is whether the trade is worth it. The argument: the prior form's static test inventory drifted every time a developer added a test, and the drift was invisible (the inventory file looked current even when its contents were stale). This proposal's per-candidate cost is paid every scan, but the result is always against the current state of the codebase. There is no drift; there is only the question of whether the AI reliably selects the right tests, which is empirically answerable.

## 8. Where this proposal costs LESS than the prior form (V1)

1. **No test inventory maintenance.** The prior form's project-level testing workflow ran nightly to refresh `test_inventory.json` — a constant background token cost. This proposal has no static test inventory; the cost is paid only when a scan runs.
2. **No `validation_group` field maintenance.** The prior form stored a separate `validation_group` field per component; this proposal derives it implicitly from `Affects ∪ Affected By`. The schema is smaller, the doc is smaller, the context budget spent reading the doc is smaller.
3. **Smaller doc schema overall.** The prior form had 9 fields; this proposal has 6. The agent reads less per component, and the workflow writes less per scan.

## 9. Where the gains are large (carried over from the prior form, with updates)

### 9.1 Cross-file refactors and changes with downstream consumers
- **Estimated improvement:** failed/broken-edit rate drops 50–80% on changes with ≥3 downstream consumers.
- **Confidence:** Medium.
- **Reasoning:** `Affects` is derived from mutation testing rather than human memory, so it's more accurate than the prior form. The 50–80% range should hold or improve.
- **Baseline problem:** AI coding agents have measurably higher failure rates on cross-file changes than on localized changes. Vendor-cited data (riftmap.dev, citing DORA/Cortex) puts the gap around 30%, but see [`SOURCES.md`](./SOURCES.md) for caveats.

### 9.2 Navigation cost on large codebases
- **Estimated improvement:** context budget saved on navigation is 40–70% for codebases above ~200 components.
- **Confidence:** Medium-high.
- **Reasoning:** Same arithmetic as the prior form. The doc is smaller (6 fields vs 9), so per-doc navigation cost is lower; the relationships stored in the COMPONENT_*.md files are more accurate (behavioral, not structural), so per-task navigation hits are fewer.

### 9.3 Test selection
- **Estimated improvement:** verification loop time drops 60–90% for changes where Workflow 3's dynamic selection produces accurate verdicts.
- **Confidence:** Medium (lower than the prior form's Medium-high, because the assumption that AI can reliably select tests is novel and unmeasured).
- **Reasoning:** The accuracy bar is "does the AI select the right tests" rather than "does the static inventory match the test suite." If the AI's selection is good, the verification loop is faster (only relevant tests run); if it's bad, the loop is longer (re-dispatches to confirm `affected` verdicts). The net is a function of AI quality, which is what a pilot would measure.

## 10. Where the gains are small or zero (carried over from the prior form)

### 10.1 Single-file, self-contained changes
- **Estimated improvement:** 0–20%. Possibly net-negative due to doc-reading overhead.
- **Confidence:** High.
- **Reason:** If the change doesn't touch anything else, the doc adds overhead for zero benefit.

### 10.2 Small codebases (<50 components)
- **Estimated improvement:** negligible.
- **Confidence:** High.
- **Reason:** The agent can grep the whole thing in one pass. The map's value is marginal. Additionally, full-system scan cost for small projects is low enough that the cost-benefit is unclear — the proposal's value proposition scales with project size.

### 10.3 Well-typed, well-named code
- **Estimated improvement:** smaller than for dynamic or legacy code.
- **Confidence:** Medium.
- **Reason:** Strong type systems (TypeScript, Rust, Haskell) already encode much of the structural dependency for free. The proposal still adds value (behavioral impact that types don't capture), but less than for dynamic languages.

### 10.4 Greenfield work
- **Estimated improvement:** near zero.
- **Confidence:** High.
- **Reason:** No history, no implicit knowledge to surface. The doc is mostly empty. Scan cost is low (few components), but so is the benefit.

## 11. Where this proposal can actively hurt

### 11.1 AI test-selection quality
- **Risk:** Medium-High (this is the new risk this proposal introduces).
- **Reason:** for example: If Workflow 3's AI cannot reliably select appropriate tests for a component, the relationships stored in the COMPONENT_*.md files will be incomplete (false negatives — edges missed because no test was selected to catch them) or noisy (false positives — edges recorded because a flaky test happened to fail in a way the AI couldn't distinguish from a real impact).
- **Mitigation:** Re-dispatch when the first verdict is `affected` (catches some flaky-test false positives); cross-check `affected_tests` against the project's actual test inventory (drops hallucinated tests); flag `inconclusive` verdicts in `notes` for human review. A pilot would measure the actual false-positive and false-negative rates and inform whether the AI is good enough.

### 11.2 Full-system scan token cost on very large projects
- **Risk:** Medium.
- **Reason:** A 2000-component project with a 15-second per-test time costs ~800M tokens for the full-system scan. This is not free.
- **Mitigation:** The scan is parallelizable and resumable. Run it over a weekend. The cost is paid once. For projects above 2000 components, consider shrinking M (fewer mutations per component) or accepting partial coverage (skip components with low `Affected By` breadth).

### 11.3 Over-trust
- **Risk:** Medium.
- **Reason:** Agents tend to treat documented facts as ground truth. A confident doc that's slightly wrong is more dangerous than no doc at all.
- **Mitigation:** Docs are derived from actual test behavior, not human memory, so they're less likely to be "slightly wrong." But mutation testing has its own failure modes (compile errors masking downstream impact, non-deterministic tests, AI test-selection gaps). The agent SHOULD treat `Affects` and `Affected By` as advisory, not authoritative.

## 12. Bottom line

For a **mature, 500-component codebase with cross-cutting changes**:

| Metric | Prior form (V1) estimate | This proposal's estimate (with W4 nested loop + hash stamping) | Confidence |
|---|---|---|---|
| Cross-file edit success rate | ~70% → ~92%+ | ~70% → ~90–93% (range wider due to AI test-selection uncertainty) | Medium |
| Context budget per navigation-heavy task | −35 to −55% | −40 to −60% (smaller doc schema) | Medium-high |
| Verification loop time | −65 to −92% | −50 to −90% (wider range; depends on AI test-selection quality) | Medium (lower than the prior form) |
| Overall AI-assisted maintenance productivity | +25 to +45% | +20 to +45% (lower bound reflects AI test-selection risk) | Low-medium |
| One-time full-system scan cost | ~12.5M tokens, ~1.6 hours wall time (8-way parallel) | ~110M tokens, ~43 hours wall time (8-way parallel) — higher because per-candidate AI calls dominate. Unchanged from earlier form of this proposal. | High (arithmetic) |
| Per-event single-component scan cost (new component via W6, or CI revalidation) | ~25–50k tokens, ~4 minutes wall time | ~150–300k tokens, ~4 minutes wall time — higher per event than V1, but no nightly test-inventory refresh cost. The same cost applies whether W2 creates a new MD or automatically replaces an existing one — there is one algorithm, no modes. | High (arithmetic) |
| Per-edit W5 cost | ~30k tokens, ~15s wall time | ~15–150k tokens, ~5s–2 minutes wall time — W5 never invokes W2, but can invoke W4. | High (arithmetic) |
| W4 cost per invocation | `O(E)` file reads, milliseconds, 0 tokens | `O(N² × L)` file reads + one hash-stamping write pass, seconds-to-minutes, 0 tokens. Higher in wall time, but still negligible next to LLM-driven workflows. The trade is drift-proofing: every `Affected By:` list is re-derived from scratch and every file's validation hash re-stamped every time. | High (arithmetic) |

For a **small or clean codebase**, expect single-digit improvements at best, and possible net-negative if the doc overhead exceeds the gains. The proposal's value proposition scales with project size and component interconnection.

## 13. What this is not

- **Not a multiplier on AI intelligence.** The agent does not get smarter. This proposal stops the agent from making two specific mistakes: missed downstream consumers and wrong test selection.
- **Not a substitute for source inspection.** The agent should still read the actual code before editing.
- **Not a benchmark.** These are hypotheses. Anyone piloting this proposal should instrument before/after metrics on their own codebase.

## 14. Recommended pilot

If you want to validate these numbers:

1. Pick a real 200+ component codebase with a fast (≤30s) test suite.
2. Instrument baseline metrics for 4–8 weeks: cross-file edit success rate, context budget per task, verification loop time, frequency of stale-doc-induced failures.
3. Run Workflow 1 (Full-System Scan). Measure: wall time, token cost, components with empty `Affects`, false-positive edges (manually spot-check 10%), Workflow 3 `inconclusive` rate.
4. Operate the system for 8–12 weeks. Measure the same metrics.
5. Compare. Pay particular attention to:
   - Full-system scan token cost vs. the arithmetic prediction in §2.2.
   - Workflow 3's `inconclusive` rate (the key new metric this proposal introduces).
   - Revalidation trigger frequency vs. the PR-velocity model in §3.3.
   - Stale-doc detection latency (how quickly the staleness check fires after a related PR merges).
   - AI test-selection quality (false-positive and false-negative rates vs. a manual ground-truth check on a sample of edges).
6. Publish the results, positive or negative. Open an issue on this repo with the data.

The single most important metric to measure is **Workflow 3's `inconclusive` rate and its false-positive/false-negative rates**. This is the empirical question this proposal raises that its prior form did not. If Workflow 3 turns out to be unreliable, the system degrades gracefully but the cost-benefit shifts; if it turns out to be reliable, the system's value proposition is strong. The pilot's job is to settle that question with data.
