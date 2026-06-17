# DerqAI `.github`

Organization-level reusable GitHub Actions workflows shared across DerqAI repositories.

---

## Edge manifest sync (removed)

**Do not use** `notify-manifests.yml` or upstream `notify-edge-manifests` workflows.

Manifest sync is **AWS-first**:

```text
component publish pipeline SUCCEEDED → EventBridge → derq-edge-bundle_pre-staging | pre-main
```

See [`derq-edge-manifests/MANIFEST_FLOW.md`](https://github.com/DerqAI/derq-edge-manifests/blob/main/MANIFEST_FLOW.md).

Upstream bundle repos need **no** manifest secrets or notify workflows.

---

## Workflows

| Workflow | Status |
|----------|--------|
| `notify-manifests.yml` | **Removed** — was `repository_dispatch` to derq-edge-manifests |

Add new org workflows here when multiple repos need shared GHA logic.
