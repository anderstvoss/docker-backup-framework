# Framework Versioning

The Docker Backup Framework uses semantic release versions together with the
exact Git commit identity of the deployed source.

## Version sources

The repository contains:

`VERSION`

This file contains the framework release version in:

`MAJOR.MINOR.PATCH`

format.

Published releases are identified by Git tags:

`vMAJOR.MINOR.PATCH`

The release tag and `VERSION` value must agree.

## Version meaning

Increment:

- MAJOR for incompatible framework, configuration, backup-format, or recovery
  behavior changes.
- MINOR for backward-compatible framework capabilities or substantial new
  operational features.
- PATCH for backward-compatible fixes, validation improvements, and minor
  operational corrections.

Version changes are made through the normal protected-main pull-request
workflow.

## Development identity

`scripts/version` reports:

- `FRAMEWORK_VERSION`
- `GIT_COMMIT`
- `GIT_DESCRIBE`

For a Git checkout, the exact commit is always recorded.

For an exported source tree without Git metadata:

- `GIT_COMMIT=unknown`
- `GIT_DESCRIBE=v<FRAMEWORK_VERSION>`

An uncommitted working tree is reported as dirty by `git describe`.

## Deployed identity

The installer writes:

`/etc/docker-backup/DEPLOYED_VERSION`

as shell-compatible metadata:

    FRAMEWORK_VERSION=1.0.0
    GIT_COMMIT=<full Git SHA>
    GIT_DESCRIBE=<Git description>

The exact Git commit is the authoritative identity of the deployed source.
The semantic framework version identifies the release compatibility level.

Deployment and recovery validation records should preserve both.

## Release procedure

A release is produced from protected `main`.

1. Update `VERSION` when required.
2. Implement and review the change through a feature branch and pull request.
3. Require all repository validation and secret-scanning checks to pass.
4. Merge to `main`.
5. Run the deployment drill from the exact merged commit:

       sudo ./scripts/deployment-drill \
         /etc/docker-backup/repos/<repository>.conf \
         /etc/docker-backup/portable/<target>.conf

6. Confirm the deployment drill reports `PASS`.
7. Review the durable validation record under
   `/var/log/docker-backup/acceptance/`.
8. Confirm the deployed version marker identifies the intended release and
   commit.
9. Tag the validated `main` commit with `vMAJOR.MINOR.PATCH`.
10. Push the release tag.

A release tag must not be created before the corresponding deployed revision
has passed the required post-change acceptance test.


## Deployment drill

`scripts/deployment-drill` is the source-side release-validation controller.

It requires a clean committed Git checkout and performs the complete production
framework-change drill:

- records the candidate framework version and exact Git commit
- hashes host-local production configuration and secrets
- deploys with `--preserve-config`
- verifies the structured deployed identity
- verifies configuration and secret preservation
- verifies installed runtime/source parity
- verifies configured timer enablement state
- runs the installed end-to-end acceptance harness
- records the fresh Restic generation, portable recovery set, and immutable
  component snapshot IDs
- temporarily removes the source checkout from its normal path
- verifies the installed runtime can perform a repository check without the
  source checkout
- restores the source checkout
- re-verifies configuration and secret preservation
- writes a durable PASS/FAIL record

Validation records are stored under:

`/var/log/docker-backup/acceptance/`

These records contain operational identities and validation results only.
They must never contain secret values.

The source checkout is restored through failure cleanup if the
source-independence stage fails or the drill is interrupted.

## Post-change acceptance

For a MariaDB logical-backup deployment, the installed acceptance harness is:

    sudo /usr/local/lib/docker-backup/acceptance-test \
      /etc/docker-backup/repos/<repository>.conf \
      /etc/docker-backup/portable/<target>.conf

The acceptance test creates fresh backup artifacts and then validates recovery
from those exact artifacts.

It verifies:

- deployed framework identity
- live database accessibility
- fresh Restic backup generation
- fresh portable recovery set
- Restic repository integrity
- portable checksums and archive integrity
- exact Restic snapshot selection from the complete required tag set
- filesystem restoration from both backup mechanisms
- required path inclusion
- configured exclusion absence
- authenticated disposable MariaDB readiness
- portable logical database restoration
- Restic logical database restoration
- restored table and trigger counts against the live database
- configured application-specific schema/migration identity when present
- disposable local test-resource cleanup

The fresh backup artifacts remain in backup storage as valid recovery points.
Temporary restore directories and the disposable database container are
removed automatically.

## Exact Restic component selection

Acceptance and recovery operations must not rely on ambiguous combinations of
`latest` and repeated `--tag` options.

The framework resolves snapshots from:

`restic snapshots --json`

and requires that a snapshot contain the complete tag set for the intended:

- application
- repository
- generation
- component
- completed state

Exactly one snapshot must match.

The resulting immutable snapshot ID is then supplied directly to `restic
restore`.
