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

### Step 3 — add the four caller workflows

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
    uses: endatix/release-workflows/.github/workflows/canary.yml@8e04473b8951790b94f7a4486e47d4e714dac9a3 # v1
    with:
      docker-image: ghcr.io/endatix/<repo>
      dotnet-version: "10.x"
    secrets: inherit
```

Repo-specific checks beyond the build (tests, linting) go in additional local
jobs alongside `pipeline`.

Every shared workflow takes an optional `runner` input (default
`ubuntu-latest`; an empty value also means the default). Private repos pass a
self-hosted runner scale set name, e.g. `runner: endatix-docker` for
`canary.yml` / `release-artifacts.yml` (they build or push images) and
`runner: endatix-light` for `release-start.yml`:

```yaml
    with:
      docker-image: ghcr.io/endatix/<repo>
      dotnet-version: "10.x"
      runner: endatix-docker
```

Only the calling repository needs access to the runner group. On such a runner
the workflows set up .NET / the JDK from their inputs, Node for
semantic-release on main and hotfix/*, and gh for release-start (amd64 runners
only: that is the one build it pins); anything else
your `release-*.sh` scripts call they must install.

> **After this change merges:** the examples' pins (`8e04473…`) predate the
> `runner` input. Squash-merging makes a new commit, so once it lands, move
> `v1` to it and, in one follow-up commit, update every example SHA in this
> file to it and delete this note. The branch commit stays fetchable after the
> branch is deleted (the pull request's `refs/pull/<n>/head` keeps it), so a pin
> left on it keeps working silently instead of failing; re-pin anyway, since
> nothing reviewed points there. In each consumer that passes `runner`, after
> `git -C <release-workflows> fetch origin`, list every pin with
> `grep -rn 'release-workflows/.*@' .github/workflows`; no line may still say
> `re-pin`, and for each pin `<file>@<sha>` the SHA must be on `main`
> (`git -C <release-workflows> merge-base --is-ancestor <sha> origin/main`) and
> that same workflow must define the input
> (`git -C <release-workflows> show <sha>:.github/workflows/<file> | grep -q '^      runner:'`).

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
    uses: endatix/release-workflows/.github/workflows/release-start.yml@8e04473b8951790b94f7a4486e47d4e714dac9a3 # v1
    with:
      canary-tag: ${{ inputs.canary_tag }}
      bump: ${{ inputs.bump }}
```

`release-hotfix.yml` — the one-field entry point for hotfix lines (always a
patch on the chosen line; see the Hotfixes section):

