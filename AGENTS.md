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
  local testing: extract the `run` scripts of the "Install semantic-release"
  and "semantic-release" steps (the first writes the config and installs the
  tooling into `$RUNNER_TEMP/release-config`, the second runs it), then run
  both with the same `RUNNER_TEMP`:
  `RUNNER_TEMP=<tmp> bash <install>` and
  `RUNNER_TEMP=<tmp> DRY_RUN=true GITHUB_TOKEN=$(gh auth token) DOCKER_IMAGE=… RELEASE_CHANNEL=canary bash <invoke>`.

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
- **Consumers pin by commit SHA**, with `# v1` as a trailing comment — not
  `@v1`. Tracking the mutable tag means a repo picks up new workflow code on its
  next release with no diff and no review, and the caller no longer records which
  version ran. `v1` remains as a **human-readable pointer** to the current
  contract: move it after a change here (`git tag -f v1 <sha> && git push -f
  origin v1`) so `git ls-remote … refs/tags/v1` resolves what to pin to, but
  nothing should track it. Breaking contract changes get a new `v2` instead.
  Consumers never track `@main`.
  Bumping a consumer is then a visible one-line diff, which is also the moment
  its release path gets re-reviewed — the pinned SHA is what actually runs.
- Caller `env:` does not flow into a called workflow — repo-specific values
  travel as `inputs`; org-constant values are derived from the `github` context.
- A `uses:` job runs on its own runner: toolchains must be set up inside the
  called workflow (`dotnet-version` / `java-version` inputs, skipped if empty).
