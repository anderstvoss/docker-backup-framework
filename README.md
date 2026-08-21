# Docker Backup Framework

Source-controlled backup and recovery framework for Docker application stacks.

The framework provides:

- coordinated Restic application backups
- conventional portable recovery sets
- retention policy enforcement
- Restic repository maintenance
- declarative systemd scheduling
- storage-source verification
- cross-mechanism application capture locking
- runtime/source separation
- documented onboarding and restore procedures

## Documentation

Primary documentation:

- `docs/CONFIGURATION.md` — configuration contract and locking model
- `docs/ONBOARDING.md` — procedure for adding an application
- `docs/RESTORE.md` — non-destructive validation and production recovery
- `docs/VALIDATION.md` — implementation and validation history

## Source of truth

This Git repository is the canonical development and release source.

The repository is not required for production execution after deployment.

Production runtime locations:

- `/usr/local/lib/docker-backup/`
- `/etc/docker-backup/apps/`
- `/etc/docker-backup/repos/`
- `/etc/docker-backup/portable/`
- `/etc/docker-backup/secrets/`
- `/etc/docker-backup/DEPLOYED_VERSION`

Runtime secrets are never stored in Git.

## Framework components

Backup engines:

- `src/backup-repository`
- `src/portable-backup`

Retention engines:

- `src/retention-repository`
- `src/portable-retention`

Repository maintenance:

- `src/maintain-repository`

Deployment:

- `scripts/install`
- `scripts/render-timers`
- `scripts/render-portable-timers`

Systemd service templates are stored under:

- `packaging/systemd/`

## Backup model

Each application defines the filesystem paths that constitute protected
application state.

`BACKUP_PATHS` is a positive inclusion list.

For database-backed applications using logical backup, the live database
storage directory is intentionally excluded from filesystem capture. Database
state is captured separately through a logical dump.

### Restic

A Restic application backup consists of coordinated component snapshots that
share a generation identifier.

Typical components:

- database
- filesystem state

Completed components are marked with `state:complete`.

Retention operates on complete application generations rather than treating
component snapshots independently.

### Portable recovery sets

Portable backups create conventional timestamped recovery directories.

A database-backed recovery set contains:

- `database.sql.gz`
- `app-state.tar.gz`
- `SHA256SUMS`

Recovery sets are finalized atomically after validation.

## Retention

Retention is implemented separately for Restic generations and portable
recovery sets.

Policies support:

- keep-last
- daily
- weekly
- monthly
- yearly

Retention engines operate in dry-run mode by default.

Destructive expiration requires explicit apply mode or the corresponding
validated systemd service.

Restic prune is deliberately separate from snapshot retention.

## Repository maintenance

Restic repository maintenance supports:

- standard repository check
- full data read/check
- prune dry-run
- explicit prune apply

Maintenance operations are serialized with other operations on the same
repository.

## Locking

Capture operations for the same application are serialized using:

`/run/lock/docker-backup-app-${APP_NAME}.lock`

Restic repository operations use:

`/run/lock/docker-backup-${REPO_NAME}.lock`

Portable backup and retention operations use:

`/run/lock/docker-portable-${PORTABLE_NAME}.lock`

Backup engines acquire the application capture lock before their
target-specific lock.

This prevents portable and Restic captures of the same application from
running concurrently.

## Scheduling

Schedules are declared directly in configuration as systemd `OnCalendar`
expressions.

This allows each application and repository to select independent execution
times without framework code changes.

Timer enablement is controlled declaratively with:

`TIMERS_ENABLED="true"` or `TIMERS_ENABLED="false"`

Generated timers use:

- `Persistent=true`
- `AccuracySec=5m`

## Storage safety

For NFS-backed targets, the framework verifies that the configured storage
mount resolves to the expected remote NFS source before operating on backup
data.

This prevents backup data from being written into an unmounted local
directory.

## Deployment

Standard workflow:

1. Develop and review changes in Git.
2. Validate syntax and static behavior.
3. Commit the logical change.
4. Deploy the committed revision.
5. Validate installed runtime behavior.
6. Record validation results.

Deploy with:

    sudo ./scripts/install

The installer:

- validates source files
- installs runtime engines
- installs application, repository, and portable configurations
- installs systemd service templates
- renders timer instances
- applies timer enablement policy
- verifies installed copies
- writes the deployed Git revision
- verifies runtime independence from the source checkout

## Secrets

Never commit:

- Restic passwords
- application `.env` files
- database credentials
- API credentials
- other deployment secrets

Restic password files live under:

`/etc/docker-backup/secrets/`

Maintain an independent off-host copy of each Restic repository password.

## Adding applications

Follow:

`docs/ONBOARDING.md`

An application is not considered fully protected until both backup mechanisms
have been created and restored successfully in a non-destructive validation.

## Recovery

Follow:

`docs/RESTORE.md`

Restore procedures distinguish between:

- non-destructive recovery validation
- production disaster recovery
- Restic generation recovery
- portable recovery-set restoration

Production recovery should not begin until the selected backup has first been
validated non-destructively where practical.

## Current reference implementation

The framework was validated against a reference application using the configuration patterns represented by the included examples.

The reference deployment validated:

- MariaDB logical backup
- multi-path application-state capture
- Restic coordinated generations
- portable recovery sets
- retention
- repository check/read/prune
- systemd execution
- declarative schedules
- NFS source verification
- runtime/source separation
- cross-mechanism capture locking
- non-destructive database and filesystem restore

See `docs/VALIDATION.md` for detailed validation history.

## Example configuration values

Configuration files included in this repository are examples, not prescribed
deployment paths.

Values such as:

- `example-app`
- `/mnt/backups/...`
- `/mnt/application-data/...`
- `192.0.2.10:/exports/...`

are illustrative placeholders. Replace them with application names, local
mount points, and storage endpoints appropriate for the target environment.

`192.0.2.10` is drawn from an address range reserved for documentation and
does not represent a real deployment host.
