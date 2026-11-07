# docker-workflows

Reusable GitHub Actions workflows to build, test and publish Docker images.

## Publish an image to GHCR

[`publish-image.yml`](.github/workflows/publish-image.yml) builds a single `linux/amd64` image,
runs a smoke test of the caller against it, and only then pushes it to the GitHub Container Registry.
What is published depends on the branch and the event:

| Branch and event | Package | Tags | Cleanup after the push |
|---|---|---|---|
| `main` (any event) | `<image-name>` | `main-<timestamp>`, `main` | keep the newest `keep-versions` images |
| other branch, `workflow_dispatch` | `<image-name>-dev` | `dev` | delete the untagged images (the overwritten `dev`) |
| branch matching `dev-excluded-branches`, `workflow_dispatch` | — | fails with an error | — |
| other branch, other event | — | build and smoke test only | — |

- The production timestamp is UTC, formatted `YYYYMMDD.HHMMSS`. Consumers pin
  `main-<timestamp>@sha256:...`: Dependabot treats `main-` as a prefix and proposes the newest timestamp.
  The commit is in the OCI label `revision`.
- The development image is a floating tag in a separate package, not meant to be pinned.
- Old production versions are deleted: a consumer pinning a version older than the newest
  `keep-versions` would no longer be able to pull it.
- No provenance nor SBOM attestations: each version listed in GHCR is a single image.
- The last `FROM` of the Dockerfile must be pinned by digest: it is published as the label
  `org.opencontainers.image.base.digest`.

### Usage

The caller holds the triggers and the permissions, e.g. publish on changes to the image folder:

```yaml
name: publish-my-image
on:
  push:
    branches-ignore:
      - 'dependabot/**'
    paths:
      - 'my-image/**'
      - '.github/workflows/publish-my-image.yml'
  workflow_dispatch:
jobs:
  publish:
    # Do not run in forks
    if: github.repository == 'your-user/your-repo'
    permissions:
      contents: read
      packages: write
    uses: javiertuya/docker-workflows/.github/workflows/publish-image.yml@v1.0.0
    with:
      image-name: my-image
      context: my-image
      smoke-test-script: my-image/smoke-test.sh
      dev-excluded-branches: 'dependabot/* digestabot/*'
```

- Reference an exact release tag: with the `github-actions` ecosystem enabled, Dependabot proposes
  a PR in each caller for every new release. Including the caller workflow in `paths` makes that PR
  build and test the image, and its merge publish it.
- The smoke test script runs from the root of the caller repository, with the image to test in the
  environment variable `SMOKE_IMAGE`. It must exit with an error if the test fails.
- After the first publication of each package (`<image-name>` and `<image-name>-dev`), make it public
  and link it to the repository in the package settings, if needed.
  As the package is created by the workflow, the repository gets the admin role on it, that the
  cleanup of old versions requires.

### Inputs

| Input | Default | Description |
|---|---|---|
| `image-name` | (required) | Name of the package in GHCR; the development package is `<image-name>-dev` |
| `context` | `.` | Folder with the Dockerfile (build context) |
| `smoke-test-script` | (required) | Script of the caller repository that tests the image |
| `dev-excluded-branches` | (required) | Branch patterns, separated by spaces, where the development image can't be published (e.g. the automated update branches, that must not overwrite it) |
| `image-title` | `image-name` | OCI title label |
| `image-description` | repository description | OCI description label |
| `dhi-login` | `false` | Log in to `dhi.io` before the build (needs the DHI secrets) |
| `keep-versions` | `20` | Number of production images kept by the cleanup |

### Secrets

| Secret | Description |
|---|---|
| `DHI_USERNAME`, `DHI_TOKEN` | Credentials of `dhi.io`, required if `dhi-login` is `true` |
