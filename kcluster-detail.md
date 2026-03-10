# Kubespray `cluster.yml` — Complete Guide

## What is `cluster.yml`?

`cluster.yml` is the **main Kubespray playbook**. It is the most powerful and
most used playbook — it can deploy a full cluster from scratch OR re-apply
configuration to existing nodes using `--limit`.

---

## Two Ways to Use `cluster.yml`

```
cluster.yml
        │
        ├── Without --limit  → deploys FULL cluster from scratch
        │                      touches every node in inventory
        │
        └── With --limit     → re-applies config to specific existing nodes only
                               does not touch other nodes
```

---

## Use Case 1 — Fresh Cluster Deployment

Use when deploying Kubernetes for the first time on brand new VMs.

```bash
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml \
  -b -v \
  --private-key=~/.ssh/id_rsa
```

### What it installs on every node:
- Container runtime (containerd, Docker, or CRI-O)
- kubelet
- kubeadm
- kubectl
- CNI network plugin (Calico, Flannel, Cilium, etc.)
- etcd (on control plane nodes)
- kube-apiserver, kube-scheduler, kube-controller-manager (on control plane nodes)
- Any enabled add-ons (metrics-server, ingress, cert-manager, etc.)

### Pre-requisites before running:
- All VMs provisioned and SSH accessible
- `inventory.ini` correctly configured with all node IPs
- `group_vars/` configured with desired Kubernetes version, network plugin, etc.
- SSH key in place

---

## Use Case 2 — Re-apply Config to Existing Nodes (`--limit`)

Use when you need to update configuration on nodes that are **already running**
in the cluster — for example after changing `ip=` in inventory.ini.

```bash
# Re-apply config to specific workers only
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml \
  -b -v \
  --private-key=~/.ssh/id_rsa \
  --limit=worker01,worker02,worker03

# Re-apply config to a single node
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml \
  -b -v \
  --private-key=~/.ssh/id_rsa \
  --limit=worker01
```

### When to use `cluster.yml --limit`:

| Situation | Example |
|---|---|
| Uncommented `ip=` on existing workers | Workers need to rebind to private IP |
| Changed container runtime settings | Switched from Docker to containerd |
| Updated CNI plugin config | Changed Calico settings |
| Fixed misconfigured node | Wrong network plugin was applied |
| Node needs full reconfiguration | Config drift from expected state |

---

## Use Case 3 — Why NOT to Use `cluster.yml` for New Nodes

For **brand new** nodes not yet in the cluster, use `scale.yml` instead.

```
New VM never in cluster
        ↓
cluster.yml --limit=newnode  ← works but runs more tasks than needed
scale.yml --limit=newnode    ← correct — optimized for joining new nodes
```

---

## `cluster.yml` vs `scale.yml` — Key Difference

| | `cluster.yml --limit` | `scale.yml --limit` |
|---|---|---|
| **Purpose** | Re-apply config to existing nodes | Add brand new nodes |
| **Node already in cluster** | ✅ Correct | ❌ Skips existing nodes |
| **Brand new node** | ⚠️ Works but not optimal | ✅ Correct |
| **Touches other nodes** | ❌ No (with --limit) | ❌ No (with --limit) |
| **Use for ip= change** | ✅ Yes | ❌ No — skips existing |

---

## Common Flags

| Flag | Purpose | Example |
|---|---|---|
| `-i` | Inventory file | `-i inventory/sample/inventory.ini` |
| `-b` | Become root (sudo) | `-b` |
| `-v` | Verbose output | `-v` or `-vvv` for more detail |
| `--private-key` | SSH private key | `--private-key=~/.ssh/id_rsa` |
| `--limit` | Target specific nodes only | `--limit=worker01,worker02` |
| `-e` | Extra variables | `-e "kube_version=v1.29.0"` |
| `--tags` | Run only specific roles/tags | `--tags=kubelet` |
| `--skip-tags` | Skip specific roles/tags | `--skip-tags=downloads` |

---

## Important `group_vars` for `cluster.yml`

