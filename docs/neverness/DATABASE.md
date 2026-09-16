# Neverness database notes

Neverness uses SQLite for player state, server state, static/reference data, and related services. Database changes therefore need to be reviewed as deployment changes, not only as code changes.

## Current connection model

`db.py` uses an autocommit-oriented connection model (`isolation_level=None`) and explicit short transactions for state-changing work. `transaction()` uses `BEGIN IMMEDIATE` and commits or rolls back around the operation.

A legacy module-level shared connection also exists. The connection wrapper applies a busy timeout and retries selected SQLite busy/locked failures.

Docker bootstrap applies:

```sql
PRAGMA busy_timeout=30000;
PRAGMA journal_mode=WAL;
```

The retained Neverness bootstrap fix also opens bootstrap schema connections with `isolation_level=None` so `static.ensure_schema()` and helper connections do not create an implicit-transaction self-lock.

## Schema ownership contract

Repository guidance says schema DDL belongs in `static.py` (`static.ensure_schema`), with reusable SQL/connection code in `db.py`. In practice, some runtime helpers in `db.py` still perform column/backfill checks and `ALTER TABLE` operations. Those exceptions to the contract should be reviewed and either documented or consolidated later.

## Migration paths

There are currently two deployment paths with different behavior:

### Non-Docker restart path

`restart.sh` can execute an idempotent one-off `migration.py` and remove it after successful use.

### Docker path

`docker/docker_entrypoint.sh` invokes `docker/docker_bootstrap.py`. Bootstrap calls `static.ensure_schema()` for both fresh and existing databases but does not execute `migration.py`.

This difference is tracked as NCR-005 in `CODE_REVIEW.md` and must be resolved before relying on one-off migrations for a Neverness Docker release.

## Static data validation

Docker bootstrap validates that a set of required static/reference tables exist and are non-empty. This checks basic usability but is not equivalent to validating a full semantic migration.

## Versioning observation

`system_properties.version` is initialized to `1`, but the initial review did not identify a complete version-driven migration dispatcher using that value. Do not assume database versioning is currently authoritative.

## Change rules

For future Neverness database changes:

1. Inspect the exact deployed schema and code path first.
2. Back up the persistent database before applying schema/data migrations.
3. Make migrations idempotent where practical.
4. Define Docker and non-Docker behavior explicitly; do not assume they are equivalent.
5. Test against a disposable copy of a realistic existing database as well as a fresh database.
6. Do not silently swallow migration failures that can leave persistent state partially upgraded.
7. Record schema/migration decisions in this file and `UPSTREAM_CHANGES.md` when they originate upstream.