- **Runner choice is the caller's** (`runner` input on all three workflows,
  default `ubuntu-latest`; `runs-on` falls back to it when the input is
  empty). Public repos keep the default; private repos pass a
  self-hosted ARC scale set name (`endatix-docker` for canary/promote, which
  build or push images; `endatix-light` for release-start). A called workflow
  reaches self-hosted runners from the **caller's** context, so only the
  caller repository needs access to the runner group — this repo does not.
  The self-hosted image (`ghcr.io/actions/actions-runner`) has git, curl, jq,
  unzip, tar, gpg, sudo, perl, python3 and the docker CLI with buildx, but no
  SDKs (.NET, Node, Java), gh or az, so the workflows set up what they use:
  `DOTNET_INSTALL_DIR=~/.dotnet` on self-hosted runners (setup-dotnet's Linux
  default `/usr/share/dotnet` is not writable by the runner user; left empty on
  GitHub-hosted runners so their behaviour is unchanged), and release-start
  installs a pinned gh CLI (linux amd64 only) on self-hosted runners when
  `gh` is missing. Node is set up only for semantic-release (main, hotfix/*);
  a consumer script that needs Node, helm or anything else on a self-hosted
  runner installs it itself.
- **No tool download runs with the Infisical secrets, the App token, the
  nuget.org key or the GHCR login in the job.** Toolchains (.NET, JDK, Node)
  and the semantic-release `npm install` run before the Infisical step, the App
  token, the nuget.org login and the GHCR login, in both canary.yml and
  release-artifacts.yml. The claim is that narrow; the exceptions:
  - `GITHUB_TOKEN` is available to every step from the first.
  - **OIDC request token.** Every job with `id-token: write` has
    `ACTIONS_ID_TOKEN_REQUEST_URL` / `ACTIONS_ID_TOKEN_REQUEST_TOKEN` in every
    step from the first. Code in ANY step of such a job, not only an install,
    can mint an OIDC token for the job and, with the Infisical identity id
    written in these workflows, log in to Infisical `prod` itself. Ordering
    the installs first does not prevent that; it only keeps already-fetched
    secrets out of their environment.
  - **The semantic-release tree** (semantic-release@24 and its plugins, from
    major ranges, no lockfile) runs during the semantic-release step with the
    App token and the Infisical env. Accepted. It is installed with
    `--ignore-scripts`, which removes install-time code execution (no
    install/postinstall hooks; today's tree has none, so nothing changed), but
    not run-time execution.
  - The consumer's own code, which can do that minting too, including where
    no secret has been fetched yet: in canary.yml the PR build validation
    (`release-prepare.sh 0.0.0-ci`) and `release-test.sh` (on every event,
    main included) run in environment Production with `id-token: write`
    before Infisical; then `release-prepare.sh` / `release-publish.sh` (and
    the dependency restores they do) under semantic-release on main, with the
    secrets; and the rebuild path of release-artifacts.
- **Known constraint: Infisical's export runs after the toolchain setup.**
  The secrets action exports every key under the project path (`recursive:
  true`) into the job environment after the toolchains have set theirs. In
  canary.yml that was already so for .NET and the JDK; this change moved only
  Node and the semantic-release install before Infisical. In
  release-artifacts.yml (with `fetch-secrets`) .NET and the JDK are newly
  before it. A key named like one of those variables
  (`JAVA_HOME*`, `DOTNET_ROOT`, `npm_config_*`, `PATH`-adjacent names) would
  override the toolchain's value for every later step. Keep such names out of
  the `github-actions-for-endatix` project.
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
`upload-nuget-artifacts: true`, which hands `build/packages/nuget/*.nupkg` to
the caller as an artifact.

**nuget.org trusted publishing works only from the caller's own workflow.**
nuget.org (NuGetGallery's `GitHubTokenPolicyValidator`) checks a GitHub OIDC
token against a policy on several claims: `repository_owner`, the owner and
repository IDs once the policy is permanently enabled, the exact workflow file
name, an optional environment, and a list of banned events. The conclusion
rests on two of them: `repository` must equal the policy's `{owner}/{repo}`,
and `job_workflow_ref` must start with that same
`{owner}/{repo}/.github/workflows/`. In a job of this reusable workflow, called
from another repository, `repository` is the caller while `job_workflow_ref`
names `endatix/release-workflows` (GitHub: "for jobs using a reusable
workflow, the ref path to the reusable workflow"), so no policy can match
either way. A caller-scoped policy fails on `job_workflow_ref` (what left
endatix v0.7.1 tagged with no packages), and a policy scoped to this repo
fails on `repository`. The minted key cannot be
handed back to the caller either (NuGet/login marks it a secret, and GitHub
redacts secrets from job outputs). So the one working path is
`upload-nuget-artifacts: true` here plus a job in the caller's own workflow
that runs `NuGet/login` and pushes: there `repository` and `job_workflow_ref`
name the same repo, and the policy is scoped to that repo and workflow file.
endatix (`release-artifacts.yml`) and endatix-saas
(`release-management-nuget.yml`, its own packing, not this workflow) do this.

The `nuget-login` input (NuGet/login inside this workflow, key exported as
`NUGET_API_KEY`) cannot succeed for any caller in another repository, which is
every caller. It is kept in this change, marked unusable, because removing an
input is a breaking contract change (a caller still passing it would fail to
start) and belongs in its own `v2`-or-cleanup change, not in the runner
work.

Current consumers: **test-release** (sandbox; docker-only, `dotnet-version:
10.x`, default `image-retag` promotion), **api-contracts** (NuGet ×2 + npm +
docker + helm; `java-version: 21` + `dotnet-version: 8.x`; `promotion:
rebuild`), **endatix** (public OSS; ~8 NuGet packages, the image
`ghcr.io/endatix/endatix-api`, which its `release-publish.sh` also mirrors to
`docker.io/endatix/endatix-api` on promotion, and the `endatix-api` Helm chart,
linted and packaged by `release-prepare.sh` and pushed to `HELM_OCI_REPO` by
`release-publish.sh` with `helm registry login` + `helm push` (so on a
self-hosted runner it must install helm itself); canary NuGet packages → GitHub
Packages only, stables → GitHub Packages + nuget.org via trusted publishing;
`dotnet-version: 10.x`; `promotion: rebuild` with `fetch-secrets: true`,
`nuget-login: false` and `upload-nuget-artifacts: true`, the nuget.org login
and push in its own caller job; canary gated on unit + integration tests via
`needs:` in its caller; when it re-pins, both its canary (Node) and its
rebuild (.NET now before Infisical) are affected by the reorder),
**endatix-hub** (public; all three workflows, pinned at `12aed2d`; image +
chart, `docker-image: ghcr.io/endatix/endatix-hub`, no SDK inputs;
release-artifacts with `promotion: rebuild`, `fetch-secrets: true` for the
Docker Hub credentials and `id-token: write`; when it re-pins, only its canary
is affected, by the Node move before Infisical: it sets up no toolchain in
release-artifacts, so nothing moved there), and **endatix-saas** (private; the first `runner` consumer:
`endatix-docker` for canary/promote, `endatix-light` for release-start;
docker-only, `promotion: image-retag`).
