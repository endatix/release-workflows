# release-workflows

Reusable GitHub Actions workflows implementing the Endatix release model.
Adopting them in a repo gives you:

- **A canary release on every push to `main`** — `vX.Y.Z-canary.N` (git tag,
  GitHub prerelease, artifacts published). Numbering is dense and collision-free;
  it resets after each stable release.
- **Human-reviewed stable releases** — you trigger a workflow that creates a
  **draft** GitHub release with generated notes; you edit the notes, click
  **Publish release**, and the canary's artifacts are promoted to the stable
  version. No stable release ever happens without a human decision.

All release logic (versioning, tags, notes, drafts, promotion, credentials)
lives in this repo. Your repo contributes only: **how to build and push its own
artifacts**.

## Adding a repo — 4 steps

### Step 1 — write the two release scripts

Create these two files in your repo. **The shared workflows execute them by
this exact path and name** — the release fails if they are missing.

Both receive the release version as the first argument (you never compute
versions yourself) and run with these environment variables set:

| Variable | Value | Meaning |
|---|---|---|
| `DOCKER_IMAGE` | from your stub's `docker-image` input | image name to build/push |
| `RELEASE_CHANNEL` | `canary` or `stable` | see the rule below |

**`scripts/release-prepare.sh`** — build every artifact, stamped with the
given version. Example (a .NET service using SDK container publish):

```bash
#!/usr/bin/env bash
set -euo pipefail
VERSION="${1:?usage: release-prepare.sh <version>}"

dotnet publish MyApi.csproj -c Release /t:PublishContainer \
  -p:Version="${VERSION}" \
  -p:ContainerRepository="${DOCKER_IMAGE}" \
  -p:ContainerImageTags="\"${VERSION};latest\""
```

**`scripts/release-publish.sh`** — push what prepare built. **The one rule:
when `RELEASE_CHANNEL=canary`, do NOT move floating pointers** (docker
`latest`, npm `latest` dist-tag, …) — those may only follow stable releases:

```bash
#!/usr/bin/env bash
set -euo pipefail
VERSION="${1:?usage: release-publish.sh <version>}"
CHANNEL="${RELEASE_CHANNEL:-stable}"

docker push "${DOCKER_IMAGE}:${VERSION}"
if [ "$CHANNEL" = "stable" ]; then
  docker push "${DOCKER_IMAGE}:latest"
fi
```

Make both executable and adapt the build/push commands to your repo (Gradle,
npm, Helm, …):

```bash
chmod +x scripts/release-prepare.sh scripts/release-publish.sh
```

