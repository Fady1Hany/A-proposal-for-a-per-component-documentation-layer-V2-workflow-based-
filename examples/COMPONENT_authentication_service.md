# Component: authentication_service

| Field | Value |
|---|---|
| `component` | `authentication_service` |
| `location` | `src/services/auth/authentication_service.py` |
| `last_validated` | `9f3ab2c` (`2026-08-19`) |
| `spec_version` | `v0.1` |

## Affects

- session_manager
- api_gateway
- user_profile

## Affected By

- config_loader
- user_repository

## Public interface (optional)

- `authenticate(user, credentials) -> Token`
- `refresh(token) -> Token`
- `revoke(token) -> void`

## Stability (optional)

`stable`

## Notes (optional)

Non-deterministic test `test_token_expiry_race` was excluded from impact analysis during the last scan — see provenance.

## Provenance

- `last_workflow_id`: `4` (Reverse Relationship Sync — rebuilt `Affected By` from all `Affects` lists and stamped the `last_validated` hash above; Workflow 4 is the sole writer of that field)
- `prior_workflow_id`: `2` (Single-Component Impact Scan — mutated this component, tested every other component in the system, and wrote the `Affects` list; an MD already existed under this component's name, so the fresh scan automatically replaced it)
- `commit`: `9f3ab2c`
- `operators`: `[RETURN_VALUE_CORRUPTION, BEHAVIOR_SWAP, EXCEPTION_INJECTION]`
- `replaced_existing_md`: `true` (set by the Workflow 2 scan that produced the `Affects` list)
- `tokens`: `~41k`
- `sub_agent_fan_out`: `8`
- `human_edits`: `[{date: 2026-08-20, reason: "fixed a typo in Notes"}]`

<!--
  Reading guide (see SPEC.md):
  - `Affects`     = components whose behavior was observed to change when this
                    component was mutated (discovered by W1/W2 via W3 verdicts).
  - `Affected By` = components whose modification was observed to affect this
                    component (rebuilt from scratch by W4 on every run).
  - `last_validated` = the validation hash, written exclusively by Workflow 4.
                    A GitHub Action compares this hash against the latest commit
                    touching authentication_service itself, or any component in
                    its Affects / Affected By lists — if a newer commit exists,
                    the doc is flagged stale and revalidation (Workflow 2) fires.
-->
