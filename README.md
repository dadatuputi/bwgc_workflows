# bwgc_workflows

Shared GitHub Actions workflows for the image repositories of the [Bitwarden self-hosted on Google Cloud](https://github.com/dadatuputi/bitwarden_gcloud) project: [bwgc_backup](https://github.com/dadatuputi/bwgc_backup), [bwgc_caddy](https://github.com/dadatuputi/bwgc_caddy), [bwgc_countryblock](https://github.com/dadatuputi/bwgc_countryblock) and [bwgc_vaultwarden](https://github.com/dadatuputi/bwgc_vaultwarden).

Each image repository used to carry its own copy of these files, and the copies drifted. Now each repository keeps a short workflow with its own triggers and inputs, and calls the shared one here. A fix lands once, here, and reaches every image when its reference is bumped.

## Workflows

| Workflow | What it does | Caller must grant |
| --- | --- | --- |
| `build-image.yml` | Builds the image, runs the repository's `tests/run-tests.sh` against it, then (outside pull requests) pushes to GHCR, scans with Trivy, uploads the SARIF and signs the digest with cosign. | `contents: read`, `packages: write`, `id-token: write`, `security-events: write` |
| `check-for-updates.yml` | Reports `needs_build` when the base image's layers have moved or packages inside the published image are behind the index. Builds nothing itself. | `contents: read` |
| `keepalive.yml` | Pushes an empty commit once the repository has been quiet for 50 days, so GitHub does not disable its scheduled workflows. | `contents: write` |
| `check-alpine-support.yml` | Opens or updates an issue when the Alpine branch the Dockerfile pins is within 180 days of end of support, and fails once it has passed. | `contents: read`, `issues: write` |

### `build-image.yml` inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `test-image` | required | Local tag for the image under test, e.g. `bwgc_backup:test` |
| `shellcheck` | `false` | Shellcheck `scripts/*.sh` at warning level first |
| `mailpit` | `false` | Start a mailpit SMTP server and pass `BWGC_SMTP_HOST`, `BWGC_SMTP_PORT` and `BWGC_MAILPIT_API` to the tests |

## The contract with an image repository

- A `Dockerfile` at the root, built with the root as context.
- `tests/run-tests.sh`, which tests the image named in `BWGC_IMAGE` and exits non-zero on failure. The tests themselves stay in the image repository, next to what they test.
- The image is published as `ghcr.io/<owner>/<repo>`, lowercased.

## Calling them

```yaml
jobs:
  build:
    uses: dadatuputi/bwgc_workflows/.github/workflows/build-image.yml@<sha> # v1.0.0
    with:
      test-image: bwgc_example:test
    permissions:
      contents: read
      packages: write
      id-token: write
      security-events: write
```

Callers pin a full commit SHA with the release tag as a comment, the same way they pin third-party actions, and Dependabot's `github-actions` updates bump it on each new release. A change here therefore reaches an image only through a reviewed pull request in that image's repository.

## Releasing

Releases are automatic. `release.yml` tags every merged pull request `vMAJOR.MINOR.PATCH` and publishes a GitHub release with generated notes. The pull request's label picks the bump:

| Label | Bump | Use for |
| --- | --- | --- |
| `release:major` | major | A caller has to change: a new required input, a new permission, or a changed contract |
| `release:minor` | minor | A new optional input or capability |
| (none) | patch | Fixes and anything else that changes a workflow |
| `release:none` | no release | README or other changes that no caller runs |

Running the workflow by hand releases whatever is on master that has not been tagged yet.

## Signatures

cosign's keyless certificate names the workflow that signed, which is now `build-image.yml` in this repository rather than a workflow in the image repository. To verify an image:

```sh
cosign verify ghcr.io/dadatuputi/bwgc_backup:master \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github.com/dadatuputi/bwgc_workflows/\.github/workflows/build-image\.yml@'
```
