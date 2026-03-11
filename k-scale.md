# Kubespray Playbook Guide — `scale.yml` & When to Use Each Playbook

---

## `scale.yml` — Full Detail

### What It Is
`scale.yml` is designed **only** for adding brand new, unconfigured nodes to an already running cluster. It does NOT reconfigure or update existing nodes.

Brand new node = NEW VM, different IP, fresh OS
        │
        ├── Fresh VM — just provisioned
        ├── Never had Kubernetes installed
        ├── Never joined this cluster before
        ├── No kubelet, no containerd, no kubeadm
        └── Clean OS only (Ubuntu, CentOS, etc.)

### How It Works Internally

```
scale.yml runs
      │
      ▼
Reads inventory.ini
      │
      ▼
For each node in inventory:
      │
      ├── Node already configured? → SKIP ✗
      │
      └── Node is new/unconfigured? → INSTALL & JOIN ✅
                  │
                  ├── Install container runtime (containerd)
                  ├── Install kubelet, kubeadm, kubectl
                  ├── Configure networking (Calico/Flannel/etc)
                  ├── Join node to existing cluster
                  └── Register node in etcd (if control plane)
```

### When to Use `scale.yml`

| Situation | Use scale.yml? |
|---|---|
| Adding a brand new worker node | ✅ Yes |
| Adding a brand new master node | ✅ Yes |
| Updating `ip=` on existing node | ❌ No |
| Changing config on existing node | ❌ No |
| Recovering a broken node | ❌ No |

### How to Run `scale.yml`

```bash
# Always use --limit to target new nodes only
ansible-playbook -i inventory/sample/inventory.ini \
  scale.yml \
  -b -v \
  --private-key=~/.ssh/id_rsa \
  --limit=new-worker04
```

> ⚠️ **Always use `--limit`** pointing to new nodes only.
> Without `--limit`, scale.yml runs against all nodes but still skips configured ones — risky and slow.

### Adding Multiple New Nodes

```bash
# Add multiple workers at once
ansible-playbook -i inventory/sample/inventory.ini \
  scale.yml \
  -b -v \
  --private-key=~/.ssh/id_rsa \
  --limit=new-worker04,new-worker05,new-worker06
```

### Adding New Master Nodes (Restore HA)

```bash
# Step 1 — Add new masters to inventory.ini first
[kube_control_plane]
master-seang  ansible_host=34.126.125.84  ip=10.148.0.7  etcd_member_name=etcd1
new-master01  ansible_host=NEW_PUBLIC_IP  ip=NEW_PRIVATE_IP  etcd_member_name=etcd2
new-master02  ansible_host=NEW_PUBLIC_IP  ip=NEW_PRIVATE_IP  etcd_member_name=etcd3
```

###  Copy kubeadm-config from master-seang(master node that already runing on cluster) First
- When you run scale.yml with --limit=master01,master02, Kubespray skips master-seang entirely, so it never gets a chance to copy files FROM master-seang TO the new masters. so it can not copy everything from master-seang to master01,master02

### so what we have to do :
-  Upload Certs Manually From master-seang First to new master (master01,master02)
```bash
# Run this on master-seang BEFORE running scale.yml
sudo kubeadm init phase upload-certs --upload-certs \
  --config /etc/kubernetes/kubeadm-config.yaml
```
- Then Set The Key in group_vars
```bash
kubeadm_certificate_key: "abc123def456...."  # ← paste key here
```
- then run this scale that not --limit or --limit with running master(master-seang)
```bash
ansible-playbook -i inventory/sample/inventory.ini \
  scale.yml \
  --become \
  --become-user=root \
  --private-key=~/.ssh/id_rsa \
  -v
or 

ansible-playbook -i inventory/sample/inventory.ini \
  scale.yml \
  -b -v \
  --private-key=~/.ssh/id_rsa \
  --limit=master-seang,new-master01,new-master02

```

```bash
scale.yml normal flow (NO --limit):

master-seang                    master01/master02
┌─────────────────┐            ┌─────────────────┐
│ 1. Run on       │            │                 │
│    master-seang │──copies───▶│ 2. Gets files   │
│                 │   files    │    automatically│
│ admin.conf ✅   │            │ admin.conf ✅   │
│ kubeadm-cfg ✅  │            │ kubeadm-cfg ✅  │
└─────────────────┘            └─────────────────┘

scale.yml WITH --limit=master01,master02:

master-seang                    master01/master02
┌─────────────────┐            ┌─────────────────┐
│ ⛔ SKIPPED      │            │                 │
│ --limit skips   │  ✗ no     │ ❌ Never gets   │
│ this node       │   copy    │    files        │
│                 │            │ admin.conf ❌   │
│ admin.conf ✅   │            │ kubeadm-cfg ❌  │
└─────────────────┘            └─────────────────┘

```
- kubeadm-config.yaml doesn't exist on master01/master02 — because these are NEW nodes that haven't been initialized yet. The scale playbook is trying to upload certs but the config file only exists on master-seang.

