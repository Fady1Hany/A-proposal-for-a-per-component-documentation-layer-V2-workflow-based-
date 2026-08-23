# SPEC: `COMPONENT_<name>.md` — Component Impact Discovery

> Status: draft v0.1 — open for discussion.
>
> This spec defines the format, fields, ownership rules, and freshness model for the per-component documentation files produced by the six workflows defined in [`WORKFLOWS.md`](./WORKFLOWS.md).

## 0. Conventions used in this spec

- **MUST / SHOULD / MAY** are used in the RFC 2119 sense.
- "Workflow" refers to one of the six workflows defined in [`WORKFLOWS.md`](./WORKFLOWS.md): Full-System Impact Scan, Single-Component Impact Scan, AI-Driven Testing, Reverse Relationship Sync, Edit Component Loop, Create New Component.
- "The agent" refers to whichever AI coding agent (Claude Code, Codex, Cursor, etc.) is consuming or producing `COMPONENT_<name>.md` files.
- "The orchestration layer" refers to the project-level file (`AGENTS.md`, `.cursorrules`, or a Claude Code workflow file) that tells the agent how to consume `COMPONENT_<name>.md` and which workflow to invoke when.
- "The component relationships" refers to the bidirectional `Affects` / `Affected By` edges stored in the set of `COMPONENT_*.md` files. There is **no central graph** and no separate graph artifact — the relationships live only in the distributed per-component MD files.

## 1. Purpose

`COMPONENT_<name>.md` is a small, structured documentation file co-located with a source component, designed to be **written and maintained by AI workflows** and **read by AI coding agents**. It gives the agent exactly the information it needs to answer two questions about that component, without reading the whole codebase:

1. *If I change this component, which other components will be affected?*
2. *Which other components, when changed, will affect this component?*

The file deliberately does **not** answer a third question — *which tests should I run?* — because that question is answered dynamically by Workflow 3 at the time the agent actually modifies the component. Storing a static test list inside the doc would force the doc to track the test inventory, which changes far more frequently than impact relationships do, and would defeat the purpose of letting the AI decide what to test.

## 2. File placement

- One `COMPONENT_<name>.md` per component.
- Placed **in the same directory** as the component's primary source file.
- Named `COMPONENT_<name>.md` where `<name>` is the component's unique identifier (uppercase prefix, lowercase name — e.g. `COMPONENT_authentication_service.md`).
- `<name>` MUST be unique across the project to prevent duplication of `COMPONENT_<name>.md` files.
- When a user requests a change to a component, the prompt MUST use the exact name defined in the corresponding `COMPONENT_<name>.md` file. The agent uses the provided name to locate `COMPONENT_<name>.md`.
- A component is defined as "a unit of code with a public interface that other components depend on" — typically a class, a module, or a small cohesive group of files.

**Granularity rule:** define components as small as practical. If a component is defined at the microservice level, a change to one function inside that microservice may have relationships the doc does not capture. Smaller components → tighter relationships in the `COMPONENT_*.md` files → more reliable AI verification.

The agent does not need to know the code of every component connected to the one it is modifying. It needs the code of the component it is modifying. Connected components can be treated as black boxes — the agent dispatches Workflow 3 against them and observes the verdicts. This keeps the active context small while still allowing impact verification across the connected set.

## 3. Required fields

Every `COMPONENT_<name>.md` MUST contain the following fields, in this order. All are produced by workflows; none are hand-authored.

### 3.1 `component`
The component's name as it appears in code. MUST match the `<name>` in the filename.

### 3.2 `location`
The file path (relative to repo root) of the component's primary source file. May be an array of paths if the component spans multiple files. Produced by the workflow from filesystem inspection.

### 3.3 `last_validated`
The validation hash: the git commit hash (or short hash) and ISO-8601 date at which this document's relationships were last confirmed. **This field is the freshness contract**, and it is written **exclusively by Workflow 4** — W4 is responsible for stamping the hash into the MD file of every component during its reverse-sync pass. No discovery workflow and no human writes this field.

`last_validated` is **not** a staleness threshold by itself. A 200-day-old `last_validated` is fine if no related component has changed in that window. A 1-day-old `last_validated` is stale if a related component had a commit land an hour ago. See §7 for the change-aware freshness model.

