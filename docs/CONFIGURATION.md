# Configuration Contract

The Docker Backup Framework separates configuration into four classes:

- application configuration
- Restic repository configuration
- portable backup configuration
- multi-application group configuration

## Application configuration

Location in source:

`config/apps/<app>.conf`

Installed location:

`/etc/docker-backup/apps/<app>.conf`

Required fields:

- `APP_NAME`
- `COMPOSE_DIR`
- `DB_TYPE`
- `DB_BACKUP_MODE`
- `DB_CONTAINER`
- `BACKUP_PATHS`

`BACKUP_PATHS` is the authoritative inclusion list for application-state
capture.

Only paths explicitly listed in `BACKUP_PATHS` are included in filesystem
backup archives/snapshots.

`EXCLUDE_PATHS` documents paths which must not be captured, but it is not a
substitute for positive inclusion. A path omitted from `BACKUP_PATHS` is not
backed up even if it is absent from `EXCLUDE_PATHS`.

Typical exclusions include:

- live database storage
- large media libraries protected separately
- caches or regenerable data

For database-backed applications, the live database data directory must not be
included when `DB_BACKUP_MODE="logical"` is used. Database state is captured
through the database dump mechanism instead.

### Optional semantic database validation

MariaDB logical recovery validation always compares generic restored structure:

- table count
- trigger count

Applications may additionally configure one semantic schema or migration
marker:

    DB_VALIDATION_LABEL="migration"
    DB_VALIDATION_QUERY="SELECT migration FROM migrations ORDER BY id DESC LIMIT 1;"

Both fields must be configured together.

`DB_VALIDATION_QUERY` must return exactly one row and one column.

Examples include:

- Alembic migration version
- Laravel migration identity
- application schema version

The framework does not assume a particular application migration system.

## Restic repository configuration

Location in source:

`config/repos/<repository>.conf`

Installed location:

`/etc/docker-backup/repos/<repository>.conf`

A repository configuration declares:

- repository identity
- application identity
- Restic repository path
- runtime password-file path
- retention policy
- maintenance schedules
- storage verification information
- timer enablement state

Schedule values are complete systemd `OnCalendar` expressions.

Execution times are therefore selected per repository and are not hard-coded
in framework code.

Secrets are runtime-only and must not be committed to Git.

## Portable backup configuration

Location in source:

`config/portable/<target>.conf`

Installed location:

`/etc/docker-backup/portable/<target>.conf`

A portable configuration declares:

- portable target identity
- application identity
- conventional backup root
- storage verification information
- backup schedule
- retention schedule
- retention policy
- timer enablement state

Schedule values are complete systemd `OnCalendar` expressions.

## Multi-application group configuration

Location in source:

`config/groups/<group>.conf`

Installed location:

`/etc/docker-backup/groups/<group>.conf`

A group configuration declares:

- group identity
- ordered application membership
- the repository configuration used by each member
- the portable backup configuration used by each member

Groups are orchestration metadata only.

Each application retains its independent repository, credentials, storage
verification, portable target, retention, and scheduling configuration.

Group backups are coordinated and sequential but are not globally atomic.

See `docs/GROUPS.md` for the complete contract.

## Locking model

Capture operations for the same application are serialized with:

`/run/lock/docker-backup-app-${APP_NAME}.lock`

Restic repository operations are serialized with:

`/run/lock/docker-backup-${REPO_NAME}.lock`

Portable backup and retention operations are serialized with:

`/run/lock/docker-portable-${PORTABLE_NAME}.lock`

Backup engines acquire the application capture lock before their
target-specific lock.

Retention and maintenance operations do not acquire the application capture
lock.

## Storage verification

For NFS-backed targets, configuration declares:

- `STORAGE_TYPE="nfs"`
- `STORAGE_MOUNT`
- `EXPECTED_SOURCE`

Backup and maintenance engines verify that the configured mount resolves to the
expected NFS source before operating on backup data.

This prevents accidental writes to an unmounted local directory.
