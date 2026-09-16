# Neverness known issues

This file is the concise issue register. Detailed evidence and review notes live in `CODE_REVIEW.md`.

## Confirmed

| ID | Severity | Area | Summary | Status |
| --- | --- | --- | --- | --- |
| NFX-001 | fixed | Docker/SQLite | Bootstrap implicit transactions could self-lock during schema setup | Retain local fix `3fd0d24` |
| NFX-002 | fixed | Profile login | Unopened-chest import could be skipped for users with no purchased inventory | Retain local fix `b8e9e9d` |
| NCR-001 | critical | Store | Purchase quantity/balance validation is missing | Review complete enough to confirm defect; not patched yet |
| NCR-002 | high | Store | Purchase can commit and then raise while constructing response | Not patched yet |
| NCR-003 | high | Store | Redeem can commit and then fail because response variables are undefined | Not patched yet |
| NCR-004 | high | Tests | Several test scripts can print failures while exiting zero | Full test-tree audit pending |
| NCR-005 | high | Deployment/DB | Docker startup does not execute documented one-off migration path | Architecture decision pending |
| NCR-006 | medium | Replay | Replay event persistence can fail silently | Not patched yet |

## Under investigation

| ID | Area | Question |
| --- | --- | --- |
| NINV-001 | Authentication | Can caller-controlled Steam IDs impersonate existing accounts when Steam API validation is disabled? |
| NINV-002 | Authentication | What guarantees exist across transition/TOTP/fallback auth routes? |
| NINV-003 | Chat/networking | Is cross-client chat loss caused by concurrent sequence/send mutation? |
| NINV-004 | Error handling | Which broad exception handlers hide correctness failures versus intentional best-effort work? |
| NINV-005 | Container hardening | Should the service stop running as root, and what writable paths would need adjustment? |

## Process rule

Do not convert this list directly into isolated patches. Findings should be resolved in subsystem-sized repair batches after the corresponding review pass is complete.