### 3.4 `Affects`
The list of components whose behavior was observed to change when this component was modified. Each entry is **a component name only** — no test names, no prose descriptions of impact, no mutation operator identifiers. The workflow populates this list from Workflow 3's verdicts during a Workflow 1 or Workflow 2 scan; specifically, it is the unique set of components Y where at least one mutation of this component X produced a `verdict=affected` from Workflow 3. When Workflow 2 scans a component that already has an MD, the freshly discovered list **replaces** the old one in full — automatic replacement, with the previous version preserved in git history.

This list is **not** a static dependency list inferred from imports. A component may import another and never appear in its `Affects` list, if the import is purely structural and the importing component's behavior doesn't actually depend on the imported component's behavior. Conversely, a component may appear in `Affects` without any direct import, if the impact flows through reflection, dynamic dispatch, plugin systems, or shared state.

### 3.5 `Affected By`
The list of components whose modification was observed to affect this component. Each entry is **a component name only** — symmetric to `Affects`. This field is populated by Workflow 4 (Reverse Relationship Sync) after Workflow 1, Workflow 2, or Workflow 6 (via W2) completes; it is the inversion of the `Affects` field across all the `COMPONENT_*.md` files in the project.

A human SHOULD NOT edit this field directly. Workflow 4 is its sole canonical producer. If a human edits it, the next Workflow 4 run will reconcile — accepting the human edit if the relationships stored in the COMPONENT_*.md files supports it, or overwriting it with a note in `provenance` if the relationships stored in the COMPONENT_*.md files contradicts it.

### 3.6 `provenance`
Workflow metadata: which workflow produced or last updated this doc, when, with what mutation operators, an approximate token cost, and the sub-agent fan-out factor used in the producing scan. Useful for debugging and cost tracking (see [`COST.md`](./COST.md)). Not consumed by the agent during normal operation; the agent SHOULD ignore this field unless explicitly debugging.

## 4. Optional fields

All optional fields are also workflow-produced. They may be omitted if the workflow has no information to populate them.

- **`public_interface`** — explicit list of exported symbols considered stable. Changes to these are breaking. The workflow identifies these from language-level visibility modifiers (e.g. `public` in Java, `export` in TypeScript, non-`pub(crate)` in Rust) plus convention-based detection (e.g. functions named `init`, `main`, `setup`).
- **`stability`** — one of `experimental`, `stable`, `frozen`, `deprecated`. The workflow infers this from a combination of: code annotations (`@experimental`, `@deprecated`), test coverage level, and the breadth of `Affected By` (broadly-used components default to `stable`; narrowly-used to `experimental`).
- **`notes`** — free-form context that does not fit elsewhere. The workflow may populate this with observations from Workflow 3 (e.g. "non-deterministic test `test_session_expiry` was excluded from impact analysis — see provenance"). Use sparingly.

## 5. Format

- Markdown, with a fixed field layout (see [`examples/COMPONENT_TEMPLATE.md`](./examples/COMPONENT_TEMPLATE.md)).
- A JSON Schema is provided in [`schema/COMPONENT_NAME.schema.json`](./schema/COMPONENT_NAME.schema.json) for tooling.
- The on-disk format is Markdown-first; tooling may parse it into structured form for validation and CI checks.
- HTML comments (`<!-- ... -->`) MAY be used to embed workflow metadata not intended for agent consumption (e.g. provenance details, mutation operator hashes). The agent SHOULD ignore unknown HTML comments.

## 6. Ownership and maintenance rules

