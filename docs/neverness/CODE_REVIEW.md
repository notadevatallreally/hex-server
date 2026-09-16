# Neverness code review

Review started: 2026-09-16

Baseline reviewed: upstream `ee0f06b3ee04b4eb678fe9e464e4d207271f67da`, overlaid with the two known Neverness local fixes `3fd0d24` and `b8e9e9d`.

The purpose of this review is to understand the whole server before making a sequence of isolated fixes. Findings are recorded here first. Repair work should be grouped into coherent batches after the relevant subsystem has been fully traced.

## Review method

- Inspect the exact Neverness source before proposing a patch.
- Treat upstream code and assumptions as references, not substitutes for the deployed source.
- Trace state-changing paths through request parsing, transaction boundaries, persistence, response generation, and retry behavior.
- Distinguish confirmed defects from hypotheses that still require end-to-end tracing.
- Do not treat a passing test runner as proof until the test harness itself has been verified.
- Prefer preserving existing tested local commits over recreating them.

## Existing Neverness fixes retained

### NFX-001 — Docker bootstrap SQLite self-lock

**Status:** confirmed fix; retain.

Local commit: `3fd0d24` — `Fix Docker bootstrap SQLite self-lock`

`docker/docker_bootstrap.py` originally opened the schema/bootstrap connections with SQLite's default implicit transaction behavior. During `static.ensure_schema()` this could self-lock when helpers opened additional connections. The local fix changes the create and upgrade connections to `isolation_level=None`, aligning bootstrap behavior with the application's autocommit-oriented connection model and explicit short transactions.

Exact changes:

```python
sqlite3.connect(str(temporary), timeout=30.0, isolation_level=None)
sqlite3.connect(str(path), timeout=30.0, isolation_level=None)
```

### NFX-002 — unopened chest import scope

**Status:** confirmed fix; retain.

Local commit: `b8e9e9d` — `Fix unopened chest import during profile login`

In `application/profile_stream.py`, `from profile_db import db_get_unopened_chests` was indented inside the loop over purchased inventory. An account with no purchased inventory therefore skipped the import and later raised `UnboundLocalError` when the function attempted to load unopened chests. The local fix moves that import outside the loop.

## Confirmed findings

### NCR-001 — purchase quantity and balance are not validated

**Severity:** critical

**Area:** `services/store.py`

`apply_purchase(conn, user_id, item_id, quantity)` converts quantity with `int(quantity)` but does not require it to be positive and does not reject a purchase whose cost exceeds the user's balance.

The remaining balance is calculated as:

```python
remaining = balance - (int(cost) * int(quantity))
```

Consequences include:

- A negative quantity can increase currency rather than spend it.
- A positive purchase can drive a balance below zero.
- Inventory quantity is adjusted using the supplied integer quantity.
- Quantity has no identified upper bound at this layer.

The full repair should be designed only after all store call sites, expected client error responses, and retry/idempotency behavior have been reviewed.

### NCR-002 — purchase can commit and then fail while building the response

**Severity:** high

**Area:** `services/store.py`, application dispatcher

`handle_purchase()` executes `PurchaseStoreItemCommand` through the transactional dispatcher. After the transaction has completed, it attempts to read `result["redeemed"]` and `result.get("error_message", "")`.

`apply_purchase()` does not return a `redeemed` field. Its result contains purchase fields such as `remaining`, `currency`, `item_name`, `template_guid`, `quantity`, `item_id`, `deck_granted`, and `granted_list`.

This creates a post-commit `KeyError`: the purchase can already be durable while the client receives no valid success response. A client retry can therefore repeat a transaction that actually succeeded.

### NCR-003 — redeem can commit and then fail because response variables are undefined

**Severity:** high

**Area:** `services/store.py`

`handle_redeem()` obtains the dispatcher result and later constructs its response using `error_value` and `error_message`, but those variables are not defined in the function. `apply_redeem()` does return `redeemed` and `error_message`, so the response path appears incomplete.

As with NCR-002, the failure occurs after the state-changing command can already have committed.

### NCR-004 — test scripts can report assertion failures while exiting successfully

**Severity:** high

**Area:** `tests/run_all.py` and individual `tests_*.py` scripts

The aggregate runner treats a test script as failed only when the subprocess exits nonzero. Multiple test scripts use a local `run(name, fn)` helper that catches `AssertionError` and generic `Exception`, prints `FAIL` or `ERROR`, and does not re-raise or otherwise force a nonzero exit status.

At minimum, the pattern was observed in:

- `tests_store.py`
- `tests_leaves.py`
- `tests_targeting.py`
- `tests_conditions.py`
- `tests_statics.py`
- `tests_deck_abilities.py`
- `tests_az1_pack.py`

The complete test tree still needs to be enumerated. Passing output from the current aggregate runner cannot yet be treated as reliable evidence that all assertions passed.

`tests_store.py` also does not exercise `handle_purchase()` or `handle_redeem()`, so NCR-001 through NCR-003 currently lack direct handler-level coverage.

### NCR-005 — Docker deployment does not follow the documented one-off migration path

**Severity:** high

**Area:** database schema/deployment

