# Application Onboarding

This procedure adds a Docker application to the backup framework.

Do not enable scheduled backup execution until the application's backup
contents and restore procedure have been validated manually.

## 1. Inventory the application

Before creating framework configuration, identify:

- Docker Compose directory
- application state directories
- database engine
- database container
- live database storage directory
- large media or bulk-data directories
- caches and regenerable data
- external mounts
- backup storage destination

Classify every persistent path as either:

- application state that must be backed up
- database state captured logically
- bulk data protected separately
- regenerable data that does not require backup

Do not proceed until the persistent-state model is understood.

## 2. Create application configuration

Create:

`config/apps/<app>.conf`

Example structure:

    APP_NAME="<app>"

    COMPOSE_DIR="/opt/docker/<app>"

    DB_TYPE="mariadb"
    DB_BACKUP_MODE="logical"
    DB_CONTAINER="<database-container>"

    BACKUP_PATHS=(
      "/opt/docker/<app>"
      "/srv/docker/<app>/config"
    )

    EXCLUDE_PATHS=(
      "/srv/docker/<app>/db"
    )

`BACKUP_PATHS` is authoritative.

Only explicitly included paths are captured.

For logical database backups, never include the live database data directory
in `BACKUP_PATHS`.

Use `EXCLUDE_PATHS` to document paths intentionally outside the backup set.

## 3. Create backup storage

Provision the application's backup destination before enabling the framework.

For isolated NFS-backed application storage, establish as applicable:

- dedicated backup dataset or directory
- dedicated service identity and ownership
- restricted read/write NFS export
- authorized Docker-host address
- expected NFS source
- stable local mount path
- Restic repository location
- portable recovery-set location

Configure the Docker host mount before creating backup artifacts.

Verify both:

- the local path resolves to the expected NFS source
- a test write reaches the remote storage with the intended ownership

Do not treat a successful write to the local mount directory as proof that NFS
is active. Verify the mounted filesystem explicitly with `findmnt`,
`mountpoint`, or equivalent.

The framework verifies the configured source during normal operations, but
storage provisioning itself remains an onboarding prerequisite.

## 4. Create Restic repository configuration

Create:

`config/repos/<repository>.conf`

Example:

    REPO_NAME="<repository>"
    APP_NAME="<app>"

    REPOSITORY="<restic-repository-path>"
    PASSWORD_FILE="/etc/docker-backup/secrets/<repository>.password"

    BACKUP_SCHEDULE="<systemd OnCalendar expression>"
    RETENTION_SCHEDULE="<systemd OnCalendar expression>"
    TIMERS_ENABLED="false"

    KEEP_LAST=3
    KEEP_DAILY=7
    KEEP_WEEKLY=4
    KEEP_MONTHLY=3
    KEEP_YEARLY=0

    CHECK_SCHEDULE="<systemd OnCalendar expression>"
    READ_DATA_CHECK_SCHEDULE="<systemd OnCalendar expression>"
    PRUNE_SCHEDULE="<systemd OnCalendar expression>"

    STORAGE_TYPE="nfs"
    STORAGE_MOUNT="<local-mount>"
    EXPECTED_SOURCE="<expected-nfs-source>"

Initially keep `TIMERS_ENABLED="false"`.

Choose execution times that avoid unnecessary overlap with existing backup
jobs.

## 5. Create the Restic password

Create the password file only under:

`/etc/docker-backup/secrets/`

Never commit repository passwords to Git.

Maintain an independent off-host copy of each Restic password.

Loss of the repository password makes the encrypted Restic repository
unrecoverable.

## 6. Initialize the Restic repository

Create the repository parent directory if required.

Initialize the configured repository using the same repository path and
password file declared in its configuration.

For example:

    mkdir -p "$(dirname "<repository-path>")"

    restic \
      --repo "<repository-path>" \
      --password-file "<password-file>" \
      init

Verify that the repository opens successfully before running the framework
backup engine.

`backup-repository` does not initialize a missing Restic repository
automatically.

## 7. Create portable backup configuration

Create:

`config/portable/<target>.conf`

Example:

    PORTABLE_NAME="<target>"
    APP_NAME="<app>"

    BACKUP_ROOT="<portable-backup-root>"

    STORAGE_TYPE="nfs"
    STORAGE_MOUNT="<local-mount>"
    EXPECTED_SOURCE="<expected-nfs-source>"

    PORTABLE_SCHEDULE="<systemd OnCalendar expression>"
    PORTABLE_RETENTION_SCHEDULE="<systemd OnCalendar expression>"
    TIMERS_ENABLED="false"

    KEEP_LAST=3
    KEEP_DAILY=7
    KEEP_WEEKLY=4
    KEEP_MONTHLY=3
    KEEP_YEARLY=0

Keep timers disabled during initial validation.

