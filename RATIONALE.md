# Rationale — Component Impact Discovery

> Why this exists. The argument has three parts: the asymmetry between human and AI developers (Part 1), why existing approaches do not solve it (Part 2), and why this proposal's specific design choices — discovered-impact rather than assumed-impact, AI-driven dynamic testing rather than static test inventory, and `Affects`/`Affected By` rather than `depends_on`/`used_by` — are the right ones (Parts 3-5).

## Part 1 — The asymmetry between human and AI developers

A human developer who has worked on a codebase for any length of time carries a large amount of implicit knowledge: where the important components live, which parts call which, which tests cover what, and which changes are likely to ripple. When asked to modify a component, they often already know where it is and what it touches.

But if they don't know, they can navigate there in a few keystrokes or read the documentation. The problem is that the documentation that exists today is made for humans, and that's exactly the gap this proposal is tackling: how to create documentation for each component that's suitable for an AI agent, not for humans.

Without component-level documentation made exclusively for AI, an AI may still be able to navigate the codebase and discover the necessary information, or read the existing documentation. However, doing so forces the model to consume a significant portion of its context window reconstructing information that could have been provided explicitly.

This is not because humans have larger working memory than AI agents. It is because humans build a **persistent mental model** that survives across interactions. They can selectively recall the parts that matter without reloading the whole project.

An AI coding agent does not have this. Each interaction starts with a near-empty context window. The agent must rediscover the structure of the codebase through tools — file search, code search, dependency tracing, reading source — every time. As the codebase grows, the cost of this rediscovery grows with it.

## Part 2 — Why existing approaches do not fully solve this

- **Human-oriented documentation** (READMEs, ADRs, Javadoc) is written for a reader who already has the implicit context. It describes *what* something does, rarely *where it sits in the relationships stored in the COMPONENT_*.md files* or *what breaks if it changes*.
- **Auto-generated code graphs** (Aider repo map, Cursor indexing, CodeGraph MCP servers) extract structure on demand, but they cost context budget on every query and they do not capture *behavioral* relationships — they capture *structural* ones (imports, call edges). A component may import another and never break when it changes; conversely, it may break when a structurally-unrelated component changes through reflection or shared state.
- **Project-level context files** (AGENTS.md) give the agent behavioral rules and conventions for the whole repo, but they are one file, not a per-component map.
- **Hand-authored component docs** go stale the moment a developer forgets to update them. The drift is not a failure of discipline; it is a structural problem with hand-authoring fields that change every time the codebase changes.

The gap is: there is no convention for **a per-component documentation artifact that tells an AI agent which components are behaviorally affected when this one changes — and which is kept current by workflows, not by humans**.

## Part 3 — Why discovered-impact beats assumed-impact

The defining design choice of this proposal is that `Affects` and `Affected By` are **discovered through actual testing**, not inferred from imports or hand-authored. This choice deserves justification.

The alternative — inferring impact from imports — has a well-known failure mode. Imports capture structural dependency, not behavioral dependency. Component A may import component B for a single utility function and never break when B's behavior changes; conversely, A may break when an unrelated-looking component C changes, because C's modification alters a shared side effect (a global cache, a database schema, a configuration default) that A relies on. Treating imports as impact relationships produces both false positives (import that never breaks) and false negatives (impact that flows through non-import channels).

The other alternative — hand-authoring impact — has the drift problem described in Part 2. The fields that most directly govern agent behavior (`Affects` and `Affected By`) are also the fields most likely to drift, because they encode judgment about a codebase that changes underneath them. Putting humans in charge of fields that drift fastest, then inventing a threshold to compensate, is the structural problem the prior form of this proposal tried and failed to solve.

This proposal's claim is that **behavioral impact is empirically answerable**, and that the answer is best produced by running the question rather than guessing at it. Workflow 1 or Workflow 2 mutates the component in specific ways, Workflow 3 runs the candidate-affected components' tests, and the verdicts — affected or not affected — directly produce the `Affects` list. The human's memory is unnecessary; the test suite is the source of truth.