Repository guidance says schema changes belong in `static.py` plus an idempotent one-off `migration.py` when required. `restart.sh` executes and then removes `migration.py`.

Docker startup follows a different path: `docker/docker_entrypoint.sh` runs `docker/docker_bootstrap.py`, which calls `static.ensure_schema()`, but it does not execute `migration.py`.

An upstream release can therefore behave correctly under the non-Docker restart path yet leave an existing persistent Docker database incompletely migrated when required migration logic exists only in `migration.py`.

### NCR-006 — replay event persistence can fail silently

**Severity:** medium

**Area:** `replay_db.py`, game event logging

Replay event recording includes broad `except Exception: pass` behavior. The game event path also treats event-logger failures as best-effort. A database lock, schema error, or unexpected event serialization problem can therefore cause replay-source events to disappear without producing an actionable failure signal.

## Findings requiring more tracing before severity/fix decisions

### NINV-001 — Steam identity is trusted when no Steam Web API key is configured

**Current concern:** potentially critical on a public server

`proxy.py` parses `steamId` and `steamauth`. With a configured `STEAM_WEB_API_KEY`, it validates the auth data and uses the validated Steam ID. Without an API key, the code explicitly trusts the caller-provided Steam ID and constructs a `steam:<steam_id>` token.

User identity/database helpers treat Steam ID as an authoritative player key in relevant paths. This could permit account impersonation, but the complete token-to-HConnect-session path must be traced before recording an end-to-end exploit as confirmed.

The same proxy path also accepts `Admin`, `Mod`, and `Founder` query values and persists user flags. Current searches have not established that those persisted flags grant meaningful privileges, so no privilege-escalation claim should be made until their consumers are traced.

### NINV-002 — alternate auth/fallback routes need end-to-end review

`/auth/hextransition`, `/auth/hextotp`, broad `steam`/`auth` fallbacks, and normal username/password paths need to be reviewed together. Some routes appear to issue identity/tokens without the same password checks as `/auth/hexlogin`, but intended legacy protocol behavior is not yet established.

### NINV-003 — cross-client chat delivery may race on sequence/send state

`services/TODO.md` documents that clients in the same room can see their own echo but fail to receive another client's messages. The repository hypothesis is that one connection thread mutates another handler's `scnt` and calls `send()` while that handler's own thread is also using those fields, without a per-handler send lock.

This remains a hypothesis until the actual send implementation, sequence semantics, and all cross-handler send sites are traced.

### NINV-004 — broad exception swallowing needs classification, not blanket removal

Broad exception handlers exist in rules/effects, commands, AI, social pushes, schema backfills, replay, and other modules. Some are deliberate best-effort projections while others may hide correctness failures. Each path needs to be classified by whether failure is optional, recoverable, or state-corrupting.

### NINV-005 — container runs as root despite UID-oriented filesystem setup

The Dockerfile changes ownership of selected paths to UID 1001, but it does not set `USER 1001`, and Supervisor has no `user=` setting. Current container processes therefore run as root. This is primarily a deployment-hardening inconsistency; it is not currently tied to a gameplay defect.

## Additional observations to revisit

- `apply_purchase()` appears to substitute a default unknown store item tuple rather than rejecting an unknown item ID. The consequences should be traced before assigning severity.
- `application/profile_stream.py` writes `/tmp/encoded_decks.bin` on login. It appears to be a debugging artifact shared across users/process activity and should be evaluated for necessity and information exposure.
- `static.system_properties.version` is initialized to `1`, but no complete version-driven migration mechanism has yet been identified.
- Existing Docker databases are schema-upgraded and static-data-validated, but startup tests are run only when a fresh database is created.
- Runtime DDL/backfill behavior still exists outside the stated `static.py` schema-ownership contract.

## Review progress

Completed initial pass:

- repository architecture and documented schema contracts
- Docker bootstrap/entrypoint/Supervisor layout
- SQLite connection and transaction architecture
- profile stream area relevant to the existing Neverness fix
- store purchase/redeem flow
- aggregate test runner and representative test scripts
- replay persistence
- initial authentication/proxy inspection
- initial chat/TODO inspection

Next passes, in order:

1. Authentication/session lifecycle: proxy token creation through HConnect identity/session binding, reconnects, token lifetime, active-client ownership, and user-flag consumers.
2. Service dispatch/protocol correctness: all state-changing handlers, commit/response boundaries, retries, cache/idempotency behavior.
3. Game-session lifecycle and persistence: creation, join/remove, ownership, concurrent writers, crash/reconnect behavior.
4. PvP RulesPort persistence/resume: transaction validation, state/hash handling, event/replay capture, sync and acknowledgement behavior.
5. Tournament scheduler/state: multi-process SQLite access, IDs, match transitions, restart recovery, cleanup.
6. Campaign/PvE progression: champions, decks, rewards, quests, chests, mail, store, Frost Ring.
7. Card/effect engine correctness: targeting, ownership, ability state, conditions, and exception handling.
8. Complete test-architecture audit and missing coverage map.
9. Operational/deployment review: migration parity, privileges, logs, persistence, backups, health/recovery behavior.

No new fixes should be made from these findings until the relevant review pass is complete and a repair batch is defined.
