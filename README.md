# DerqAI `.github`

Organization-level [GitHub Actions](https://docs.github.com/en/actions) reusable workflows shared across DerqAI repositories.

Centralizing workflow logic here keeps upstream repos thin: each caller adds a small workflow file and passes secrets; behavior is maintained in one place.

---

## Workflows

| Workflow | Purpose |
|----------|---------|
| [`notify-manifests.yml`](.github/workflows/notify-manifests.yml) | Notify `derq-edge-manifests` when an upstream repo pushes to `staging` or `main` |

---

## Edge manifest notification

Drives the manifest-driven edge release pipeline in [`derq-edge-manifests`](https://github.com/DerqAI/derq-edge-manifests).

### Flow

```
Upstream repo (e.g. rsu-pc)
  push staging | main
    → notify-edge-manifests.yml   (caller, in upstream repo)
    → notify-manifests.yml        (this repo, reusable)
    → repository_dispatch         (derq-edge-manifests)
    → pre-staging | pre-main      (manifest update, AWS gate, PR)
```

### Branch → event mapping

| Upstream branch | `repository_dispatch` event | Manifest track |
|-----------------|----------------------------|----------------|
| `staging` | `repo-updated` | Integration — `pre-staging` → PR → `staging` |
| `main` | `repo-updated-main` | Production — `pre-main` → PR → `main` |

Other branches are rejected with a clear workflow error.

### Upstream setup

Copy the caller template from `derq-edge-manifests`:

**`.github/workflows/notify-edge-manifests.yml`**

```yaml
name: Notify edge manifests

on:
  push:
    branches: [staging, main]

concurrency:
  group: notify-edge-manifests-${{ github.ref }}
  cancel-in-progress: false

jobs:
  notify:
    uses: DerqAI/.github/.github/workflows/notify-manifests.yml@main
    secrets:
      EDGE_MANIFESTS_TOKEN: ${{ secrets.EDGE_MANIFESTS_TOKEN }}
```

Template source: [`derq-edge-manifests/scripts/upstream-notify.yml`](https://github.com/DerqAI/derq-edge-manifests/blob/main/scripts/upstream-notify.yml)

---

## Secrets

### Upstream repositories (callers)

| Secret | Required | Notes |
|--------|----------|-------|
| `EDGE_MANIFESTS_TOKEN` | Yes | GitHub PAT (classic: `repo` scope). Used only to fire `repository_dispatch`. |

Prefer an **organization secret** shared with repos that notify. If org secrets are unavailable, add the same secret at **repository** level on each caller.

**Do not** add AWS credentials to upstream repos.

### `derq-edge-manifests`

| Secret | Purpose |
|--------|---------|
| `EDGE_MANIFESTS_TOKEN` | Push pre-branches, open PRs, read upstream versions |
| `AWS_ACCESS_KEY_ID` | Trigger CodePipeline pre-gate and release pipelines |
| `AWS_SECRET_ACCESS_KEY` | Same |

---

## PAT requirements

Classic personal access token with **`repo`** scope is sufficient.

The token’s GitHub user must have access to:

- `derq-edge-manifests` (read/write)
- Each upstream repository that calls this workflow
- Upstream repos read by manifest version extraction (when rolled out)

---

## Adding a new upstream repository

1. Add `notify-edge-manifests.yml` (see template above).
2. Grant `EDGE_MANIFESTS_TOKEN` (org secret or repo secret).
3. Ensure the PAT can read the new repo if manifest version pins are extracted from it.
4. Push to `staging` or `main` and confirm `derq-edge-manifests` receives the dispatch.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|----------------|
| `repository_dispatch failed (HTTP 403)` | Invalid token, or token user lacks access to `derq-edge-manifests` |
| `repository_dispatch failed (HTTP 404)` | Wrong manifest repo name or repo missing |
| Workflow fails on branch name | Push was not to `staging` or `main` |
| Dispatch succeeds, nothing runs on manifest | Listeners on `derq-edge-manifests` `main` missing or outdated |
| Duplicate runs | Expected to be serialized per repo+ref via workflow concurrency |

---

## Related documentation

- [`derq-edge-manifests` README](https://github.com/DerqAI/derq-edge-manifests) — manifest tracks, release process, PKCS7 outputs
- [`derq-edge-manifests/MANIFEST_FLOW.md`](https://github.com/DerqAI/derq-edge-manifests/blob/main/MANIFEST_FLOW.md) — architecture and AWS pipelines

---

## Contributing

Changes to shared workflows affect every caller that references `@main`. Prefer:

1. Test with a single upstream pilot (e.g. `rsu-pc`) before broad rollout.
2. Use descriptive commit messages; keep workflow inputs backward compatible when possible.
3. Update this README when adding workflows or changing dispatch contracts.
