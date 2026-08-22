# Validation History

## Backup engine

Validated:

- MariaDB logical dump streamed directly to Restic
- filesystem component backup
- coordinated generation tagging
- `state:building` -> `state:complete`
- exact tag-set matching
- source-path preflight validation
- interrupted-generation cleanup
- disposable database restore
- disposable filesystem restore

Reference application restore validation:

- 32 database tables
- 4 database triggers

## Retention engine

Validated policy:

- KEEP_LAST=3
- KEEP_DAILY=7
- KEEP_WEEKLY=4
- KEEP_MONTHLY=3
- KEEP_YEARLY=0

Synthetic test history:

- 18 complete generations
- 36 component snapshots

Expected policy result:

- 10 retained generations
- 8 expired generations
- 0 invalid generations

Destructive validation:

- exactly 8 expired generations selected
- exactly 16 component snapshots forgotten
- 10 retained generations remained
- reference-application snapshots were unaffected
- no prune was performed

Synthetic snapshots were subsequently removed.

## Safety properties

Retention only manages snapshots matching the configured:

- application tag
- repository tag
- `state:complete`
- generation tag
- expected component set

Malformed generations block destructive retention.

Legacy or unmanaged snapshots are reported but not automatically removed.

## Runtime/source separation

Validated after commit `04fb709`:

- source checkout `/opt/docker-backup-framework` was temporarily removed
- deployed engines remained available under `/usr/local/lib/docker-backup`
- deployed app/repository configs remained available under `/etc/docker-backup`
- both deployed engines passed shell syntax validation
- reference-application retention dry-run completed successfully with the source checkout absent
- Restic repository access and runtime secret references worked
- `/etc/docker-backup/DEPLOYED_VERSION` remained available
- deployed version matched Git commit `04fb70902e3a7fb17baf9ff1139ab8647cfb4ed5`

Conclusion: the Docker VM runtime has no dependency on the colocated Git checkout.
The checkout may later be moved off-host without changing runtime operation.

## Repository maintenance engine

Validated against the reference application operational repository.

### Standard check

- NFS source verification passed
- repository opened successfully
- 42 / 42 index files loaded
- 4 / 4 snapshots checked
- no errors found

### Full data check

- 42 / 42 index files loaded
- 4 / 4 snapshots checked
- 83 / 83 packs read
- no errors found

### Prune dry-run

Prune planning completed without modifying the repository.

Observed plan:

- 0 blobs / 0 B to repack
- 77 blobs / 87.604 KiB to delete
- 87.604 KiB total reclaimable
- 84 blobs / 8.232 MiB remaining
- 0 B unused after prune

No actual prune has been performed yet.

### Actual prune validation

Actual prune was executed against the reference application operational repository.

Observed result:

- 77 blobs / 87.604 KiB deleted
- 0 blobs repacked
- 84 blobs / 8.232 MiB remained
- repository index rebuilt successfully
- all 4 / 4 snapshots remained valid

Post-prune validation:

- `restic check` completed with no errors
- post-prune repository contained 1 index
- prune dry-run reported 0 blobs / 0 B reclaimable
- unused size after prune was 0 B

This validates the explicit `prune --apply` maintenance path.

## systemd service execution

Validated generic systemd service templates after commit `8158731`.

Validated:

- all five service templates passed `systemd-analyze verify` as root
- templates installed under `/etc/systemd/system`
- deployed framework version matched Git commit
- `docker-check@example-app-operational.service` executed successfully
- service ran as `Type=oneshot`
- service exited cleanly to `inactive (dead)`
- reference application repository check completed successfully
- all 4 / 4 snapshots validated
- no repository errors were found

SSH disconnect observed immediately after the test was investigated separately.
Host uptime and boot ID remained continuous, and OpenSSH logged
`disconnected by user`, indicating the client initiated the disconnect.
No framework or service fault was identified.

This validates the systemd execution layer independently of timers.

## systemd timer scheduling