so this the flow that scale --limit node
- verify
```bash
sudo cat /etc/kubernetes/kubeadm-config.yaml
```
Copy it to master01 and master02:
```bash
# Copy to master01
scp -i ~/.ssh/id_rsa \
  /etc/kubernetes/kubeadm-config.yaml \
  window@10.140.0.18:/tmp/kubeadm-config.yaml

ssh -i ~/.ssh/id_rsa window@10.140.0.18 \
  "sudo mv /tmp/kubeadm-config.yaml /etc/kubernetes/kubeadm-config.yaml"

# Copy to master02
scp -i ~/.ssh/id_rsa \
  /etc/kubernetes/kubeadm-config.yaml \
  window@10.140.0.19:/tmp/kubeadm-config.yaml

ssh -i ~/.ssh/id_rsa window@10.140.0.19 \
  "sudo mv /tmp/kubeadm-config.yaml /etc/kubernetes/kubeadm-config.yaml"
```
- verificate
```bash
ssh -i ~/.ssh/id_rsa window@10.140.0.18 \
  "sudo cat /etc/kubernetes/kubeadm-config.yaml"
ssh -i ~/.ssh/id_rsa window@10.140.0.19 \
  "sudo cat /etc/kubernetes/kubeadm-config.yaml"
```
```bash
# Step 1 - Copy to home dir with sudo
sudo cp /etc/kubernetes/admin.conf ~/admin.conf
sudo chown window:window ~/admin.conf

# Step 2 - SCP to master01
scp -i ~/.ssh/id_rsa ~/admin.conf window@34.80.157.115:/tmp/admin.conf

# Step 3 - SCP to master02
scp -i ~/.ssh/id_rsa ~/admin.conf window@35.229.164.126:/tmp/admin.conf

# Step 4 - Move to correct location on master01
ssh -i ~/.ssh/id_rsa window@34.80.157.115 \
  "sudo mkdir -p /etc/kubernetes && sudo mv /tmp/admin.conf /etc/kubernetes/admin.conf"

# Step 5 - Move to correct location on master02
ssh -i ~/.ssh/id_rsa window@35.229.164.126 \
  "sudo mkdir -p /etc/kubernetes && sudo mv /tmp/admin.conf /etc/kubernetes/admin.conf"

# Step 6 - Cleanup the copy above
rm ~/admin.conf

ssh -i ~/.ssh/id_rsa window@35.229.164.126 \
  "sudo cat /etc/kubernetes/admin.conf"
ssh -i ~/.ssh/id_rsa window@34.80.157.115\
  "sudo cat /etc/kubernetes/admin.conf"  

```

# Step 2 — Run scale.yml targeting new masters only
ansible-playbook -i inventory/sample/inventory.ini \
  scale.yml \
  -b -v \
  --private-key=~/.ssh/id_rsa \
  --limit=new-master01,new-master02
```

### What `scale.yml` Will NOT Do


```

```
❌ Will not update ip= on existing nodes
❌ Will not change container runtime on existing nodes
❌ Will not upgrade Kubernetes version
❌ Will not fix broken nodes
❌ Will not reconfigure networking on existing nodes
```

---

## What to Use Instead of `scale.yml`

### Update Config on Existing Nodes → `cluster.yml --limit`

If you changed something in `inventory.ini` for an **existing** node (like uncommenting `ip=`), use:

Exaple 
```bash
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml \
  -b -v \
  --private-key=~/.ssh/id_rsa \
  --limit=master01,worker02,worker03
```

This re-applies ALL settings to existing nodes without touching the rest of the cluster.

---

## Full Playbook Reference

### `cluster.yml`
**Purpose:** Deploy a full Kubernetes cluster from scratch OR re-apply config to existing nodes with `--limit`.

```bash
# Full fresh deploy
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml -b -v \
  --private-key=~/.ssh/id_rsa

# Re-apply config to specific existing nodes
ansible-playbook -i inventory/sample/inventory.ini \
  cluster.yml -b -v \
  --private-key=~/.ssh/id_rsa \
  --limit=worker01,worker02,worker03
```

| Use when | Example |
|---|---|
| First time deploying cluster | Fresh VMs, no Kubernetes yet |
| Re-applying config to existing nodes | Uncommented `ip=` on workers |
| Fixing misconfigured nodes | Wrong network plugin applied |

---

### `scale.yml`
**Purpose:** Add NEW nodes to an already running cluster.

```bash
ansible-playbook -i inventory/sample/inventory.ini \
  scale.yml -b -v \
  --private-key=~/.ssh/id_rsa \
  --limit=new-worker04
```