```yaml
name: Start Hotfix Release
on:
  workflow_dispatch:
    inputs:
      branch:
        description: "Hotfix line branch to release from (e.g. hotfix/0.8.x)"
        required: true
permissions:
  contents: write
jobs:
  draft:
    uses: endatix/release-workflows/.github/workflows/release-start.yml@8e04473b8951790b94f7a4486e47d4e714dac9a3 # v1
    with:
      ref: ${{ inputs.branch }}
      bump: patch
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
    uses: endatix/release-workflows/.github/workflows/release-artifacts.yml@8e04473b8951790b94f7a4486e47d4e714dac9a3 # v1
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
  image (GHCR login and the image-retag path are then skipped).
- If your stable publish needs secrets beyond the job token, store them in
  Infisical (prod) and add `fetch-secrets: true` + `id-token: write` to your
  release-artifacts caller. The canary workflow injects Infisical secrets
  automatically.
- Publishing stables to **nuget.org**: a short-lived API key is minted per run
  via OIDC trusted publishing, so no long-lived key is stored. One-time setup:
  the nuget.org username in Infisical as `ENDATIX_NUGET_ORG_USERNAME` — this must
  be the **individual profile name of the policy creator**, not an organization —
  plus a trusted publishing policy on nuget.org scoped to **your repo** and the
  workflow file that runs the login. Only one wiring works:

  | | `upload-nuget-artifacts: true` (works) | `nuget-login: true` (does not work) |
  |---|---|---|
  | Where the login runs | a job in **your** workflow | this shared workflow |
  | Policy scoped to | your repo + your workflow file | no policy can match |
  | Push happens in | that job in your caller | your `release-publish.sh` |

  nuget.org requires the token's `repository` claim to equal the policy's repo
  and its `job_workflow_ref` to be a workflow of that same repo. In this shared
  workflow, `repository` is your repo but `job_workflow_ref` names
  `endatix/release-workflows`, so a login here is rejected whatever the policy
  says. The minted key cannot be handed back to your caller either
  (`NuGet/login` marks it a secret, and GitHub redacts secrets from job
  outputs), so the push moves into your workflow. `nuget-login` stays only for
  input compatibility. See [endatix/endatix](https://github.com/endatix/endatix)
  `.github/workflows/release-artifacts.yml` for a working example, and
  [endatix/endatix-relay](https://github.com/endatix/endatix-relay) for a repo
  that logs in from its own workflow directly.
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

## Hotfixes & maintenance lines

To patch a stable release without shipping the unreleased work on `main`, each
version line gets **one long-lived branch named `hotfix/X.Y.x`** (the line, not
a single version — it produces `X.Y.1`, `X.Y.2`, … over its lifetime):

```bash
git checkout -b hotfix/0.8.x v0.8.0      # branch from the line's stable tag
git push -u origin hotfix/0.8.x
```

The branch behaves like a mini-`main` for its line:

1. **Fixes arrive by PR** targeting `hotfix/0.8.x` (the `hotfix/**` entries in
   the caller's `pull_request`/`push` filters make PRs validate exactly like
   PRs to main), or by cherry-pick from main.
2. **Every push/merge to the branch publishes a prerelease build**
   `0.8.1-hotfix.N` — same artifacts, git tag, and GitHub prerelease as a
   canary. The distinct `-hotfix` identifier can never collide with main's
   `-canary` builds even when both lines share a base version. Nothing
   floating moves; npm uses the `hotfix` dist-tag.
3. **Releasing**: Actions → *Start Hotfix Release* → enter the line branch
   (`hotfix/0.8.x`) — that's the only input; the bump is always `patch` →
   review the draft → **Publish release**. The tag is created at the
   **branch's commit** — the release contains the line plus its fixes and
   *never anything from main*, regardless of any merges before or after.
   (The underlying reusable workflow is the same as Start Stable Release;
   the dedicated one-field entry point exists so a hotfix can't accidentally
   be cut with a minor/major bump or from the wrong source.)
4. **Repeated patches just work**: after `v0.8.1` is published (its tag lives
   on the branch), further pushes build `0.8.2-hotfix.N`, and the next
   promotion yields `v0.8.2`. The version base is always the nearest stable
   tag reachable from the branch — the branch *name* is never parsed.

Rules that keep this safe:

- **Older-line releases become *maintenance* automatically.** If a newer
  stable exists (releasing `v0.8.1` while `v0.9.0` is out), the draft is
  created un-"Latest"-ed and publishing moves no floating pointers: docker
  `latest` stays, npm gets `maintenance` instead of `latest`, nuget.org sorts
  by semver on its own.
- **Carry fixes across lines by cherry-pick, not branch merges.** These repos
  squash-merge, and repeatedly squash-merging a long-lived branch into main
  re-presents already-merged commits. Either land the fix on main first and
  cherry-pick it into `hotfix/X.Y.x`, or fix the line first and cherry-pick to
  main via a small PR. Both directions are fine — but the fix **must** reach
  main, or it regresses in the next release.
- **One hotfix branch per line** (the `X.Y.x` name makes this natural). Two
  branches on the same line would race for the same `-hotfix.N` numbers.
- **Main's numbering is untouched by all of this.** Canaries and stables from
  main always select the highest reachable release; hotfix tags on the side
  line sort lower (or aren't reachable at all under squash-merge) and are
  simply never selected.
- Delete the branch only when the line goes out of support. If it was deleted
  early, recreate it from the line's newest tag:
  `git checkout -b hotfix/0.8.x v0.8.1` — tags are the durable anchors.

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

**Pin by commit SHA, with the version as a trailing comment:**

```yaml
uses: endatix/release-workflows/.github/workflows/canary.yml@8e04473b8951790b94f7a4486e47d4e714dac9a3 # v1
```

A mutable tag is a supply-chain risk and an operational one: `v1` is force-moved
whenever this repo changes, so a repo tracking `@v1` silently picks up new
workflow code on its next release with no diff, no review, and no way to tell
from the caller which version actually ran. A SHA is immutable and shows up in a
pull request when it changes. It is the same reason every third-party action in
these workflows is SHA-pinned.

`v1` still exists as a human-readable pointer to the current contract — it is
what the comment refers to, and what you resolve when bumping a pin — but nothing
should *track* it. Breaking changes to the contract (inputs, script interface)
bump to `v2`; consumers move deliberately. Never track `@main`.

To bump a consumer: resolve the tag, then update the pin.

```bash
git ls-remote https://github.com/endatix/release-workflows refs/tags/v1
```

## Org prerequisites (already configured)

- This repo is private: Actions access is set to *organization*
  (`gh api -X PUT repos/endatix/release-workflows/actions/permissions/access -f access_level=organization`).
- The Infisical machine identity and the Endatix Automations GitHub App cover
  all org repos; `canary.yml` reads `ENDATIX_PROJECT_AUTOMATIONS_APP_ID` /
  `ENDATIX_PROJECT_AUTOMATIONS_PRIVATE_KEY` via Infisical.