Validated reference application operational repository timers after commit `12a1a75`.

Configured schedules:

- backup: daily at 02:00 local time
- retention: daily at 03:00 local time
- repository check: Sunday at 04:00 local time
- full repository data check: first day of each month at 04:00 local time
- repository prune: second day of each month at 04:00 local time

Validated:

- all generated timer units passed `systemd-analyze verify`
- all calendar expressions normalized correctly
- all five timers installed under `/etc/systemd/system`
- all five timers enabled successfully
- all five timers are active
- enabling the timers did not trigger any corresponding service
- all service execution timestamps remained empty immediately after enablement
- `Persistent=true` is enabled for all timers

Initial next-run schedule:

- backup: 2026-08-21 02:00 PDT
- retention: 2026-08-21 03:00 PDT
- check: 2026-08-23 04:00 PDT
- read-data: 2026-09-01 04:00 PDT
- prune: 2026-09-02 04:00 PDT

## Declarative timer enablement

Validated after commit `f77215f`.

Repository configuration now declares:

- `BACKUP_SCHEDULE="daily"`
- `RETENTION_SCHEDULE="daily"`
- `CHECK_SCHEDULE="weekly"`
- `READ_DATA_CHECK_SCHEDULE="monthly"`
- `PRUNE_SCHEDULE="monthly"`
- `TIMERS_ENABLED="true"`

Validated deployment behavior:

- installer rendered all five concrete reference-application timer instances
- renderer applied timer enablement automatically
- all five timers were enabled
- all five timers were active
- deployed repository configuration matched source
- deployed framework version matched Git commit
- no manual `systemctl enable --now` step was required

Active schedule:

- backup: daily 02:00 local time
- retention: daily 03:00 local time
- repository check: Sunday 04:00 local time
- full repository read: first day of month 04:00 local time
- repository prune: second day of month 04:00 local time

This validates timer configuration and enablement as reproducible deployment state.

## Portable backup engine

Validated generic portable backup implementation against the reference application.

Recovery-set format:

- `database.sql.gz`
- `app-state.tar.gz`
- `SHA256SUMS`

Validated:

- TrueNAS NFS backing storage verified before backup
- all configured `BACKUP_PATHS` existed before capture
- MariaDB logical dump completed successfully
- database gzip stream validated
- application-state tar archive validated
- SHA-256 manifest validated
- live MariaDB data directory was not included
- ROM library was not included
- recovery set finalized atomically
- no `.incomplete-*` recovery set remained

Non-destructive restore validation:

- application-state archive extracted successfully into temporary storage
- all required application paths restored
- forbidden database and ROM-library paths remained absent
- database dump imported successfully into an isolated temporary MariaDB container
- restored database contained 32 tables
- restored database contained 4 triggers
- the application's configured Alembic migration marker was readable
- restored schema marker was `0103_roms_facets_provider_ids`

The Alembic marker above is specific to the reference application. Generic
MariaDB validation does not require Alembic; applications may define their own
single-value semantic validation query.

The initial restore harness used `mariadb-admin ping`, which could succeed
before authenticated SQL access was ready. Validation was repeated using an
authenticated `SELECT 1` readiness check and completed successfully.

## Portable runtime deployment

Validated after commit `21d2ce5`.

Validated:

- `portable-backup` installed under `/usr/local/lib/docker-backup/`
- portable reference-application policy installed under `/etc/docker-backup/portable/`
- installed engine hash matched source
- installed portable configuration hash matched source
- deployed framework version matched Git commit
- runtime contained no references to the source checkout

Runtime-independence validation:

- `/opt/docker-backup-framework` was temporarily moved offline
- installed `portable-backup` executed using only deployed runtime configuration
- backup completed successfully
- completed recovery set `2026-08-20_203206Z` was created
- no `.incomplete-*` directories remained

This validates the portable backup runtime independently of the temporary Git checkout.

## Portable retention engine

Validated generic portable recovery-set retention.

Policy tested:

