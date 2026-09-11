# talos-ansible-playbooks

Ansible playbooks to provide Talos Linux + Cilium + Ceph deployments.

## Use cases

### Core services

| Name      | Description                                                      | URL                    |
|-----------|------------------------------------------------------------------|------------------------|
| Talos     | Linux designed for Kubernetes                                    | https://www.talos.dev/ |
| Cilium    | CNI to provide eBPF-based Networking, Observability and Security | https://cilium.io/     |
| Rook-Ceph | Production ready management for File, Block and Object Storage   | https://rook.io/       |

### Middlewares

| Name       | Description                           | URL                     |
|------------|---------------------------------------|-------------------------|
| Tinkerbell | Bare metal machines provisioning tool | https://tinkerbell.org/ |

## Content

These are a set of playbooks used to operate Talos Linux clusters: day-0, day-1, day-2.

- [Day-0](./day-0/README.md): set prerequisites to deploy a cluster.
- [Day-1](./day-1/README.md): deploy a cluster.
- [Day-2](./day-2/README.md): add nodes, upgrade or destroy resources.

## Homelab fork notes

Driven by the parent `homelab` superproject (`make talos`), whose
`inventory/talos/` is copied in by `make talos-inventory`. Run **`make help`** in
the superproject for the operator scenarios (build from scratch / replace a
failed host / replace a failed Talos VM). Customizations on top of upstream, all
**idempotent + fail-safe** (a second `make talos` is `changed=0`):

- **`make talos` converges** — one command both builds a new cluster and
  **repairs a degraded one**. `apply-conf` gracefully *skips* on an
  already-deployed cluster (was a hard failure); `bootstrap` catches
  `AlreadyExists` and only waits for full-cluster health / pulls kubeconfig after
  an *actual* fresh bootstrap — otherwise it deadlocks waiting on the very nodes
  the next step is about to repair.
- **Rebuilt-node rejoin** (`tasks/rejoin-cleanup.yml`, run by `add-nodes`):
  `add-nodes` matches only **Ready** members, so a rebuilt node (whose stale
  NotReady k8s node still holds its IP) is detected and repaired. Before it
  rejoins, its stale **etcd member** and **k8s node** are removed — always via a
  *healthy* control-plane, and the etcd removal is **quorum-gated: the run aborts
  if removing the member would drop below the post-removal etcd quorum.** No-op
  for a brand-new node.
- **Dry run**: `make talos CHECK=1` (or any day-1/day-2 target) runs
  `ansible --check`; read-only discovery still runs so the gates are evaluated.

### Replace a failed Talos node/VM

Recreate the VM (parent `make proxmox-vms`) so it boots Talos in **maintenance
mode**, then `make talos` — it removes the old node's stale etcd/k8s membership
and joins the fresh one. Verify: `kubectl get nodes` all Ready; `talosctl -n
<cp> etcd members` shows 3 healthy.
