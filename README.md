# gke-3tier — GitOps

Desired state for the [gke-3tier](https://github.com/mhassaankhokhar/gke-3tier)
cluster. ArgoCD reconciles this repository; nothing here is applied by hand.

```
Terraform (gke-3tier)  ──installs──▶  ArgoCD  ──reconciles──▶  this repo
```

## Why this is a separate repository

Not tidiness — two concrete reasons:

**CI would loop.** When the pipeline builds an image it bumps the tag in the
GitOps repo. If that were the same repo as the application source, every deploy
would be a commit that re-triggers CI. The usual workaround is `[skip ci]` in the
commit message, which is a convention holding back an infinite loop.

**Write access differs.** Whoever can merge application code should not thereby
be able to change what runs in the cluster. Separate repositories make that a
permission rather than a policy.

The application source, Terraform, and CI live in `gke-3tier`. ArgoCD needs read
access to this repository only.

## Access

Private, read by ArgoCD through a **read-only deploy key**. The key can clone
this repository and nothing else — it cannot write here and cannot reach any
other repository.

## Layout

```
argocd/apps/       one Application per component, reconciled recursively
manifests/         raw manifests too small to justify a chart
```

## Conventions

**Pin every chart.** `targetRevision` is an exact version, never a range and
never `HEAD`. ArgoCD reconciles continuously and on its own schedule, so an
unpinned chart can install something nobody reviewed, with no commit behind it.
Upgrading is a version bump in a pull request — which is the review.

**Sync waves order what dependencies cannot express.** ArgoCD applies everything
at once unless told otherwise, and some things must land first:

| Wave | What | Why it cannot go later |
|---|---|---|
| `-2` | cert-manager | provides the Issuers other apps consume |
| `-1` | external-dns, cluster secrets | need the operator's CRDs and the token it projects |
| `0` | Longhorn | provides the RWX StorageClass |
| `1` | CloudNativePG operator | CRDs before any Cluster resource |
| `2` | Postgres cluster | needs the operator and a StorageClass |
| `3` | web, api | need the database |

Set with `argocd.argoproj.io/sync-wave`.

**Auto-sync with prune and selfHeal.** Without prune, deleting a file leaves the
resource running forever. Without selfHeal, a manual `kubectl edit` silently
becomes the real state and Git becomes fiction.

**Placement is explicit.** Neither node pool is tainted — a `NoSchedule` taint
also repels GKE's own system pods, which put cluster DNS on preemptible nodes the
first time it was tried. So anything that must run on the stable pool says so
with `nodeSelector: { workload: stateful }`. That currently means External
Secrets, cert-manager, external-dns, Longhorn and (later) CloudNativePG.

**No admin UI is exposed to the internet.** ArgoCD and Longhorn are reached over
Tailscale. Longhorn's UI in particular ships with no authentication of its own:
anyone who reaches it can view, back up or delete volumes.

## What is not here

External Secrets Operator, the `ClusterSecretStore`, and the deploy key ArgoCD
reads this repository with are installed by Terraform in the other repository —
not because they belong there, but because each one is required in order to read
*this* repository. Everything past that point lives here.

## Secrets

None are stored here. The Cloudflare API token lives in GCP Secret Manager;
External Secrets reads it using Workload Identity — no key file exists — and
projects it into the namespaces that consume it.

Rotating it means adding a new Secret Manager version. Nothing in this
repository changes.

## License

MIT — see [LICENSE](LICENSE). Built by Mohammad Hassan Ur Rehman.