- `KEEP_LAST=3`
- `KEEP_DAILY=4`
- `KEEP_WEEKLY=3`
- `KEEP_MONTHLY=2`
- `KEEP_YEARLY=1`

Synthetic fixture:

- 13 valid timestamped recovery sets
- retention union selected 8 sets to keep
- 5 sets selected for expiration

Validated:

- dry-run reported the expected keep/expire selection
- `--apply` removed exactly 5 expired recovery sets
- exactly 8 expected recovery sets remained
- whole recovery-set directories were treated as indivisible generations
- malformed timestamped recovery sets were detected before deletion
- a malformed set missing `SHA256SUMS` and `database.sql.gz` caused exit code 1
- malformed-set validation failure caused no valid recovery-set deletion

This validates both retention bucket selection and destructive-apply safety.

## Portable systemd services

Validated generic portable backup and retention service templates.

Validated:

- both portable service templates passed `systemd-analyze verify`
- portable backup service executed successfully through systemd
- portable backup service returned `Result=success`
- portable backup service returned `ExecMainStatus=0`
- recovery set `2026-08-20_203849Z` was created successfully
- portable retention service executed successfully through systemd
- portable retention service returned `Result=success`
- portable retention service returned `ExecMainStatus=0`
- retention evaluated 3 managed recovery sets
- all 3 recovery sets were retained
- 0 recovery sets were removed
- no `.incomplete-*` directories remained

This validates the portable systemd execution layer independently of timers.

## Portable timer scheduling

Validated portable backup timer rendering after commit `8cd2e06`.

Configured schedules:

- portable backup: daily at 01:00 local time
- portable retention: daily at 01:30 local time

Validated:

- both generated portable timers passed isolated `systemd-analyze verify`
- portable timers installed successfully
- both portable timers enabled successfully
- both portable timers became active
- enabling the timers did not immediately execute either service
- both timers use `Persistent=true`

Combined reference-application protection schedule:

- 01:00 daily: portable backup
- 01:30 daily: portable retention
- 02:00 daily: Restic operational backup
- 03:00 daily: Restic operational retention
- Sunday 04:00: Restic repository check
- first day of month 04:00: Restic full-data check
- second day of month 04:00: Restic prune

This validates the complete automated reference-application backup schedule.

## Legacy reference-application backup retirement

The original application-specific portable backup implementation was removed after the
generalized framework was fully validated.

Removed:

- `/usr/local/sbin/backup-example-app`
- `/etc/systemd/system/example-app-backup.service`

Validated after removal:

- legacy `example-app-backup.service` no longer exists
- all 7 generalized reference-application backup timers remained active
- all 3 portable recovery sets remained intact
- Restic repository remained accessible
- Restic repository check completed successfully
- all 4 / 4 Restic snapshots validated
- no Restic errors were found

The generalized Docker backup framework is now the sole reference application
backup mechanism on this host.

## Generic configuration deployment

Validated generic configuration installation after commit `7a39c2c`.

Validated:

- installer contains no application-specific configuration filenames
- application configs are deployed from `config/apps/*.conf`
- repository configs are deployed from `config/repos/*.conf`
- portable configs are deployed from `config/portable/*.conf`
- all three configuration directories are preflighted
- deployment fails if a required configuration directory is missing or empty
- reference application config installed successfully through generic discovery
- reference application repository config installed successfully through generic discovery
- reference-application portable config installed successfully through generic discovery
- all deployed configs matched source exactly
- all 7 reference-application backup timers remained active after deployment

This validates configuration deployment without application-specific installer logic.

## Fully declarative schedule timing

Validated explicit systemd calendar scheduling after commit `97b0efa`.

Validated:

- repository timer schedules are stored directly in repository configuration
- portable timer schedules are stored directly in portable configuration
- timer renderers contain no application-specific schedule mappings
- all 7 configured calendar expressions were accepted by `systemd-analyze calendar`
- all 7 generated timer units passed `systemd-analyze verify`
- live reference-application timer schedule remained unchanged after deployment
- deployed repository configuration matched source exactly
- deployed portable configuration matched source exactly
- deployed framework version matched Git commit

