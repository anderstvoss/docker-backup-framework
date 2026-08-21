# Multi-Application Backup Groups

## 1. Purpose

Backup groups provide coordinated backup and recovery-point orchestration
across multiple Docker applications.

Groups do not replace application-level backup configuration.

Each application continues to own its own:

- application configuration
- Restic repository
- Restic repository credentials
- storage verification contract
- portable recovery target
- retention policy
- maintenance policy
- scheduling policy

This allows applications in the same group to use different backup storage
providers, datasets, pools, mounts, or other independent targets.

## 2. Configuration location

Source configuration:

`config/groups/<group>.conf`

Installed configuration:

`/etc/docker-backup/groups/<group>.conf`

A group configuration requires:

- `GROUP_NAME`
- `GROUP_MEMBERS`

Each member is encoded as:

`application|repository|portable-target`

Example:

    GROUP_NAME="media"

    GROUP_MEMBERS=(
        "app-one|app-one-operational|app-one"
        "app-two|app-two-operational|app-two"
    )

For every member:

- `/etc/docker-backup/apps/<application>.conf` must exist
- `/etc/docker-backup/repos/<repository>.conf` must exist
- `/etc/docker-backup/portable/<portable-target>.conf` must exist
- the repository config `APP_NAME` must match the member application
- the portable config `APP_NAME` must match the member application

## 3. Repository independence

A backup group does not define or own a shared Restic repository.

Each member uses the repository configured by its repository configuration.

Applications in the same group may therefore use:

- different Restic repository directories
- different TrueNAS datasets
- different storage pools
- different NFS exports
- different backup providers
- different credentials
- different retention policies

The group layer records immutable recovery coordinates across those independent
targets.

## 4. Execution semantics

Group backups execute members sequentially.

The group orchestrator invokes the existing application-level backup engines.

It does not reimplement filesystem capture, database capture, Restic handling,
portable backup creation, locking, or storage verification.

A first-generation group implementation must not hold multiple application
capture locks simultaneously.

Application-level locking remains authoritative.

## 5. Consistency model

A completed group generation represents a:

`coordinated-non-atomic`

recovery point.

Applications are captured sequentially and therefore do not represent a
globally atomic distributed transaction.

The group manifest records the exact recovery point selected for every member.

Applications requiring stronger cross-application consistency may later define
explicit quiesce/dependency hooks, but this is outside the initial group
contract.

## 6. Group generation

Each group operation receives a unique group generation identifier.

Example:

`20260821T180000Z-123456`

The group generation is distinct from each application's Restic generation.

Example relationship:

    group_generation
      |
      +-- app-one
      |     restic_generation=...
      |     db_snapshot=...
      |     files_snapshot=...
      |     portable_set=...
      |
      +-- app-two
            restic_generation=...
            db_snapshot=...
            files_snapshot=...
            portable_set=...

Application generations remain independently valid recovery points.

## 7. Manifest storage

Canonical manifests are stored under:

`/var/lib/docker-backup/groups/<group>/<group-generation>.json`

The directory must be root-owned and not contain secret values.

The manifest is canonical machine-readable metadata.

Human-readable command output may summarize the same information.

## 8. Manifest schema

Initial manifest schema version:

`1`

Example:

    {
      "schema_version": 1,
      "group": "media",
      "group_generation": "20260821T180000Z-123456",
      "consistency": "coordinated-non-atomic",
      "result": "PASS",
      "members": [
        {
          "application": "app-one",
          "repository": "app-one-operational",
          "portable_target": "app-one",
          "result": "PASS",
          "restic_generation": "20260821T180001Z-...",
          "db_snapshot": "<immutable-restic-id>",
          "files_snapshot": "<immutable-restic-id>",
          "portable_set": "<absolute-path>"
        }
      ]
    }

The manifest must never contain:

- Restic passwords
- database passwords
- application secrets
- environment-secret values
- authentication tokens

## 9. Completion semantics

A group generation is `PASS` only if every configured member completes all
required backup operations and immutable recovery coordinates are recorded.

If any member fails:

- the group generation is `FAIL`
- the manifest records the failed member
- later members are not run in the initial implementation
- backups already completed for earlier members remain independently valid
- the failed group generation must not be presented as a complete group
  recovery point

The framework must never delete successfully created member backups merely
because a later member fails.

## 10. Validation rules

Before capture begins, the group orchestrator must reject:

- missing group configuration
- missing `GROUP_NAME`
- empty `GROUP_MEMBERS`
- malformed member tuples
- duplicate application members
- duplicate repository members
- duplicate portable-target members
- missing application configuration
- missing repository configuration
- missing portable configuration
- repository/application identity mismatch
- portable/application identity mismatch

Initial duplicate-target rejection is intentionally conservative.

Support for deliberate sharing may be added later only with explicit semantics.

## 11. Restore contract

Group restore tooling must restore or validate only the immutable recovery
coordinates stored in the selected completed group manifest.

It must not substitute:

- `latest`
- a newer application generation
- a newer portable set
- another complete generation

unless the operator explicitly chooses a different recovery point.

This provides deterministic recovery of the selected group checkpoint.

## 12. Restore ordering

The initial implementation uses:

- backup order: `GROUP_MEMBERS` order
- restore order: reverse `GROUP_MEMBERS` order

Explicit dependency graphs or custom restore ordering may be added later.

## 13. Relationship to existing engines

The following existing application-level engines remain authoritative:

- `backup-repository`
- `portable-backup`
- `retention-repository`
- `portable-retention`
- `maintain-repository`
- `acceptance-test`
- `resolve-snapshot`

Group orchestration must consume their results rather than reproduce their
backup logic.

## 14. Initial scope

The first multi-application release will provide:

- group configuration
- configuration validation
- sequential group backup
- group-generation identifiers
- canonical JSON manifests
- explicit PASS/FAIL aggregation
- exact immutable member recovery coordinates
- regression coverage
- installation support for group configurations

Later work will add:

- automated application restore
- group restore validation
- guarded production group restore
- group scheduling
- dependency/quiesce hooks