### `group_vars/k8s_cluster/k8s-cluster.yml`
```yaml
# Kubernetes version
kube_version: v1.29.3

# Network plugin
kube_network_plugin: calico

# Pod and service CIDRs
kube_pods_subnet: 10.233.64.0/18
kube_service_addresses: 10.233.0.0/18

# Container runtime
container_manager: containerd

# DNS
cluster_name: cluster.local
```

### `group_vars/k8s_cluster/addons.yml`
```yaml
# Enable add-ons
helm_enabled: true
metrics_server_enabled: true
ingress_nginx_enabled: true
cert_manager_enabled: false
dashboard_enabled: false
```

### `group_vars/all/all.yml`
```yaml
# Load balancer for HA
loadbalancer_apiserver:
  address: 10.0.0.100
  port: 6443

# SSH settings
ansible_user: ubuntu
```

---

## Practical Examples

### Deploy full cluster from scratch
```bash
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml \
  -b -v \
  --private-key=~/.ssh/id_rsa
```

### Re-apply after uncommenting `ip=` on workers
```bash
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml \
  -b -v \
  --private-key=~/.ssh/id_rsa \
  --limit=worker01,worker02,worker03
```

### Re-apply one worker at a time (safer — verify before next)
```bash
# Worker 1
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml -b -v --private-key=~/.ssh/id_rsa \
  --limit=worker01

# Verify
kubectl get nodes

# Worker 2
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml -b -v --private-key=~/.ssh/id_rsa \
  --limit=worker02

# Verify
kubectl get nodes

# Worker 3
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml -b -v --private-key=~/.ssh/id_rsa \
  --limit=worker03
```

### Deploy with specific Kubernetes version
```bash
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml \
  -b -v \
  --private-key=~/.ssh/id_rsa \
  -e "kube_version=v1.29.3"
```

---

## ⚠️ Warnings

### Running without `--limit` on existing cluster
```
cluster.yml without --limit
        ↓
Runs against ALL nodes
        ↓
Re-applies everything to all masters and workers
        ↓
⚠️ May cause brief disruption on running workloads
⚠️ Takes a long time
```
Always use `--limit` on an existing running cluster unless doing a full reinstall.

### `ip=` Change Risk on Running Workers
```
Workers deployed without ip= (public IP used)
        ↓
Uncomment ip= with private IP
        ↓
cluster.yml --limit applies new binding
        ↓
⚠️ kubelet rebinds to new IP
⚠️ Worker may go NotReady temporarily
⚠️ Pods may lose connectivity briefly
```
Always update workers **one at a time** and verify each one is Ready before proceeding.

---

## Decision Tree — Which Playbook to Use

```
What do you need?
        │
        ├── Fresh install on new VMs?
        │         └── cluster.yml (no --limit)
        │
        ├── Update config on EXISTING nodes?
        │         └── cluster.yml --limit=nodename
        │
        ├── Add BRAND NEW nodes to existing cluster?
        │         └── scale.yml --limit=newnode
        │
        ├── Upgrade Kubernetes version?
        │         └── upgrade-cluster.yml
        │
        ├── Remove a node?
        │         └── remove-node.yml -e "node=nodename"
        │
        ├── Control plane broken?
        │         └── recover-control-plane.yml (if SSH reachable)
        │         └── manual etcd recovery (if SSH unreachable)
        │
        └── Wipe everything?
                  └── reset.yml ⚠️ destructive
```

---

## Playbook Reference

| Playbook | Use Case | Touches All Nodes |
|---|---|---|
| `cluster.yml` | Fresh deploy OR re-apply config | ✅ Yes (without --limit) |
| `cluster.yml --limit` | Update existing specific nodes | ❌ No |
| `scale.yml --limit` | Add new nodes only | ❌ No |
| `upgrade-cluster.yml` | Upgrade Kubernetes version | ✅ Yes (rolling) |
| `remove-node.yml` | Remove a node | ❌ No (targeted) |
| `recover-control-plane.yml` | Fix broken control plane | ❌ No (control plane only) |
| `reset.yml` | ⚠️ Wipe entire cluster | ✅ Yes — destructive |