| Use when | Example |
|---|---|
| Adding new worker nodes | Scaling up capacity |
| Adding new master nodes | Restoring HA after node loss |

---

### `recover-control-plane.yml`
**Purpose:** Recover a broken or degraded control plane and restore etcd quorum.

```bash
# Always remove dead nodes from inventory.ini FIRST
ansible-playbook -i inventory/sample/inventory.ini \
  recover-control-plane.yml -b -v \
  --private-key=~/.ssh/id_rsa
```

| Use when | Example |
|---|---|
| Master nodes are down | 2 out of 3 masters failed |
| etcd lost quorum | Cluster API is frozen/unreachable |
| `kubectl` commands timing out | API server not responding |

> ⚠️ Remove dead nodes from `inventory.ini` before running.

---

### `remove-node.yml`
**Purpose:** Safely drain and remove a node from the cluster.

```bash
ansible-playbook -i inventory/sample/inventory.ini \
  remove-node.yml -b -v \
  --private-key=~/.ssh/id_rsa \
  -e "node=worker01"
```

| Use when | Example |
|---|---|
| Decommissioning a node | Old hardware being retired |
| Replacing a node | Reprovisioning with new specs |
| Scaling down cluster | Too many workers |

> ⚠️ After running, also remove the node from `inventory.ini`.

---

### `upgrade-cluster.yml`
**Purpose:** Rolling upgrade of Kubernetes to a newer version.

```bash
# Step 1 — Update kube_version in group_vars/k8s_cluster/k8s-cluster.yml
kube_version: v1.29.3

# Step 2 — Run upgrade
ansible-playbook -i inventory/sample/inventory.ini \
  upgrade-cluster.yml -b -v \
  --private-key=~/.ssh/id_rsa
```

| Use when | Example |
|---|---|
| Upgrading Kubernetes version | v1.28 → v1.29 |
| Applying security patches | Critical CVE fix |

> ⚠️ Only upgrade ONE minor version at a time (e.g. 1.27 → 1.28, NOT 1.27 → 1.29).

---

### `reset.yml`
**Purpose:** Completely wipe Kubernetes from all nodes. Destructive and irreversible.

```bash
ansible-playbook -i inventory/sample/inventory.ini \
  reset.yml -b -v \
  --private-key=~/.ssh/id_rsa \
  -e "reset_confirmation=yes"
```

| Use when | Example |
|---|---|
| Starting over completely | Failed deployment beyond repair |
| Decommissioning cluster | Shutting down environment |
| Lab/test environments | Rebuilding from scratch |

> ⚠️ This destroys ALL data including etcd. Always backup before running.
> ⚠️ Never use just because masters are down — use `recover-control-plane.yml` instead.

---

## Decision Tree — Which Playbook to Use?

```
What do you need to do?
        │
        ├── Brand new cluster?
        │         └── cluster.yml
        │
        ├── Masters down / etcd broken?
        │         └── recover-control-plane.yml
        │
        ├── Add NEW nodes?
        │         └── scale.yml --limit=newnode
        │
        ├── Update config on EXISTING nodes?
        │         └── cluster.yml --limit=nodename
        │
        ├── Remove a node?
        │         └── remove-node.yml -e "node=nodename"
        │
        ├── Upgrade Kubernetes version?
        │         └── upgrade-cluster.yml
        │
        └── Wipe everything?
                  └── reset.yml ⚠️ destructive
```

---

## Your Specific Recovery Order

```
Step 1: Edit inventory.ini
        - Remove master01 and master02
        - Uncomment ip= on all workers
        - Fix worker03 duplicate IP (10.3.0.4 → 10.3.0.6)
        - Add [k8s_cluster:children] section
        │
        ▼
Step 2: recover-control-plane.yml
        - Restores etcd quorum on master-seang
        - Brings API server back online
        │
        ▼
Step 3: Verify cluster healthy
        kubectl get nodes
        │
        ▼
Step 4: cluster.yml --limit=worker01,worker02,worker03
        - Applies the uncommented ip= to existing workers
        │
        ▼
Step 5: scale.yml --limit=new-master01,new-master02
        - Add replacement master nodes (if needed)
        │
        ▼
        Cluster fully HA again ✅
```

---

## Common Mistakes

| Mistake | Result | Fix |
|---|---|---|
| Using `scale.yml` to update existing nodes | Changes are skipped silently | Use `cluster.yml --limit` instead |
| Running `scale.yml` without `--limit` | Slow, runs against all nodes | Always add `--limit=newnode` |
| Not removing dead nodes before recovery | Playbook tries to reach dead node and fails | Edit `inventory.ini` first |
| Upgrading 2 minor versions at once | Cluster breaks | Only upgrade one minor version at a time |
| Running `reset.yml` when masters are down | Destroys all workloads unnecessarily | Use `recover-control-plane.yml` first |