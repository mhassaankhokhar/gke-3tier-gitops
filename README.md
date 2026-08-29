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
| `0` | Longhorn GKE COS node agent | installs iscsid, which longhorn-manager needs to start |
| `1` | Longhorn | provides the RWX StorageClass |
| `2` | Longhorn storage extras | the RWX class and the NFS module every client node needs |
| `2` | CloudNativePG operator | CRDs before any Cluster resource |
| `3` | Postgres cluster | needs the operator and a StorageClass |
| `4` | web, api | need the database |

Set with `argocd.argoproj.io/sync-wave`.

**Auto-sync with prune and selfHeal.** Without prune, deleting a file leaves the
resource running forever. Without selfHeal, a manual `kubectl edit` silently
becomes the real state and Git becomes fiction.

**Placement is explicit.** Neither node pool is tainted — a `NoSchedule` taint
also repels GKE's own system pods, which put cluster DNS on preemptible nodes the
first time it was tried. So anything that must run on the stable pool says so
with `nodeSelector: { workload: stateful }`. That currently means External
Secrets, cert-manager, external-dns, Longhorn's UI and driver deployer, and
(later) CloudNativePG. Longhorn's replica data is placed by node label rather
than by `nodeSelector` — see below.

**But placement follows the volume's consumer.** Pinning "everything Longhorn
owns" to the stable pool looked like the safe default and was not. Longhorn
classes its CSI driver as a system-managed component, so a single
`systemManagedComponentsNodeSelector: workload:stateful` also pinned
`longhorn-csi-plugin`:

```
longhorn-csi-plugin   DESIRED 2   NODE SELECTOR workload=stateful
```

The CSI node plugin performs the mount on the node the *consuming* pod runs on.
RWX exists here for web and api, which run on spot — so the one pool that had to
be able to mount a Longhorn volume was the only pool that could not, and nothing
on the stable pool wanted RWX in the first place (Postgres takes RWO).

The same mistake had a second layer underneath it. `longhorn-manager` was pinned
to the stable pool as well, on the reasoning that a node without the manager is
not a Longhorn node and so cannot hold replicas. True — and it is also why the
spot pool could still mount nothing after the CSI plugin reached it:

```
FailedAttachVolume: node.longhorn.io "gke-...-spot-196ee9be-x6f7" not found
```

Longhorn will not attach a volume to a node it has not registered, RWX included:
the client only mounts NFS from the share-manager, but the CSI attach still names
the consuming node. So the manager runs everywhere, and replicas are kept off
spot one level down — `createDefaultDiskLabeledNodes` plus the
`node.longhorn.io/create-default-disk` label that the stateful pool carries from
its Terraform `node_config`. A node with no disk can mount a volume and can never
store one:

```
NAME                    READY   DISKS
...-spot-196ee9be-x6f7      True    map[]
...-stateful-...-5e9g       True    map[default-disk-...]
```

Neither layer was found by reading. Both were found by mounting an RWX volume
from a pod on each pool.

**Anything a replaceable node needs is a DaemonSet with no nodeSelector.** A new
spot node arrives with nothing but what DaemonSets give it. A nodeSelector that
excludes that pool is therefore not a restriction, it is a permanent hole — the
DaemonSet simply never appears there. This applies to `longhorn-csi-plugin`, the
COS node agent, and `nfs-module-loader`.

**Longhorn needs a node agent on GKE.** It requires `iscsiadm` and the
`iscsi_tcp` module on the host, and no GKE node image provides them — Longhorn's
docs recommend Ubuntu on GKE "since it contains open-iscsi already", but a GKE
Ubuntu 24.04 node does not have it. The agent Longhorn ships for this case loads
the module and runs `iscsid` in a container, and is pinned to the same version as
the chart so the two cannot drift.

It covers iSCSI and nothing else. Its entrypoint runs `modprobe iscsi_tcp`, so
the *other* host requirement — an NFS client, which every node consuming an RWX
volume needs — is still unmet: COS ships `nfs.ko` and `nfsv4.ko` but loads
neither. `nfs-module-loader` in `manifests/longhorn` does that. "The prerequisite
DaemonSet exists" is not the same as "the prerequisites are met"; the way to tell
them apart is to read the script it runs.

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
