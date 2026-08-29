# GitOps

Everything inside the cluster is reconciled from here. Terraform installs ArgoCD
and one root Application; the root Application reconciles `apps/` recursively,
so adding a component is a file in Git rather than a Terraform change.

```
Terraform  ──installs──▶  ArgoCD  ──reconciles──▶  apps/*.yaml
                                                      ├── longhorn
                                                      ├── cloudnative-pg
                                                      ├── cert-manager
                                                      └── the application
```

## Conventions

**Pin every chart.** `targetRevision` is an exact version, never a range and
never `HEAD`. ArgoCD reconciles continuously and on its own schedule, so an
unpinned chart can install something nobody reviewed, with no commit behind it.
Upgrading is a version bump in a PR — which is the review.

**Sync waves order what dependencies cannot express.** ArgoCD applies everything
at once unless told otherwise, and some things genuinely must land first:

| Wave | What | Why it cannot go later |
|---|---|---|
| `-1` | Longhorn, cert-manager | provide a StorageClass and Issuers other apps consume |
| `0` | CloudNativePG operator | its CRDs must exist before a Cluster resource is valid |
| `1` | Postgres cluster | needs both the operator and a StorageClass |
| `2` | web, api | need the database |

Set with `argocd.argoproj.io/sync-wave` on each Application.

**Auto-sync with prune and selfHeal.** Without prune, deleting a file leaves the
resource running forever. Without selfHeal, a manual `kubectl edit` silently
becomes the real state and Git becomes fiction.

**Placement is explicit.** Neither node pool is tainted (see
`terraform/modules/gke/main.tf` for why), so anything that must run on the stable
pool says so with `nodeSelector: { workload: stateful }`. That currently means
Longhorn and CloudNativePG.
