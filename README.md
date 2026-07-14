# release-workflows

Reusable GitHub Actions workflows implementing the Endatix release model:

- **Canary on every push to `main`**: `vX.Y.Z-canary.N` via semantic-release
  (main as a prerelease branch; N derived from tags — dense, collision-free,
  re-run safe, resets after each stable). Git tag + GitHub prerelease + artifacts.
- **Stable releases are manual and human-reviewed**: a dispatch workflow creates
  a **draft** GitHub release pinned to a canary's commit with aggregated notes
  since the last stable → you edit the notes → clicking **Publish release**
  creates the tag and promotes the canary's Docker image digest (no rebuild).

The semantic-release configuration lives inside [canary.yml](.github/workflows/canary.yml)
— consumer repos carry **no** release config.

## Adopting in a repo

1. **Provide the artifact contract** — two scripts, the only repo-specific code:
   - `scripts/release-prepare.sh <version>` — build all artifacts at that version
   - `scripts/release-publish.sh <version>` — push them; honor `RELEASE_CHANNEL`
     (`canary` must not move floating tags like docker `latest`)
2. **Create the dormant branch** (satisfies semantic-release branch validation;
   never touched again):
   ```bash
   git branch stable main && git push origin stable
   ```
3. **Add three thin caller workflows** (adjust image name / toolchain inputs):

   `.github/workflows/ci.yml` — your build/validation job plus:
   ```yaml
   permissions: { contents: read, packages: write, id-token: write }
   jobs:
     build: # … repo-specific gating build …
     canary:
       needs: build
       if: github.ref_name == 'main'
       uses: endatix/release-workflows/.github/workflows/canary.yml@v1
       with: { docker-image: ghcr.io/endatix/<repo>, dotnet-version: "10.x" }
       secrets: inherit
   ```

   `.github/workflows/release-start.yml`:
   ```yaml
   on:
     workflow_dispatch:
       inputs:
         canary_tag: { description: "Canary tag (empty = newest)", required: false, default: "" }
         bump: { description: "Version bump", type: choice, options: [minor, patch, major], default: minor }
   permissions: { contents: write }
   jobs:
     draft:
       uses: endatix/release-workflows/.github/workflows/release-start.yml@v1
       with: { canary-tag: "${{ inputs.canary_tag }}", bump: "${{ inputs.bump }}" }
   ```

   `.github/workflows/release-artifacts.yml`:
   ```yaml
   on:
     release: { types: [published] }
   permissions: { contents: read, packages: write }
   jobs:
     promote:
       if: ${{ !github.event.release.prerelease }}
       uses: endatix/release-workflows/.github/workflows/release-artifacts.yml@v1
       with: { docker-image: ghcr.io/endatix/<repo>, dotnet-version: "10.x" }
   ```

## Versioning

Consumers pin `@v1`. Breaking changes to the contract (inputs, script
interface) bump to `v2` — move consumers deliberately, never track `@main`.

## Requirements

- This repo is private: org setting/API `actions/permissions/access` must allow
  organization access (already configured).
- Infisical machine identity + Endatix Automations GitHub App must cover the
  consumer repo (org-wide today).
- Secrets used by canary.yml (via Infisical): `ENDATIX_PROJECT_AUTOMATIONS_APP_ID`,
  `ENDATIX_PROJECT_AUTOMATIONS_PRIVATE_KEY`.
