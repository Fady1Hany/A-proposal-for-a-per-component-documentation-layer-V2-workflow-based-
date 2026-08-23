<!-- COMPONENT_TEMPLATE.md — blank template for a COMPONENT_<name>.md file.
     Copy as COMPONENT_<your_component>.md.
     All required fields are produced by the workflows defined in WORKFLOWS.md —
     do not hand-author them. Human edits must be marked per SPEC.md §6 rule 4. -->

# Component: <name>

| Field | Value |
|---|---|
| `component` | `<name>` |
| `location` | `<path/to/primary/source/file>` |
| `last_validated` | `<commit_hash>` (`<YYYY-MM-DD>`) — validation hash, stamped exclusively by Workflow 4 |
| `spec_version` | `v0.1` |

## Affects

<!-- Component names only — no test names, no prose, no operator IDs.
     Populated by Workflow 1 / Workflow 2 from Workflow 3 verdicts.
     When Workflow 2 scans a component that already has an MD, the freshly
     discovered list replaces this one in full (automatic replacement;
     the previous version is preserved in git history). -->

- (none discovered yet)

## Affected By

<!-- Component names only. Produced exclusively by Workflow 4
     (Reverse Relationship Sync) — do not edit by hand. -->

- (none discovered yet)

## Public interface (optional)

<!-- Exported symbols considered stable; identified by the workflow from
     language-level visibility modifiers plus convention-based detection. -->

- (none)

## Stability (optional)

<!-- One of: experimental | stable | frozen | deprecated -->

`experimental`

## Notes (optional)

<!-- Free-form workflow observations; use sparingly. -->

(none)

## Provenance

<!-- Workflow metadata for debugging and cost tracking. Produced by the
     workflows; not consumed by the agent during normal operation. -->

- `workflow_id`: `<1–6>` (the workflow that last touched this doc)
- `commit`: `<commit_hash>`
- `operators`: `[<mutation operator IDs used>]`
- `replaced_existing_md`: `<true|false>` (Workflow 2 only — whether this scan automatically replaced an existing MD)
- `tokens`: `<approximate token cost>`
- `sub_agent_fan_out`: `<P>`
- `human_edits`: `[]` (append `{date, reason}` per SPEC.md §6 rule 4)
