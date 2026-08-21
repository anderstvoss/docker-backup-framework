# Production Restore Contract

This document defines the safety and execution contract for destructive
production recovery in the Docker Backup Framework.

Production restore is separate from non-destructive recovery validation.

A production restore must never discover or select backup state implicitly.

## 1. Recovery source

Production recovery must consume an explicitly selected recovery point that has
already passed framework recovery validation.

For Restic-backed recovery, the selected recovery point must identify:

- application
- repository
- Restic generation
- immutable filesystem snapshot ID
- immutable database snapshot ID when applicable

For portable recovery, the selected recovery point must identify the absolute
completed portable recovery-set path.

Group recovery must use only coordinates recorded in a validated PASS group
manifest.

The current group configuration must not be used to select or reorder recovery
members.

`latest` is forbidden for production recovery.

## 2. Execution modes

Production restore tooling must support two distinct modes.

### Plan / preflight

This is the default and is non-destructive.

It must verify:

- configuration identities
- recovery coordinates
- storage availability
- recovery validation status
- application Compose directory
- required backup paths
- database mode
- required runtime tools
- timers that would need suspension
- production paths that would be modified
- locations to be used for fallback capture
- expected recovery order

Plan mode must not:

- stop containers
- stop or disable timers
- modify production files
- modify production databases
- create a new production backup
- delete data

### Execute

Destructive execution must require both:

- an explicit execution option
- an explicit confirmation value derived from the target application or group

Execution must fail closed when either is absent or incorrect.

## 3. Pre-restore recovery checkpoint

Before production mutation begins, the framework must create a fresh recovery
checkpoint of the current production state whenever production is sufficiently
healthy to do so.

The checkpoint must use the existing backup framework and must record immutable
coordinates.

If checkpoint creation fails, production restore must stop unless an explicit
future emergency override is designed and documented.

The first implementation must not provide an emergency override.

## 4. Scheduling isolation

Before application mutation, all backup activity relevant to the application
must be stopped.

This includes applicable:

- operational backup timer
- repository retention timer
- repository maintenance timer
- portable backup timer
- portable retention timer

The restore engine must verify no conflicting operation holds the application,
repository, or portable-target locks before proceeding.

Scheduling state before restore must be recorded so it can be restored
afterward.

## 5. Application shutdown

The application Compose stack must be stopped before production application
files are modified.

The restore engine must verify that targeted application containers are no
longer running before filesystem mutation.

## 6. Filesystem staging

Restic or portable application state must first be restored into a temporary
staging tree.

The staging tree must be validated using the same expected-path and
excluded-path rules used by recovery validation.

Production paths must not be restored directly from Restic.

## 7. Existing production filesystem fallback

Before replacing a configured production backup path, the existing path must be
preserved in a restore-specific fallback location on the same filesystem when
practical.

The fallback operation should prefer atomic rename when source and fallback are
on the same filesystem.

The restore record must identify all fallback paths.

Existing fallback material must not be deleted automatically during the first
implementation.

## 8. Filesystem placement

Only configured application `BACKUP_PATHS` may be restored.

Configured excluded paths must not be introduced.

Logical database storage excluded from application backup must not be restored
as ordinary filesystem state.

Ownership and permissions from the validated recovery content must be
preserved.

## 9. Database recovery

For `DB_TYPE=none`, no database mutation occurs.

For `mariadb:logical`:

- raw MariaDB data directories must never be restored
- only the database service should be started when practical
- authenticated readiness must be verified
- the target database must be recreated as required
- the immutable validated SQL dump must be imported
- import success must be verified before application startup

The production restore must use the same database component represented by the
selected recovery point.

## 10. Application restart and validation

After filesystem and database recovery:

- start the application stack
- verify expected containers are running
- verify configured health checks when available
- verify application/database connectivity where practical
- inspect startup state for immediate recovery failure

Only after application validation passes may scheduling be restored.

## 11. Scheduling restoration

Timers must be returned to their pre-restore enabled/running state.

The restore record must identify whether scheduling restoration succeeded.

## 12. Restore record

Every attempted production restore must write a durable record containing at
least:

- result
- application
- selected recovery coordinates
- restore start timestamp
- restore completion/failure timestamp
- pre-restore checkpoint coordinates
- timers suspended
- application shutdown result
- staged restore result
- fallback paths
- filesystem placement result
- database restore result
- application startup result
- application validation result
- timer restoration result
- last completed stage
- failure stage when applicable

Restore records must contain no secrets.

## 13. Failure behavior

Production restore is fail-fast.

If failure occurs before production mutation, production remains untouched.

If failure occurs after mutation begins:

- do not silently continue
- preserve staging and fallback information required for recovery
- record the exact failure stage
- do not automatically delete fallback state
- do not automatically claim rollback success

Automated rollback may be added only after independent rollback procedures have
been implemented and tested.

## 14. Group production recovery

Group production recovery must:

1. validate the selected PASS group manifest
2. validate all member recovery points non-destructively
3. create pre-restore checkpoints
4. process production recovery in reverse manifest order
5. stop immediately on the first member failure
6. never substitute current group membership or order
7. write a group-level recovery record referencing each member restore record

A partially restored group must not be reported as a successful group recovery.

## 15. Initial implementation boundary

The first production implementation will support:

- `DB_TYPE=none`
- `mariadb:logical`
- Restic filesystem recovery
- immutable Restic coordinates
- validated portable set as an independent recovery reference
- explicit fallback preservation
- guarded execution

The first implementation will not provide:

- automatic rollback
- emergency checkpoint bypass
- PostgreSQL recovery
- dependency graph recovery
- parallel group recovery
- automatic deletion of fallback data
