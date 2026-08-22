# Restore Runbook

This document defines restore procedures for the Docker Backup Framework.

Two backup mechanisms are supported:

- Restic application generations
- portable conventional recovery sets

Restore validation should be performed non-destructively before any production
recovery is attempted.

## 1. General restore principles

Before restoring:

- identify the application being recovered
- identify the intended backup generation or recovery set
- verify backup storage is mounted from the expected source
- preserve the current production state if it still exists
- avoid overwriting production data during initial validation
- validate filesystem and database recovery independently

For database-backed applications using logical backup, live database files are
not restored directly.

The database is recreated from a logical dump.

## 2. Non-destructive restore validation

A non-destructive restore should use temporary filesystem locations and an
isolated disposable database instance.

The validation should prove:

- backup data is readable
- expected files and directories are present
- intentionally excluded paths are absent
- database dump imports successfully
- basic database structure is intact

Do not use production application paths during validation.

## 3. Restic restore overview

A complete Restic application generation consists of coordinated component
snapshots sharing the same generation tag.

Typical components:

- database dump
- filesystem state

Restore both components from the same complete generation.

Do not mix components from different generations unless performing a deliberate
recovery investigation.

## 4. Identify complete Restic generations

Repository configuration is installed `root:root 0640`, so perform Restic
repository operations through a root shell that loads the configuration.

List snapshots:

    sudo bash -c '
    source /etc/docker-backup/repos/<repository>.conf
    export RESTIC_REPOSITORY="$REPOSITORY"
    export RESTIC_PASSWORD_FILE="$PASSWORD_FILE"

    restic snapshots
    '

Inspect tags and identify a complete application generation containing all
expected components.

The framework uses tags describing:

- application
- repository
- component
- generation
- state

Select only snapshots representing a complete generation.

### Exact snapshot selection

For validation and production recovery, resolve the intended snapshot from
`restic snapshots --json` using the complete required tag set and restore by
the resulting immutable snapshot ID.

Do not rely on `latest` combined with several repeated `--tag` arguments to
prove that all desired tags identify the same snapshot. During validation this
can select the wrong component while still producing a syntactically valid
restore.

Required tags normally include:

    app:<application>
    repo:<repository>
    generation:<generation>
    component:<db|files>
    state:complete

Example exact resolver:

    SNAPSHOT_ID="$(
      restic snapshots --json |
      python3 -c '
    import json
    import sys

    required = set(sys.argv[1:])
    matches = []

    for snapshot in json.load(sys.stdin):
        tags = set(snapshot.get("tags") or [])
        if required.issubset(tags):
            matches.append(snapshot["id"])

    if len(matches) != 1:
        raise SystemExit(
            f"expected exactly one matching snapshot; found {len(matches)}"
        )

    print(matches[0])
    ' \
        "app:<application>" \
        "repo:<repository>" \
        "generation:<generation>" \
        "component:<component>" \
        "state:complete"
    )"

The resolver must return exactly one snapshot. Record the full snapshot ID and
use that ID explicitly in `restic restore`.

## 5. Restore the Restic filesystem component

Create a temporary destination:

    sudo rm -rf /tmp/docker-restore-files
    sudo mkdir -p /tmp/docker-restore-files

Restore the exact filesystem snapshot ID resolved from the complete tag set:

    sudo bash -c '
    source /etc/docker-backup/repos/<repository>.conf
    export RESTIC_REPOSITORY="$REPOSITORY"
    export RESTIC_PASSWORD_FILE="$PASSWORD_FILE"

    restic restore <filesystem-snapshot-id> \
      --target /tmp/docker-restore-files
    '

Inspect the restored tree:

    sudo find /tmp/docker-restore-files \
      -maxdepth 4 \
      -printf '%p\n' \
      | sort

Validate all configured application `BACKUP_PATHS`.

Confirm intentionally excluded paths are absent.

## 6. Restore the Restic database component

The Restic database component is stored separately from the filesystem
component.

For logical MariaDB backup, the framework streams `mariadb-dump` directly into
Restic using:

`--stdin-filename database.sql`

The restored database component is therefore plain SQL at:

`/database.sql`

Create a temporary restore destination:

    sudo rm -rf /tmp/docker-restore-db
    sudo mkdir -p /tmp/docker-restore-db