Current reference-application schedule remains:

- 01:00 daily: portable backup
- 01:30 daily: portable retention
- 02:00 daily: Restic operational backup
- 03:00 daily: Restic operational retention
- Sunday 04:00: Restic repository check
- first day of month 04:00: Restic full-data check
- second day of month 04:00: Restic prune

This allows each application/repository to select its own execution times
without modifying framework code.

## Cross-mechanism application capture locking

Validated shared application-level capture locking against the reference application.

Lock hierarchy:

- Restic backup:
  application lock -> repository lock
- portable backup:
  application lock -> portable-target lock
- Restic retention and maintenance:
  repository lock only
- portable retention:
  portable-target lock only

Application lock:

- `/run/lock/docker-backup-app-${APP_NAME}.lock`

Validated:

- an externally held reference application lock blocked Restic backup
- blocked Restic backup exited with status 1
- an externally held reference application lock blocked portable backup
- blocked portable backup exited with status 1
- both backup engines reported that another application capture was running
- Restic retention remained available while the application lock was held
- portable retention remained available while the application lock was held
- no backup data was created during the blocked-capture test
- application lock released successfully after testing

This prevents concurrent portable and Restic capture of the same application
without unnecessarily blocking retention or repository maintenance.

## Final reusable-framework audit

Final audit performed after framework implementation and documentation
consolidation.

Runtime revision:

`88b9536cf61ae7132e7acd70e15e68f9b4c86a3b`

Documentation HEAD at audit:

`3f16ef579206660b78611757d7ea7cddbd3d0be4`

The version difference is intentional. Commits after the runtime revision were
documentation-only and did not require runtime redeployment.

Validated:

- all framework shell scripts passed `bash -n`
- README and all framework documentation files were present and non-empty
- application configuration matched deployed runtime exactly
- repository configuration matched deployed runtime exactly
- portable configuration matched deployed runtime exactly
- all five deployed framework engines matched source exactly
- all 7 reference-application backup and maintenance timers remained scheduled
- Restic NFS source verification passed
- Restic repository opened successfully
- Restic repository check validated all 4 / 4 snapshots
- Restic repository check found no errors
- portable retention dry-run found 3 managed recovery sets
- all 3 portable recovery sets were retained
- no portable recovery sets were selected for expiration
- repository whitespace validation passed
- Git working tree was clean

This establishes the current implementation as the validated reusable baseline
for onboarding additional Docker applications.

## Public deployment with production-config preservation

Validated public deployment revision:

`ee8bd70ec466d7093898ffe62f1911429a434360`

The framework was deployed from the canonical public Git repository using:

    sudo ./scripts/install --preserve-config

Validated:

- deployed revision matched the intended public `main` commit
- all five installed runtime engines matched the Git checkout
- application configuration remained byte-for-byte unchanged
- repository configuration remained byte-for-byte unchanged
- portable-backup configuration remained byte-for-byte unchanged
- Restic password file remained byte-for-byte unchanged
- all seven production timer instances remained enabled
- timer instances were regenerated from retained host-local configuration
- repository check completed successfully after deployment
- repository check completed successfully with the Git checkout temporarily
  moved away
- source checkout was restored afterward with a clean working tree

This validates direct framework deployment from public Git while keeping
deployment-specific production configuration and secrets host-local.

## Post-framework-change end-to-end acceptance

A fresh backup/restore acceptance test was performed against deployed revision:

`ee8bd70ec466d7093898ffe62f1911429a434360`

Fresh Restic generation:

`20260821T005307Z-233885`

Fresh portable recovery set:

`2026-08-21_005314Z`

Validated:

- fresh coordinated Restic database and filesystem components completed
- both Restic components reached `state:complete`
- fresh portable database dump completed
- fresh portable application-state archive completed
- portable `SHA256SUMS` validation passed
- portable database gzip integrity passed
- portable application tar integrity passed
- Restic filesystem component restored non-destructively
- portable filesystem archive restored non-destructively
- all expected application-state paths were present in both restores
- live MariaDB storage remained absent from both filesystem restores
- separately protected bulk media remained absent from both filesystem restores
- portable logical database dump imported into an isolated `mariadb:11`
  container
- portable database restore contained 32 tables
- portable database restore contained 4 triggers
- portable database restore reported Alembic marker
  `0103_roms_facets_provider_ids`
- Restic logical database component was resolved by exact complete tag-set
  matching and restored by immutable snapshot ID
- Restic database snapshot ID was
  `49e39914b291a925e41abc0b8907dfffd953573ff1d6a9c1113318e98f28bf9e`
- restored Restic database dump imported successfully into the isolated
  MariaDB container
- Restic database restore contained 32 tables
- Restic database restore contained 4 triggers
- Restic database restore reported Alembic marker
  `0103_roms_facets_provider_ids`

### Restore-validation lesson: Restic tag filtering

An initial manual database restore test used `restic restore latest` together
with multiple repeated `--tag` arguments.

That command selected the completed filesystem snapshot instead of the
intended database component. The restore itself succeeded, which demonstrated
that a syntactically successful restore is not sufficient proof that the
intended snapshot was selected.

The validation procedure was corrected to:

1. read `restic snapshots --json`;
2. require the complete tag set:
   - application
   - repository
   - generation
   - component
   - `state:complete`;
3. require exactly one matching snapshot;
4. record the full immutable snapshot ID; and
5. restore that exact snapshot ID.

The corrected lookup selected database snapshot `49e39914...`, which restored
`/database.sql` successfully and passed the isolated MariaDB import test.

This exact-snapshot procedure is now required for framework acceptance testing
and documented recovery validation.

### Restore-validation lesson: MariaDB readiness

The first disposable MariaDB restore harness used `mariadb-admin ping`.

The temporary MariaDB server could respond to `ping` during initialization
before password-authenticated SQL access was usable, causing an early import
attempt to fail with an authentication error.

The readiness requirement is therefore an authenticated SQL query such as:

    SELECT 1;

Database import must not begin until that authenticated query succeeds.

## Post-change acceptance policy

After any material framework change affecting installation, capture,
retention, maintenance, snapshot selection, or restore behavior, validation
must include a fresh backup created by the newly deployed revision and a
non-destructive restore of that fresh backup.

Static validation, CI, installer success, and repository integrity checks
remain required, but they do not replace end-to-end backup/restore acceptance.

## PostgreSQL logical-backup development validation for v1.2.0

PostgreSQL logical-backup support was validated on the
`feature/postgresql-logical-backup` development branch before release.

The implementation and regression coverage include:

- PostgreSQL logical capture for Restic using `pg_dump`
- PostgreSQL logical capture for portable recovery sets
- dumps created with `--no-owner` and `--no-acl`
- non-destructive PostgreSQL recovery validation
- base-table, view, and trigger comparison
- optional application-specific semantic database validation
- PostgreSQL production restore execution
- pre-restore logical database checkpoint staging
- database rollback after selected-import failure
- application restart suppression when database rollback fails
- PostgreSQL operation through the installed acceptance-test logic
- regression coverage of both MariaDB and PostgreSQL acceptance paths

After acceptance-harness support was added, the complete executable framework
regression suite passed:

- total tests: 21
- passed: 21
- failed: 0
- `git diff --check`: PASS

The PostgreSQL recovery-validation fixture reported:

- tables: 37
- views: 0
- triggers: 0
- semantic migration marker: `20260802162816`

This is development validation only.

Production validation requires the released framework to be installed through
the normal deployment path and then exercised against a real PostgreSQL-backed
application using fresh Restic and portable recovery artifacts. Vikunja is the
planned production acceptance application for this release.
