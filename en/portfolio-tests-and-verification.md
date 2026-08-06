## 🧪 Tests & Verification: Full Cluster Rebuild and etcd Restore

Documentation and target-state architecture are one thing; proving the
recovery story actually works under real conditions is another. This
section covers the most complete infrastructure test performed on this
homelab to date: a full K3s node rebuild combined with an etcd restore
from an S3-backed snapshot — triggered, unexpectedly, by a Terraform
incident rather than a planned drill.

### What Happened

A `terraform apply` intended to target a single K3s node instead
destroyed and recreated all three K3s node VMs. The root cause hasn't
been confirmed yet — the leading hypothesis is a shell-quoting issue with
Terraform's `-target` flag against a `for_each`-indexed module, though
`-target` is documented by HashiCorp as a break-glass tool rather than a
guaranteed-safe scoping mechanism in the first place, and that's a more
likely long-term lesson than any one command.

This turned an ordinary maintenance operation into an unplanned,
full-fidelity disaster recovery test: rebuild three K3s server nodes from
Terraform and restore full cluster state from the most recent etcd
snapshot in S3-compatible object storage (MinIO), without any prior
warning or preparation.

### The Recovery Path

1. **Node rebuild** — all three K3s node VMs recreated via the existing
   Terraform module.
2. **etcd restore** — the first node restored directly from the most
   recent etcd snapshot using k3s's built-in
   `--cluster-reset-restore-path` mechanism, pointed at the S3 bucket on
   the NAS; the remaining two nodes rejoined the restored cluster.
3. **Post-restore friction** — an etcd snapshot restores *Kubernetes
   object state* (Deployments, PersistentVolumeClaims, Secrets), but not
   *node-level state*. The rebuilt nodes were missing required host
   packages (`nfs-common`), and the Ceph RBD storage backend held stale
   volume-attachment locks referencing the previous node identities.
   Both had to be resolved before dependent workloads (Photoprism's
   NFS-backed originals volume, in particular) could come back healthy.
4. **Partial automation, confirmed gap** — the existing Ansible role
   handled general host configuration cleanly, but does not currently
   have a code path for "this node is being restored from an etcd
   snapshot." That step, along with clearing the stale storage locks,
   was done manually. This is now a tracked, scoped follow-up rather
   than a hidden gap.
5. **Verification** — confirmed full pod health across the cluster
   (control plane, storage plugins, security stack, and application
   workloads) via `kubectl get pods -A` post-recovery.

### Why This Matters

Backup mechanisms being *configured correctly* and backups being
*actually restorable under real conditions* are two different claims.
Prior verification confirmed the etcd snapshot schedule was running and
producing valid snapshots in S3; this incident is the first time a full
restore was carried through to a healthy, verified cluster state — the
stronger of the two proofs, even though (or arguably because) it wasn't
planned.

It also surfaced a concrete, well-scoped piece of future work: extending
the cluster-bootstrap Ansible role with a dedicated
restore-from-snapshot path, so the next recovery is fully automated
rather than requiring manual intervention.

