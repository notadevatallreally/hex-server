# Neverness deployment

## Current production shape

Neverness runs as the PvE HEX server in CT103 under Proxmox, using Docker. Player/server state is stored in persistent SQLite data rather than treated as disposable container filesystem state.

The repository should become the authoritative source for application code. The production container should eventually be built/deployed from a specific commit or tag on `neverness`.

## Deployment principles

- Do not edit production source as the primary workflow.
- Review and commit changes in the fork first.
- Deploy an identified commit/tag.
- Keep persistent database and required game-data inputs outside disposable image layers.
- Back up persistent state before any migration or first deployment of schema-changing code.
- Verify the deployed checkout/image revision after deployment.
- If deployed code differs from the reviewed branch, reconcile the difference before applying a patch.

## Current Docker observations

- `docker/docker_entrypoint.sh` runs `docker/docker_bootstrap.py` before Supervisor starts the network services.
- Bootstrap creates or upgrades the database and validates required static tables.
- Fresh database creation can run the supported test suite.
- Existing database upgrades do not run the full startup tests.
- The Docker path does not currently execute the repository's documented one-off `migration.py` mechanism.
- The image adjusts ownership of selected paths to UID 1001, but no `USER 1001` is set and Supervisor does not select a non-root user; services therefore run as root in the current model.

## Deployment identity

Once the fork workflow is established, use tags for known-good deployments, for example:

```text
neverness-0.3.0-1
neverness-0.3.0-2
```

The exact naming scheme can change, but each production deployment should be traceable to one immutable Git commit.

## Required deployment documentation still to capture

Before changing the current production deployment workflow, record the actual CT103 values for:

- Docker image/build command and tag
- compose/run configuration
- persistent `HEX_DB_PATH`
- Records/gamedata mounts
- generated-data mount behavior
- published ports and reverse-proxy path
- environment flags
- restart/autostart mechanism
- log locations/retention
- database backup/restore procedure

Those values should be copied from the running deployment rather than inferred from repository defaults.