Restore the exact database snapshot ID resolved from the complete tag set:

    sudo bash -c '
    source /etc/docker-backup/repos/<repository>.conf
    export RESTIC_REPOSITORY="$REPOSITORY"
    export RESTIC_PASSWORD_FILE="$PASSWORD_FILE"

    restic restore <database-snapshot-id> \
      --target /tmp/docker-restore-db
    '

Verify the expected file exists and is non-empty:

    sudo test -s /tmp/docker-restore-db/database.sql

Inspect the file type if needed:

    sudo file /tmp/docker-restore-db/database.sql

Do not expect gzip compression for the Restic database component.

Do not import into the production database during validation.

## 7. Validate a MariaDB logical restore

For MariaDB applications, use a disposable container.

Example:

    docker run -d \
      --name docker-backup-restore-db \
      --tmpfs /var/lib/mysql \
      -e MARIADB_ROOT_PASSWORD=restore-test \
      mariadb:11

Wait for authenticated SQL access:

    until docker exec docker-backup-restore-db \
      mariadb \
      -uroot \
      -prestore-test \
      -e 'SELECT 1' \
      >/dev/null 2>&1
    do
      sleep 2
    done

Create a temporary database:

    docker exec docker-backup-restore-db \
      mariadb \
      -uroot \
      -prestore-test \
      -e 'CREATE DATABASE restore_test;'

Import the Restic database dump directly:

    sudo cat /tmp/docker-restore-db/database.sql \
      | docker exec -i docker-backup-restore-db \
          mariadb \
          -uroot \
          -prestore-test \
          restore_test

Validate basic database structure.

Examples:

    docker exec docker-backup-restore-db \
      mariadb \
      -uroot \
      -prestore-test \
      -e 'SHOW TABLES FROM restore_test;'

    docker exec docker-backup-restore-db \
      mariadb \
      -uroot \
      -prestore-test \
      -e '
        SELECT TRIGGER_NAME
        FROM information_schema.TRIGGERS
        WHERE TRIGGER_SCHEMA="restore_test";
      '

Remove the disposable database after validation:

    docker rm -f docker-backup-restore-db

### PostgreSQL logical restore validation

For PostgreSQL applications, use a disposable container based on the same
major PostgreSQL image family as the application database.

Example:

    docker run -d \
      --name docker-backup-restore-postgresql \
      --tmpfs /var/lib/postgresql \
      -e POSTGRES_USER=restore_test \
      -e POSTGRES_PASSWORD=restore-test \
      -e POSTGRES_DB=restore_test \
      postgres:18

Wait for authenticated SQL readiness:

    until docker exec docker-backup-restore-postgresql \
      psql \
      --username=restore_test \
      --dbname=restore_test \
      --no-align \
      --tuples-only \
      --command='SELECT 1;' \
      2>/dev/null | grep -Fxq 1
    do
      sleep 1
    done

Import the Restic database dump:

    sudo cat /tmp/docker-restore-db/database.sql \
      | docker exec -i docker-backup-restore-postgresql \
          psql \
          --username=restore_test \
          --dbname=restore_test \
          --set=ON_ERROR_STOP=1

Validation should compare at least:

- base-table count
- view count
- trigger count
- the configured semantic validation marker, when present

PostgreSQL logical dumps are created without restored object ownership or ACLs,
allowing isolated validation without recreating the production role ownership
and grants.

Remove the disposable database when validation is complete:

    docker rm -f docker-backup-restore-postgresql

## 8. Portable recovery-set overview

A completed database-backed portable recovery set contains:

- `app-state.tar.gz`
- `database.sql.gz`
- `SHA256SUMS`

Recovery sets are timestamped directories under the configured `BACKUP_ROOT`.

Only completed timestamp directories should be used.

Do not use `.incomplete-*` directories.

## 9. Select and verify a portable recovery set

Choose a recovery set:

    RECOVERY_SET="<path-to-recovery-set>"

Enter the directory:

    cd "$RECOVERY_SET"

Verify checksums:

    sha256sum -c SHA256SUMS

Validate the database archive:

    gzip -t database.sql.gz

Validate the application archive:

    tar -tzf app-state.tar.gz >/dev/null

Do not continue if any validation fails.

## 10. Restore portable application state non-destructively

