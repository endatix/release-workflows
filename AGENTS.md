# AGENTS.md — technical reference for release-workflows

Context for AI agents (and humans) working on this repo or its consumers.
Covers the decisions behind the design and the non-obvious constraints that
will break things if ignored. User-facing adoption docs live in `README.md`.

## The release model

- **Canary on every push to `main`** of a consumer repo: `vX.Y.Z-canary.N`
  (git tag + GitHub prerelease + all artifacts published).
- **Stable releases are manual and human-reviewed** — never cut from a push:
  dispatch `release-start.yml` → draft GitHub release with generated notes →
  human edits notes → **Publish release** creates the tag → the consumer's
  `release-artifacts` caller publishes/promotes the stable artifacts.
- Commit-message release keywords (`release-major`/`release-feat`/`release-fix`)
  are **retired** — an earlier iteration used them; commit messages never
  trigger or size releases now. The human picks the bump when starting a release.

## How canary versioning works (and why)

semantic-release runs with `main` configured as a **prerelease branch**
(`prerelease: "canary"`, channel `canary`), so it derives `N` from existing
tags. This was chosen over two rejected schemes:

1. `github.run_number` — unique and monotonic but non-sequential (every PR run,
   failed run, and dispatch burns a number).
2. commits-since-last-stable — dense, but a re-run on the same HEAD recomputes
   the same version and collides with the already-pushed tag.

Tag-derived numbering is dense (`.1, .2, .3 …`), collision-free, resets after
each stable, and a re-run on the same HEAD is a clean no-op (the custom
analyzer returns no release for zero new commits).

Canary base = last stable + **patch** (`v2.1.0` → `2.1.1-canary.N`): semver
orders a prerelease *below* its own version (`4.0.0-canary.1 < 4.0.0`), so a
canary must be based on the *next* version to sort above the stable it follows.
The actual stable bump is chosen at promotion time and is not constrained by
the canary base.

## Hard constraints discovered from semantic-release source (v24.2.9)

- **`ERELEASEBRANCHES`**: the branches config needs 1–3 non-prerelease
  branches, validated **after** `expand()` filters the config against branches
  that actually exist on the remote. Hence every consumer keeps a **dormant
  `stable` branch** (created once, never touched). A made-up name in config
  does not satisfy it; `main` listed twice trips the duplicate check.
- On a release branch, prerelease tags are ignored when computing the last
  release (`!semver.prerelease(tag.version)` in get-last-release) — so stable
  release notes aggregate **everything since the last stable**, across canaries.
- On a prerelease branch, a tag only counts for the channel if its git note
  records the channel. Tags created by other tooling (e.g. `gh release create`)
  lack notes and are invisible to semantic-release — source of collisions when
  migrating numbering schemes; cutting a stable release moves the window past
  the foreign tags.
- semantic-release on a non-configured branch exits **before** the prepare
  step, and `--dry-run` also skips prepare. Consequence: it cannot serve as a
  PR build check — that's why `canary.yml` has an explicit validation step.

## Config materialization (why consumers carry no release config)

`canary.yml` writes `release.config.cjs` + `analyze-commits.cjs` into
`$RUNNER_TEMP/release-config/` via heredocs at runtime, installs
semantic-release there, and runs it with `--extends <abs path>`. Rationale:
no npm package to publish, no self-checkout-at-my-own-ref headaches, and the
config version always matches the workflow version.

- The analyzer path inside the config uses `` `${__dirname}/analyze-commits.cjs` ``
  (plugin name strings resolve from CWD, not from the config file).
- `echo '{"private":true}' > package.json` anchors the npm install — without
  it npm walks up to the nearest project root and installs elsewhere.
- **A consumer repo must NOT have its own `.releaserc.*`** — cosmiconfig finds
  it in CWD and it silently overrides `--extends`.
- `DRY_RUN=true` makes the semantic-release invocation a dry run — used for
  local testing: extract the step's `run` script, then
  `RUNNER_TEMP=<tmp> DRY_RUN=true GITHUB_TOKEN=$(gh auth token) DOCKER_IMAGE=… RELEASE_CHANNEL=canary bash <script>`.

## One build per event

- Non-main refs (PRs, branches): the pipeline runs the consumer's
  `scripts/release-prepare.sh "0.0.0-ci"` as the build check. Nothing publishes.
- Pushes to `main`: the **only** build is semantic-release's prepare hook at
  the real version. A broken build fails the release before any tag, GitHub
  release, or artifact exists (semantic-release pushes the tag after prepare).