This is the same conceptual move that mutation testing made in the test-quality-assessment field decades ago: rather than asking humans "is this test suite good?", mutate the code and observe whether the tests catch the mutations. This proposal applies the same move to impact discovery: rather than asking humans "what does this component affect?", mutate the component and observe which other components' tests fail.

## Part 4 — Why test names do not belong in component docs

The prior form of this proposal stored a `targeted_verification` field inside each `COMPONENT_<name>.md` — a list of test names to run after modifying the component. This proposal removes that field, and the rationale deserves explicit statement.

The argument for storing test names is convenience: the agent doesn't have to figure out which tests to run; the doc tells it. The argument against is drift: every time a developer adds, renames, or removes a test, the doc goes stale, and the staleness is invisible because the doc still *looks* current — it has a `last_validated` date, the file paths are correct, but the test names no longer match the actual test suite. The agent runs the named tests, they pass, and the agent reports success — while the real regression tests, the ones that would have caught the agent's bug, were never run.

This proposal's solution is to remove the static test list entirely. Workflow 3 — the AI-driven testing workflow — dynamically decides which tests to run at the moment the agent needs to verify a change. The AI inspects the component, considers the mutation context, selects appropriate tests from the project's current test inventory (plus any ad-hoc tests the AI writes on the fly), runs them, and returns the verdict. The doc stores only the relationship (`X affects Y`); the *how to test Y* question is answered at test time, against the current state of the test suite.

This trade is honest about a risk: it assumes current AI can reliably select appropriate tests given a component and a mutation context. That empirical question is exactly what a pilot would measure (see [`WORKFLOWS.md` §9 open questions](./WORKFLOWS.md)). If the answer turns out to be "no, current AI cannot reliably do this," the system degrades gracefully — Workflow 3 returns `inconclusive` more often, the relationships stored in the COMPONENT_*.md files have more gaps, and humans review more flagged entries in `notes` — but it does not break. The doc remains readable; the agent remains able to use it; the failure is gradual, not catastrophic.

## Part 5 — Why `Affects` and `Affected By` rather than `depends_on` and `used_by`

The prior form of this proposal used `depends_on` and `used_by` as the primary relationship fields. This proposal replaces them with `Affects` and `Affected By`. The terminology shift is intentional.

`depends_on` and `used_by` describe **structural relationships**: which components import which. They are a snapshot of the import graph at `last_validated` time. They go stale whenever the import graph changes, even if the behavioral relationships haven't changed.

`Affects` and `Affected By` describe **behavioral relationships**: which components' behavior was observed to change when this one was modified. They are a snapshot of the relationships stored in the COMPONENT_*.md files at `last_validated` time. They go stale only when the *behavioral* relationships change — which happens far less frequently than the import graph changes.

The two perspectives are not redundant — they answer different questions. `depends_on` answers "what does this component import?" `Affects` answers "what does this component *break*?" The agent's actual question, when modifying a component, is the second one. The agent does not care that X imports Y; the agent cares that modifying X will break Y's tests. Storing the answer to the agent's actual question — rather than a proxy for it — makes the doc more useful per token of context budget spent reading it.

This proposal therefore drops `depends_on` and `used_by` entirely. The structural information is recoverable on demand from the codebase itself (Aider, Cursor, CodeGraph MCP servers all provide it). The behavioral information is not recoverable on demand — it requires running the test suite under mutation, which is what the workflows exist to do. Storing the behavioral information and recovering the structural information at query time is the right division of labor.

## Part 6 — Why the workflow is what makes this reliable

The naive objection is: "AI-generated documentation will drift faster than human-authored documentation, because there's no human judgment slowing it down." This is the right concern to take seriously.

The answer is that the workflow itself — not the file format, not the AI model — is what makes the documentation reliable. Specifically:

- **Reproducibility** — a workflow that takes a codebase at commit C and produces a doc is reproducible. Two runs against the same commit produce the same doc (modulo LLM non-determinism, which is bounded and detected via re-runs). Human authoring is not reproducible.
- **Coverage** — a workflow that mutates every public symbol of every component produces complete coverage. Humans miss components, miss symbols, miss edge cases.
- **Freshness tied to code, not to calendar** — the change-aware staleness model (see [`SPEC.md` §7](./SPEC.md)) ties doc freshness to actual code changes in the validation group, not to wall-clock time. A workflow run is triggered by a related change, not by a calendar timer.
- **Cost discipline** — the full-system scan cost is explicit and bounded (see [`COST.md`](./COST.md)). The single-component scan cost is cheap relative to the full-system scan — it is the same algorithm minus the outer loop, one component wide instead of N, so it can run on every new component and every revalidation trigger. The cost model is honest about what's expensive and what isn't.
- **Provenance** — every doc carries the workflow ID, commit, operators used, token cost, and fan-out factor of the run that produced it. For Workflow 2 runs, it also records whether an existing MD was automatically replaced by the fresh scan, so that an audit can distinguish creations from replacements. A human-authored doc has none of that provenance; an audit requires reading the human's mind.
- **W4 drift-proofing** — Workflow 4 is always a full nested loop over every MD file in the project, regardless of which workflow invoked it. This means drift in any `Affected By:` entry is corrected on every W4 run, not just when the diff happens to touch that file. The trade-off is a higher wall-time cost (`O(N² × L)` instead of `O(E)`), but at zero token cost — the additional seconds of file I/O are invisible next to the LLM-driven workflows. W4 is also the workflow responsible for stamping the validation hash (`last_validated`) into the MD file of every component, which is exactly the field the CI staleness check compares against git history — one workflow, one authority, no drift.

The workflow doesn't make AI-generated docs *perfect*. It makes them *auditable* — you can see which mutations were applied, which tests Workflow 3 ran, which verdicts it returned, which edges were added to the relationships stored in the COMPONENT_*.md files, and at what token cost. That auditability is what makes the documentation trustworthy enough to be the agent's primary reference.

## Part 7 — What this proposal is not

- **Not a replacement for source inspection.** The agent should still read the actual code before editing. The doc is a navigation and impact map, not a substitute for the source.
- **Not a multiplier on AI intelligence.** The agent does not get smarter. This proposal stops the agent from making two specific mistakes: missing downstream consumers and selecting the wrong tests. It does not make the agent better at writing code.
- **Not a tool.** This proposal specifies workflows at the algorithm level. A particular implementation (a Claude Code workflow file, a Codex workflow, a Python script orchestrating an LLM API) is compliant if it implements the algorithms in [`WORKFLOWS.md`](./WORKFLOWS.md).
- **Not a claim that humans are out of the loop.** Humans review and may edit fields after a workflow runs. The point is that humans are not the *primary author*; the workflow is. This is a subtle but important shift in accountability.
- **Not a claim that manual tests are obsolete.** Existing hand-authored tests remain part of the project and remain valuable as records of previously discovered bugs, business rules, and regression cases. The system simply does not store their names inside the component Markdown files. Workflow 3 is free to discover and execute them as part of its dynamic test selection.

## Part 8 — Why now

The shift from "AI as autocomplete (code generator)" to "AI as agent that edits across multiple files (can maintain a large codebase)" is what makes this worth proposing. When an agent only completed the next line, it did not need an impact map. When an agent modifies a service and is expected to update its consumers and run the right tests, the absence of an impact map becomes the dominant failure mode.

The arXiv paper *"Evaluating AGENTS.md"* (Feb 2026) found that repository-level context files can *reduce* task success rates — which is a warning that **design matters more than existence**. This proposal takes that warning seriously: the format is small, co-located, freshness-stamped, and stores only behaviorally-discovered relationships (not structural assumptions, not test inventories, not human-authored intent). The six-workflow lifecycle is what turns "AI generates docs" into "AI maintains docs that stay trustworthy across years of changes."

The reason this proposal is possible now and wouldn't have been in 2024 is that AI orchestration layers (Claude Code, Codex, Cursor Agent, Aider's edit formats) have matured enough to execute multi-step workflows reliably. Dispatching parallel sub-agents to test candidate-affected components, merging their verdicts, and writing the results back into structured Markdown is a routine agent operation in 2026; it was not in 2024. The proposal's workflow-based design is feasible now in a way it wasn't two years ago.