Create a temporary destination:

    sudo rm -rf /tmp/docker-portable-restore
    sudo mkdir -p /tmp/docker-portable-restore

Extract:

    sudo tar \
      -C /tmp/docker-portable-restore \
      -xzf "$RECOVERY_SET/app-state.tar.gz"

Inspect:

    sudo find /tmp/docker-portable-restore \
      -maxdepth 5 \
      -printf '%p\n' \
      | sort

Confirm:

- all required application-state paths are present
- live database storage is absent
- separately protected bulk-data paths are absent
- files can be read successfully

## 11. Restore portable database non-destructively

Use the same isolated database procedure described for Restic.

Import:

    gzip -dc "$RECOVERY_SET/database.sql.gz" \
      | docker exec -i docker-backup-restore-db \
          mariadb \
          -uroot \
          -prestore-test \
          restore_test

Validate expected database structure before declaring the recovery set usable.

## 12. Production recovery preparation

Production restore is destructive and should only follow successful
non-destructive validation.

Before modifying production:

- identify the exact backup being restored
- record the current application state
- stop scheduled backup jobs for the application
- stop the application stack
- preserve the current filesystem state if possible
- preserve the current database state if possible
- verify the intended restore source again

Do not delete the existing production state until a recoverable fallback exists
unless the existing state is already unusable.

## 13. Disable scheduled backup activity during recovery

Disable or stop the relevant timer instances before production recovery.

Examples:

    sudo systemctl stop docker-backup@<repository>.timer
    sudo systemctl stop docker-retention@<repository>.timer
    sudo systemctl stop docker-portable-backup@<target>.timer
    sudo systemctl stop docker-portable-retention@<target>.timer

Also stop repository maintenance timers if they could overlap the recovery.

Verify no backup operation is currently active before proceeding.

## 14. Stop the application stack

Change to the application's Compose directory:

    cd /opt/docker/<app>

Stop the application:

    docker compose down

Confirm application and database containers are stopped.

## 15. Restore production application files

Restore only paths represented by the application's configured
`BACKUP_PATHS`.

Do not restore logical database content into the live database data directory.

When using a portable archive, extract first into temporary staging and inspect
the result before copying files into production.

Preserve ownership and permissions.

After file placement, compare ownership and mode against the application
deployment requirements.

## 16. Restore the production database

Start only the database service if practical.

Wait until authenticated SQL access succeeds.

Create or recreate the intended application database as required by the
application.

Import the validated logical dump.

Do not restore raw database storage from filesystem backup when the application
uses `DB_BACKUP_MODE="logical"`.

Validate:

- tables exist
- expected schema markers are present
- triggers or other database objects exist where applicable

## 17. Restore separately protected bulk data

Bulk-data paths intentionally excluded from the framework must be recovered
from their independent protection mechanism.

Examples include:

- media libraries
- ROM libraries
- large content datasets

The application backup framework does not reconstruct data that was
intentionally excluded from `BACKUP_PATHS`.

## 18. Start and validate the application

Start the application stack:

    docker compose up -d

Validate:

- all expected containers are running
- application health checks pass
- application UI or API responds
- database connectivity succeeds
- restored configuration is loaded
- required external mounts are available
- separately restored bulk data is visible

Review logs for restore-related errors.

## 19. Re-enable scheduling

After successful production validation:

    sudo systemctl start docker-backup@<repository>.timer
    sudo systemctl start docker-retention@<repository>.timer
    sudo systemctl start docker-portable-backup@<target>.timer
    sudo systemctl start docker-portable-retention@<target>.timer

Restart repository maintenance timers as appropriate.

Verify:

    systemctl list-timers 'docker-*.timer' --all

## 20. Post-restore validation

After production recovery, record:

- application restored
- selected Restic generation or portable recovery set
- restore timestamp
- filesystem validation result
- database validation result
- application health result
- excluded bulk-data recovery status
- timers re-enabled
- any deviations from the standard procedure

## 21. Recovery acceptance criteria

Recovery is complete only when:

- backup integrity was validated before restore
- filesystem state was restored successfully
- database import completed successfully
- expected database structure is present
- intentionally excluded data was handled separately
- application starts successfully
- application health checks pass
- scheduled backup activity is restored
- recovery details are documented