- All release steps carry `if: github.ref_name == 'main'`; the validation step
  carries the inverse. Consumers therefore need no local build job — extra
  checks (tests, lint) go in additional local caller jobs.

## Hotfix / maintenance mechanics

- Convention: **one long-lived branch per line, named `hotfix/X.Y.x`** (cut
  from the line's stable tag). The name is never parsed — all version math is
  tag-reachability — but the convention keeps one branch per line and stays
  truthful across repeated patches (`v0.8.1`, `v0.8.2`, … from one branch).
- **Prerelease builds per push**: `canary.yml` treats `hotfix/*` as a second
  prerelease lineage (`{ name: "hotfix/*", prerelease: "hotfix", channel:
  "hotfix" }`); pushes publish `X.Y.Z-hotfix.N` with base = nearest stable
  reachable from the branch. The `-hotfix` identifier is deliberately distinct
  from `-canary`: on a shared base (newest-line hotfix) identical identifiers
  would collide in tags and registries. `RELEASE_CHANNEL=hotfix` on these
  pushes; consumers' `pull_request`/`push` filters add `"hotfix/**"`.
- `release-start.yml` has two modes: canary (default) and **ref mode** (`ref`
  input = branch/SHA, typically `hotfix/X.Y.x`). In BOTH modes the version
  base is the **nearest stable ancestor of the released commit** (`git
  describe` from its SHA) — never the globally newest stable — so hotfixing an
  older line computes the right version, and squash-merge histories can't
  break ancestry assumptions (no descendant guard: ancestry by construction).
  Consumers expose ref mode ONLY through a dedicated one-field caller stub
  (`release-hotfix.yml`: branch input, `bump: patch` hardcoded) — the Start
  Stable Release form deliberately has no `ref`, so a hotfix can't be cut
  with the wrong bump and a promotion can't accidentally target a branch.
- **Channel is decided at promote time**, not draft time: release-artifacts
  compares the published tag against the highest stable tag; equal → `stable`,
  lower → `maintenance`. Maintenance exports `RELEASE_CHANNEL=maintenance` to
  the scripts and skips `-t latest` in the image-retag path (which also
  recognizes `-hotfix.N` tags for byte-identical promotion). Contract rule:
  publish scripts move floating pointers ONLY when `RELEASE_CHANNEL=stable`;
  npm dist-tag = channel name otherwise (canary/hotfix/maintenance).
- The draft for a maintenance release is created with `--latest=false` so
  publishing can't steal the GitHub "Latest" badge from the newer line.
- Reachability: on squash-merge repos a hotfix tag NEVER becomes reachable
  from `main` (the squashed commit is a new object). That's fine everywhere:
  semantic-release ignores unreachable tags (main's canaries continue on their
  current base and re-sync at the next stable), the ancestor-based base
  ignores them too, and the version-exists guard uses `git tag -l` (all tags),
  so collisions are still caught immediately. Cross-line fix transport is by
  **cherry-pick** (squash-merge makes repeated whole-branch merges painful);
  the fix must reach main or it regresses in the next release.
- After a newest-line hotfix (e.g. `v0.8.1` of latest `v0.8.0`), the next
  stable from main must be minor+ — a patch would collide with the hotfix
  version, and the version-exists guard reports it.

## Stable flow details

- `release-start.yml` computes next version from the last stable tag
  (`git tag --sort=-v:refname`, prerelease tags excluded by `grep -v -- -`) +
  the human-chosen bump, generates notes with a bash generator (sections:
  Features/Bug Fixes/Performance from conventional subjects, plus **Other
  Changes** for non-conforming messages so nothing is silently dropped;
  `--no-merges` prevents double-counting when a merge commit reuses a
  conventional PR title), then creates a **draft** release pinned to the
  canary's commit. Drafts create **no tag** — deleting the draft aborts cleanly.
- Guards: chosen canary must exist and be a descendant of the last stable
  (rejects already-released canaries); target version must not exist as tag or
  draft.
- `release-artifacts.yml` fires on `release: published`; callers filter
  `!github.event.release.prerelease` (canary prereleases also fire this event).
- **Promotion strategies** (`promotion` input): `image-retag` (default)
  re-tags the canary image digest via `docker buildx imagetools create` —
  byte-identical, no rebuild; `rebuild` runs the consumer's release scripts at
  the tagged commit with `RELEASE_CHANNEL=stable`. **Repos with package
  artifacts (NuGet/npm/Helm) must use `rebuild`** — package versions are baked
  into the artifact and cannot be retagged. Missing canary image falls back to
  rebuild automatically.
- Promoting an older canary (e.g. canary.9 of 10) leaves later canaries
  untouched; their commits flow into the next canary/stable automatically
  because ranges are computed from the new stable tag.

## Credentials model

- **Job `GITHUB_TOKEN`** (caller's, `packages: write`) — every registry push:
  GHCR docker login, NuGet push, npm publish, `helm registry login` (helm does
  not reuse the docker credential store). Repo contents stay read-only.
- **GitHub App installation token** (Endatix Automations; Contents: write
  only) — git tags, GitHub releases, git-notes push. App id/private key arrive
  via Infisical (OIDC, `environment: Production`, env-slug `prod`, project
  `github-actions-for-endatix`). The `@semantic-release/github` comment/label
  options are disabled because the App lacks Issues/PR write.
- Standard env exported to consumer scripts (org-derived, ignored if unused):
  `DOCKER_IMAGE`, `RELEASE_CHANNEL`, `GH_PACKAGES_USER`, `GH_PACKAGES_TOKEN`,
  `NUGET_SOURCE`, `HELM_OCI_REPO`.
- **`RELEASE_CHANNEL` rule** for `release-publish.sh`: `canary` must never move
  floating pointers (docker `latest`, npm `latest` dist-tag); canary npm
  publishes use the `canary` dist-tag.

## Reusable-workflow platform gotchas

- This repo is **private**: callers get "workflow was not found" unless
  `gh api -X PUT repos/endatix/release-workflows/actions/permissions/access -f access_level=organization`
  is set (GitHub reports missing access as not-found).
- `v1` is a **floating tag** — it does not follow `main`. After any change
  here: `git tag -f v1 && git push -f origin v1`, or callers keep the old
  behavior (and fail on inputs the old commit doesn't declare). Breaking
  contract changes get a new `v2` instead. Consumers never track `@main`.
- Caller `env:` does not flow into a called workflow — repo-specific values
  travel as `inputs`; org-constant values are derived from the `github` context.
- A `uses:` job runs on its own runner: toolchains must be set up inside the
  called workflow (`dotnet-version` / `java-version` inputs, skipped if empty).
- `secrets.GITHUB_TOKEN` is implicitly available in called workflows;
  everything else needs `secrets: inherit`.
- Workflow-file parse failures produce runs that cannot be re-run — push a new
  commit or dispatch.

## Consumer contract (summary)

Per adopting repo: `scripts/release-prepare.sh <version>` +
`scripts/release-publish.sh <version>` (executable, honor the env above),
optionally `scripts/release-test.sh` (runs on PRs and before every canary —
merge skew makes the re-run on main mandatory — never on the stable rebuild;
alternative for heavy suites: caller test jobs + `needs:`), the dormant
`stable` branch, three thin caller stubs (see README), squash-merge PRs
(merge commits with conventional titles double-count in notes).

`docker-image` is optional: package-only repos omit it (GHCR login and the
image-retag path are skipped; `DOCKER_IMAGE` is exported empty).
`release-artifacts.yml` also takes `fetch-secrets: true` to inject Infisical
prod secrets into the rebuild path (caller grants `id-token: write`; the
canary workflow injects Infisical secrets unconditionally), and
`nuget-login: true` for nuget.org **OIDC trusted publishing**: `NuGet/login`
mints a short-lived API key per run (no long-lived key stored), exported to
the scripts as `NUGET_API_KEY` (falls back to an Infisical-provided
`NUGET_API_KEY` when the input is off). The login step must run after
Infisical — the nuget.org username arrives as `ENDATIX_NUGET_ORG_USERNAME`
(profile name, not email). Requires a trusted publishing policy on nuget.org
scoped to the caller repo + workflow filename (`release-artifacts.yml` — the
caller stub and this reusable workflow share that filename deliberately).

Current consumers: **test-release** (sandbox; docker-only, `dotnet-version:
10.x`, default `image-retag` promotion), **api-contracts** (NuGet ×2 + npm +
docker + helm; `java-version: 21` + `dotnet-version: 8.x`; `promotion:
rebuild`), and **endatix** (public OSS; ~8 NuGet packages, no image; canaries →
GitHub Packages only, stables → GitHub Packages + nuget.org via trusted
publishing; `promotion: rebuild` + `fetch-secrets: true` + `nuget-login: true`;
canary gated on unit + integration tests via `needs:` in its caller).
