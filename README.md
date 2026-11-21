# Organization workflows

Reusable GitHub Actions workflows shared by repositories in
`kauno-arkivyskupija`.

## `ghcr-cleanup.yml`

Prunes old versions of a GHCR container package. GitHub has no built-in
retention policy for container packages, so each repository schedules this
workflow itself.

The workflow keeps the newest tagged builds and deletes older versions. The
underlying action protects multi-arch children and attestations of kept images,
so deleting a version cannot break an image that is still being kept.

### Caller

```yaml
name: Clean GHCR

on:
    schedule:
        - cron: "0 4 * * 0"
    workflow_dispatch:
        inputs:
            dry_run:
                description: Print deletions instead of applying them
                type: boolean
                default: true

permissions:
    packages: write

jobs:
    cleanup:
        uses: kauno-arkivyskupija/.github/.github/workflows/ghcr-cleanup.yml@v1
        with:
            package-name: my-project
            keep-n-most-recent: 10
            dry-run: ${{ inputs.dry_run || false }}
        secrets:
            token: ${{ secrets.GHCR_CLEANUP_TOKEN || secrets.GITHUB_TOKEN }}
```

`cut-off` defaults to `1d`, so `keep-n-most-recent` is the main retention knob.
It counts tagged builds, not the untagged child manifests.

### Token

- `GITHUB_TOKEN` works when the calling repository has the **Admin** role on
  the package: package → **Settings → Manage Actions access** → repository →
  Role **Admin**. The caller job must grant `packages: write`.
- Otherwise create a **classic** PAT with `read:packages` and `delete:packages`
  and store it as the `GHCR_CLEANUP_TOKEN` repository secret in the calling
  repository. Organization secrets are not available to private repositories on
  GitHub Free.

Run the workflow manually with `dry_run: true` before relying on the schedule.