1. **All required fields are produced by workflows.** Hand-authoring is permitted only as a post-workflow edit (see rule 4) and MUST be marked as such.
2. **Workflows are the canonical producer.** When a workflow runs against a component, it regenerates the entire file. The previous version is preserved in git history. (Workflow 2's automatic replacement of an existing `COMPONENT_<NAME>.md` is this rule in action — the fresh scan result fully replaces the old file.)
3. **`last_validated` is updated exclusively by Workflow 4**, not by hand and not by the discovery workflows. W4 stamps the validation hash into the MD file of every component during its reverse-sync pass. A human cannot "bump" `last_validated` without running a workflow — that would defeat the freshness contract.
4. **Human edits after a workflow run are permitted** but MUST be marked in two places. First, an HTML comment in the doc body: `<!-- [human-edited] YYYY-MM-DD: reason -->`. Second, a matching entry in the structured `provenance.human_edits` array (`{date, reason}`) so that tooling can audit human edits without parsing comments — this is the same information in machine-readable form, as defined in [`schema/COMPONENT_NAME.schema.json`](./schema/COMPONENT_NAME.schema.json). Human edits do not update `last_validated`. If a human edit changes the meaning of a field (e.g. corrects a wrong `Affects` entry), the next workflow run will reconcile — either accepting the human edit (if the scan confirms it) or overwriting it (if the scan contradicts it) with a note in `provenance`.
5. **`Affected By` and `last_validated` are produced exclusively by Workflow 4.** Humans SHOULD NOT edit them; if they do, the next Workflow 4 run will overwrite their edit. If a human needs to flag a relationship the workflow missed, the right place to do so is `notes`, not `Affected By`.
6. **The orchestration layer** (e.g. `AGENTS.md`) defines which workflow runs when. The default triggers:
   - Workflow 1 (Full-System Scan): manually invoked once at project setup.
   - Workflow 2 (Single-Component Scan): triggered when an existing component needs revalidation or an on-demand scan — invoked with the path to its `COMPONENT_<NAME>.md`; W2 mutates that component, tests every other component, and **automatically replaces** the existing MD. Triggered by Workflow 6 for new components (no MD exists under the component's name yet, so W2 creates one). Workflow 2 has no outer loop — it always works on exactly one component.
   - Workflow 3 (AI-Driven Testing): invoked by Workflows 1, 2, and 5 as a sub-step; may also be invoked on demand for ad-hoc impact analysis.
   - Workflow 4 (Reverse Relationship Sync): invoked by Workflows 1, 2,  6 and 5 ( also can ) after they complete — always the same full nested loop over every MD afters in the project, no diff input. Workflow 4 also stamps the validation hash (`last_validated`) into the MD file of every component; it is the sole writer of that field
     
   - Workflow 5 (Edit Component Loop): invoked by the orchestration layer whenever the agent modifies an existing component; may also be invoked on demand for edit verification. Workflow 5 has **no relation to Workflow 2** — it never invokes it; it only reads the Affects list and verifies the edit via Workflow 3. W5 can invoke W4
     
   - Workflow 6 (Create New Component): invoked by the orchestration layer whenever the agent creates a new component; may also be invoked on demand for one-shot new-component bootstrap. W6 invokes Workflow 2 on the new component (no MD exists under its name yet, so W2 creates one), and W2 triggers Workflow 4.
7. **CI checks** (see §7) verify that `last_validated` is consistent with the validation group (the union of `Affects` and `Affected By`). They do not enforce a time threshold.

## 7. Change-aware freshness model

A time-based threshold ("older than 90 days = outdated") is rejected. This proposal uses a change-aware model instead.

### 7.1 The validation group

For any component X, the validation group is the set of components whose behavior is coupled to X's — both directions. Concretely:

```
validation_group(X) = Affects(X) ∪ Affected_By(X)
```

No separate field is needed; the validation group is implicit in the two required fields. The staleness checker computes it on demand by reading the doc.

### 7.2 The staleness check

A GitHub Actions workflow (or equivalent CI) runs the staleness check on every push and on a schedule (default: daily). The Action's job is to look at the hash in each MD file and compare it against the latest commit made on the component that owns the file, or on any component that is with it in its `Affects` list, or on any component in its `Affected By` list. For each component X:

1. Read `COMPONENT_<X>.md` to get the validation hash in `last_validated` (stamped by Workflow 4), plus `Affects` and `Affected By`.
2. Compute `validation_group(X) = Affects(X) ∪ Affected_By(X) ∪ {X}` (X itself — the component that owns the file — is always in its own validation group).
3. Find the latest commit in the project's git history that touches any component in `validation_group(X)`.
4. If that commit is newer than the hash recorded in X's `last_validated`, emit a warning:

   ```
   Outdated Component Documentation:
   Component X's MD file was last validated at commit <hash1>,
   but a newer commit <hash2> touched Component Y, which is
   either the owner of this file or a member of its
   Affects / Affected By lists.
   Revalidation is required.
   ```

5. The warning does not fail CI by default. It fails CI only if configured to do so (e.g. for `frozen`-stability components where staleness is unacceptable).

### 7.3 Why not time-based

The time-based threshold has two failure modes:

- **False positive:** a stable, low-churn component that hasn't been touched in 91 days is flagged as stale, even though nothing about it has changed. The team either ignores the warning (devaluing all warnings) or runs a workflow to "refresh" the doc for no semantic reason (wasting tokens).
- **False negative:** a component whose doc was last validated yesterday, but whose related component just had a breaking PR merged today, is treated as fresh. The doc is actually stale — the related component's change may have invalidated an `Affects` entry — but the 90-day clock just reset.

The change-aware model eliminates both failure modes by tying staleness to actual code changes in the relevant validation group, not to wall-clock time.

### 7.4 Revalidation trigger

When the staleness check fires for component X, the CI workflow SHOULD trigger Workflow 2 with the path to X's `COMPONENT_<NAME>.md`. Workflow 2 mutates X, tests every other component in the system, **automatically replaces** X's MD with the freshly discovered `Affects` list, and invokes Workflow 4 to refresh all `Affected By:` lists across the project and re-stamp the validation hashes.

## 8. Agent consumption contract

An AI agent reading a `COMPONENT_<name>.md` SHOULD:

1. Read `last_validated` first.
2. Check the staleness status (via the CI warning, or by re-running the staleness check locally).
3. If stale, invoke Workflow 2 (Single-Component Scan) for this component before trusting `Affects` and `Affected By`.
4. Always trust `location` — it is structural and stable across minor changes.
5. Treat `Affects` and `Affected By` as advisory, not authoritative. They reflect observed behavior at `last_validated` time. If the agent's planned change is unusual (e.g. changing a function's async-ness, or removing a public symbol), the agent SHOULD run Workflow 3 manually against the suspected-affected components, even if the doc lists them.
6. Before modifying the component, save a copy of the original source file (path documented in `location`). Then **invoke Workflow 5 (Edit Component Loop)** rather than applying the edit directly. Workflow 5 reads the `Affects` list from the component's MD, backs up the source, applies the edit, fans out Workflow 3 sub-agents against every component under `Affects:`, restores on any `affected` verdict, retries with a different edit, and only commits the edit when all sub-agents report `not_affected`. Workflow 5 has **no relation to Workflow 2** — it never invokes it. Bypassing W5 means giving up the runtime edit-verify guarantee — the agent MAY do so for trivial edits (e.g. comments, whitespace) but MUST document the bypass in the commit message.
7. **Before creating a new component, invoke Workflow 6 (Create New Component)** rather than writing the code and documenting it separately. Workflow 6 generates the source, writes it, and invokes Workflow 2 to populate the `Affects` list (no MD exists under the new component's name yet, so W2 creates one — it would automatically replace it if one existed). W2 triggers Workflow 4 to propagate `Affected By` entries into the docs of any existing components the new one affects and to stamp the validation hashes.

The agent SHOULD NOT look for a list of test names inside `COMPONENT_<name>.md`. There is no such list, by design. When the agent needs to know what tests to run, Workflow 3 (invoked by W5 for edits, or by W2/W6 for new-component or scan-time testing) handles test selection dynamically.

## 9. What is intentionally not in the spec

- **A specific workflow implementation.** The six workflows are specified in [`WORKFLOWS.md`](./WORKFLOWS.md) at the algorithm level, not as code. Any runner that implements the algorithm is compliant.
- **A specific language.** The format is language-agnostic. The mutation operator catalog (see [`WORKFLOWS.md` §1.5`](./WORKFLOWS.md)) is language-specific; the spec does not mandate a particular set.
- **A specific test framework.** The doc stores component names, not test identifiers; the format is therefore framework-agnostic. Workflow 3 handles test selection and execution at runtime.
- **A static test inventory.** The doc deliberately does not store test names. This is the defining design choice of the proposal — see [`RATIONALE.md`](./RATIONALE.md) §"Why test names do not belong in component docs".
- **A specific orchestration layer.** `AGENTS.md`, `.cursorrules`, or a Claude Code workflow file are all acceptable. The spec only requires that *some* orchestration layer exists and that it points to the six workflows and routes edit requests through W5 and new-component requests through W6.
- **A specific AI provider for Workflow 3.** Any provider that can read code, reason about test selection, and execute test runners is acceptable. The spec deliberately under-specifies Workflow 3's internals (see [`WORKFLOWS.md` §3.5`](./WORKFLOWS.md)) so that the empirical question of how well current AI can dynamically select appropriate tests can be measured by a pilot rather than answered by speculation.
- **A specific code generator for Workflow 6.** Any LLM that can produce source code from a natural-language creation prompt is acceptable. The spec is silent on whether the code generator is the same model as the test selector (W3) or a different one; this is an implementation detail left to the runner.

## 10. Versioning

This spec is `v0.1`. The major version (0) indicates proposal stage. A `1.0` release would follow a successful pilot on a real codebase (see [`COST.md` §"Recommended pilot"](./COST.md)).

The `spec_version` field is OPTIONAL in `COMPONENT_<name>.md`. If present, it MUST match the major.minor version of this spec. A mismatch is a non-fatal warning; the agent SHOULD still attempt to consume the doc.