Create `BACKUP_ROOT` before the first portable backup:

    mkdir -p "<portable-backup-root>"

Verify that the directory is on the intended backup storage and is writable.

`portable-backup` intentionally fails if `BACKUP_ROOT` does not already exist;
it does not create the root automatically.

## 8. Validate source configuration

Before committing:

    bash -n src/backup-repository
    bash -n src/retention-repository
    bash -n src/maintain-repository
    bash -n src/portable-backup
    bash -n src/portable-retention
    bash -n scripts/install
    bash -n scripts/render-timers
    bash -n scripts/render-portable-timers

    git diff --check

Validate each configured schedule with:

    systemd-analyze calendar '<expression>'

## 9. Validate timer rendering in isolation

Render repository timers into a temporary directory:

    sudo rm -rf /tmp/docker-backup-timer-test
    sudo mkdir -p /tmp/docker-backup-timer-test

    sudo env \
      TIMER_DEST=/tmp/docker-backup-timer-test \
      ./scripts/render-timers \
      ./config/repos/<repository>.conf

Render portable timers similarly:

    sudo rm -rf /tmp/docker-portable-timer-test
    sudo mkdir -p /tmp/docker-portable-timer-test

    sudo env \
      TIMER_DEST=/tmp/docker-portable-timer-test \
      ./scripts/render-portable-timers \
      ./config/portable/<target>.conf

Run `systemd-analyze verify` against every generated timer.

Do not enable live timers yet.

## 10. Commit and deploy disabled configuration

Commit the new application, repository, and portable configuration.

Deploy:

    sudo ./scripts/install

Verify:

- installed configuration matches source
- runtime engines match source
- configured timers remain disabled
- storage-source verification succeeds
- Restic repository opens successfully

## 11. Create a manual Restic backup

Run:

    sudo /usr/local/lib/docker-backup/backup-repository \
      /etc/docker-backup/repos/<repository>.conf

Verify that one complete generation exists and contains all expected
components.

Confirm intentionally excluded live database and bulk-data paths were not
captured.

## 12. Create a manual portable backup

Run:

    sudo /usr/local/lib/docker-backup/portable-backup \
      /etc/docker-backup/portable/<target>.conf

A completed database-backed recovery set should contain:

- `database.sql.gz`
- `app-state.tar.gz`
- `SHA256SUMS`

Validate the recovery set:

    cd <recovery-set-directory>

    sha256sum -c SHA256SUMS
    gzip -t database.sql.gz
    tar -tzf app-state.tar.gz >/dev/null

Confirm the archive contains all expected application-state paths and does not
contain intentionally excluded paths.

## 13. Perform a non-destructive restore test

Do not consider an application onboarded until both backup mechanisms have
been restored into disposable locations.

At minimum validate:

- filesystem extraction
- expected files and directories
- absence of intentionally excluded paths
- database dump import into an isolated database instance
- basic restored database structure

Do not overwrite the production application during initial restore
validation.

Record application-specific restore validation results in
`docs/VALIDATION.md`.

## 14. Validate retention

Run both retention engines in dry-run mode first:

    sudo /usr/local/lib/docker-backup/retention-repository \
      /etc/docker-backup/repos/<repository>.conf

    sudo /usr/local/lib/docker-backup/portable-retention \
      /etc/docker-backup/portable/<target>.conf

Review every keep/expire decision before permitting destructive retention.

Repository pruning remains a separate maintenance operation from retention.

## 15. Enable scheduling

Only after backup and restore validation:

1. Set `TIMERS_ENABLED="true"` in the repository configuration.
2. Set `TIMERS_ENABLED="true"` in the portable configuration.
3. Commit the change.
4. Deploy the committed version.

Verify:

    systemctl list-timers 'docker-*.timer' --all

Confirm:

- all intended timer instances are enabled
- all intended timer instances are active
- execution times match configuration
- application jobs do not unintentionally collide with existing backup jobs

The application capture lock prevents Restic and portable capture of the same
application from executing concurrently, but schedules should still be
staggered to avoid unnecessary resource contention.

## 16. Record the validated deployment

Document:

- application name
- protected paths
- intentionally excluded paths
- database backup method
- repository location
- portable backup location
- retention policy
- timer schedule
- successful Restic restore validation
- successful portable restore validation
- deployed framework revision

## 17. Final acceptance criteria

Application onboarding is complete only when:

- application persistence has been inventoried
- backup inclusion and exclusion decisions are documented
- Restic repository is initialized and accessible
- Restic password is preserved outside the Docker host
- manual Restic backup succeeds
- manual portable backup succeeds
- Restic recovery is validated non-destructively
- portable recovery is validated non-destructively
- retention dry-runs produce expected results
- timer rendering passes validation
- deployed configuration matches source
- scheduled timers are enabled and active
- validation history has been recorded

The application should not be considered fully protected until a restore has
been demonstrated.