A working reference lives in [endatix/test-release](https://github.com/endatix/test-release/tree/main/scripts).

**Optional: `scripts/release-test.sh`** — if present, the pipeline runs it on
PRs *and* again before every canary (main may differ from what the PR
validated), but never on the stable rebuild (a stable promotes a commit whose
canary already required green tests). It may read `GITHUB_REF` /
`GITHUB_EVENT_NAME` and parallelize internally. Repos with heavy multi-runner
test matrices can instead keep tests as local jobs in `ci.yml` gating the
pipeline via `needs:` (separate logs and per-suite checks — see
endatix/endatix).

### Step 2 — create the dormant `stable` branch

```bash
git branch stable main && git push origin stable
```

Required once, by semantic-release's branch validation. Nothing is ever
committed to it and no release runs from it — just leave it alone.

### Step 3 — add the three caller workflows

Copy these into `.github/workflows/`, replacing `<repo>` and the toolchain
inputs (`dotnet-version` / `java-version` — set the ones your release scripts
need; omit the others).

`ci.yml` — one call covers both PR validation and the canary release. Each
event builds exactly once: on PRs/branches the shared pipeline runs your
`release-prepare.sh` at the throwaway version `0.0.0-ci` (build check, nothing
publishes); on pushes to main the canary release's prepare step is the build.

```yaml
name: Build, Validate & Release
on:
  pull_request: { branches: [main] }
  push: { branches: [main] }
  workflow_dispatch:
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false
permissions:
  contents: read
  packages: write
  id-token: write
jobs:
  pipeline:
    uses: endatix/release-workflows/.github/workflows/canary.yml@v1
    with:
      docker-image: ghcr.io/endatix/<repo>
      dotnet-version: "10.x"
    secrets: inherit
```

Repo-specific checks beyond the build (tests, linting) go in additional local
jobs alongside `pipeline`.

`release-start.yml` — the manual trigger you run to begin a stable release:

```yaml
name: Start Stable Release
on:
  workflow_dispatch:
    inputs:
      canary_tag:
        description: "Canary tag to promote (leave empty for the newest canary)"
        required: false
        default: ""
      bump:
        description: "Version bump relative to the last stable release"
        type: choice
        options: [minor, patch, major]
        default: minor
permissions:
  contents: write
jobs:
  draft:
    uses: endatix/release-workflows/.github/workflows/release-start.yml@v1
    with:
      canary-tag: ${{ inputs.canary_tag }}
      bump: ${{ inputs.bump }}
```

`release-artifacts.yml` — reacts when you publish the reviewed draft:

```yaml
name: Publish Stable Artifacts
on:
  release: { types: [published] }
permissions:
  contents: read
  packages: write
jobs:
  promote:
    if: ${{ !github.event.release.prerelease }}
    uses: endatix/release-workflows/.github/workflows/release-artifacts.yml@v1
    with:
      docker-image: ghcr.io/endatix/<repo>
      dotnet-version: "10.x"
```

If your repo publishes package artifacts besides the Docker image (NuGet, npm,
Helm), add `promotion: rebuild` — package versions are baked into the artifacts,
so unlike a Docker image they can't be digest-retagged; the stable release is
rebuilt at the tagged commit via your release scripts instead (see
[endatix/api-contracts](https://github.com/endatix/api-contracts) for a
multi-artifact example).

Notes:
- `docker-image` is optional everywhere — omit it if your repo publishes no
  image (see [endatix/endatix](https://github.com/endatix/endatix), NuGet-only).
- If your stable publish needs secrets beyond the job token (e.g. a nuget.org
  API key), store them in Infisical (prod) and add `fetch-secrets: true` +
  `id-token: write` to your release-artifacts caller. The canary workflow
  injects Infisical secrets automatically.
- To gate the canary on tests, put your test jobs in `ci.yml` and give the
  `pipeline` job `needs: [your-test-jobs]`.

### Step 4 — push and verify

Push to `main`. The run should end with a new `vX.Y.Z-canary.N` tag, a
GitHub **Pre-release**, and your artifacts in the registry.

## Day-to-day usage

- **Ship a canary**: push to `main`. That's it.
- **Cut a stable release**: Actions → *Start Stable Release* → pick the bump
  (and optionally a specific canary tag) → open the draft it creates → edit the
  release notes → **Publish release**. The tag, the release, and the promoted
  artifacts (including `latest`) follow automatically.
- **Abandon a started release**: delete the draft. Nothing was tagged or shipped.

## Who owns what

| Concern | Lives in |
|---|---|
| Version computation, canary numbering | here (`canary.yml`) |
| Git tags, GitHub releases, release notes | here |
| Draft/review/promote stable flow | here (`release-start.yml`, `release-artifacts.yml`) |
| Credentials (Infisical, GitHub App, GHCR login) | here |
| How to **build** artifacts | your repo (`scripts/release-prepare.sh`) |
| How to **push** artifacts | your repo (`scripts/release-publish.sh`) |
| PR build validation | here (runs your `release-prepare.sh` @ `0.0.0-ci` on non-main) |
| Extra checks (tests, linting) | your repo (additional local jobs) |

## Versioning of these workflows

Consumers pin `@v1`. Breaking changes to the contract (inputs, script
interface) bump to `v2` — consumers move deliberately; never track `@main`.

## Org prerequisites (already configured)

- This repo is private: Actions access is set to *organization*
  (`gh api -X PUT repos/endatix/release-workflows/actions/permissions/access -f access_level=organization`).
- The Infisical machine identity and the Endatix Automations GitHub App cover
  all org repos; `canary.yml` reads `ENDATIX_PROJECT_AUTOMATIONS_APP_ID` /
  `ENDATIX_PROJECT_AUTOMATIONS_PRIVATE_KEY` via Infisical.
