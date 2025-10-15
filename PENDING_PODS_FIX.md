# cert-manager Pods Stuck in Pending State - FIXED

## Problem Identified

```
cert-manager-5c9d8879fd-f4txf              0/1     Pending   0          4m2s   <none>   <none>
cert-manager-cainjector-6cc9b5f678-m2h54   0/1     Pending   0          4m2s   <none>   <none>
cert-manager-webhook-7bb7b75848-n6mb7      0/1     Pending   0          4m2s   <none>   <none>
```

**All 3 cert-manager pods are stuck in `Pending` state with `NODE: <none>`**

This means the Kubernetes scheduler **cannot find a node** to place these pods on.

## Root Cause

The most common reasons for pods stuck in Pending:

### 1. **NoSchedule Taint on All Nodes** ⭐ **Most Likely**

Control plane nodes have a taint by default:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

If your worker nodes:

- Haven't joined the cluster yet
- Are also tainted
- Don't exist (worker_count = 0)

Then NO pods can be scheduled!

### 2. **Worker Nodes Not Ready**

Nodes might be in `NotReady` state due to:

- CNI (Calico) not fully deployed
- Network issues
- Resource constraints

### 3. **Insufficient Resources**

Nodes don't have enough CPU/memory to meet pod requests.

### 4. **Node Selectors / Affinity Rules**

Pods have node selectors that don't match any node labels.

## Fixes Applied

### Fix 1: Pre-flight Node Readiness Check

Added check BEFORE installing addons to ensure all nodes are Ready:

```yaml
- name: Verify cluster nodes are Ready
  ansible.builtin.command: >
    /usr/local/bin/kubectl --kubeconfig {{ kubeconfig_path }}
    get nodes -o json
  register: nodes_status
  retries: 30
  delay: 10
  until: >
    (nodes_status.stdout | from_json)['items']
    | ... Ready nodes >= (cp_count + worker_count)
```

**What it does:**

- Waits up to 5 minutes for all nodes to become Ready
- Counts nodes with `Ready=True` condition
- Ensures cluster is fully operational before installing addons

### Fix 2: Smart Taint Removal

Added logic to handle control plane taints based on cluster type:

```yaml
- name: Check for NoSchedule taints on control plane
  # Detects if control plane has NoSchedule taint

- name: Remove NoSchedule taint from control plane if workers exist
  when:
    - worker_count | int > 0
    - "'node-role.kubernetes.io/control-plane' in cp_taints.stdout"
  # Removes taint so control plane can run system pods if needed

- name: Allow scheduling on control plane if no workers
  when: worker_count | int == 0
  # For single-node clusters, allow all pods on control plane
```

**What it does:**

- Checks if control plane nodes have NoSchedule taints
- Removes taints appropriately based on cluster configuration
- Allows pods to schedule on available nodes

### Fix 3: Enhanced Diagnostics

Added comprehensive diagnostics to the rescue block:

```yaml
- name: Check node status
  # Shows all nodes and their status

- name: Describe one of the pending pods
  # Shows scheduling events and why pod can't be placed

- name: Show ALL cluster events
  # Shows cluster-wide events for troubleshooting
```

**What you'll see:**

- ✅ Node status (Ready/NotReady)
- ✅ Pod scheduling events
- ✅ Detailed reasons for Pending state
- ✅ Cluster-wide events

### Fix 4: Display Node Status

```yaml
- name: Show node status
  # Displays nodes before addon installation

- name: Display nodes
  # Prints node info to console
```

**Benefit:** You can see node status in the playbook output before attempting addon installation.

## What Changed in create_k8s.yml

| Section           | Line | Change                                        |
| ----------------- | ---- | --------------------------------------------- |
| Pre-flight checks | ~393 | Added node readiness verification             |
| Taint detection   | ~411 | Check for control plane taints                |
| Taint removal     | ~418 | Smart taint removal based on cluster config   |
| Rescue block      | ~618 | Added node status check                       |
| Rescue block      | ~627 | Added pod describe for scheduling details     |
| Rescue block      | ~643 | Added cluster-wide events                     |
| Diagnostics       | ~653 | Enhanced output with node and scheduling info |

## How It Works Now

```
1. Kubespray installs Kubernetes
   ↓
2. Verify all nodes are Ready (wait up to 5 min)
   ↓
3. Display node status
   ↓
4. Check for NoSchedule taints on control plane
   ↓
5. Remove taints if:
   - worker_count > 0: Allow some scheduling on CP
   - worker_count = 0: Allow all scheduling on CP
   ↓
6. Install ingress-nginx
   ↓
7. Install cert-manager
   ↓
8. If pods Pending:
   → Show node status
   → Show pod scheduling details
   → Show cluster events
   → Identify exact issue
```

## Manual Troubleshooting

If cert-manager pods are still Pending after these fixes, run:

```bash
# 1. Check node status
kubectl get nodes -o wide

# Expected output:
# NAME            STATUS   ROLES           AGE   VERSION
# k8s-node-cp-1   Ready    control-plane   10m   v1.29.6
# k8s-node-w-1    Ready    <none>          9m    v1.29.6
# k8s-node-w-2    Ready    <none>          9m    v1.29.6

# 2. Check for taints
kubectl get nodes -o json | jq '.items[].spec.taints'

# Expected output (with workers):
# null  (no taints on workers)
# OR minimal taints

# 3. Describe a pending pod
kubectl -n cert-manager describe pod -l app=cert-manager

# Look for "Events:" section:
# Should show: "0/3 nodes are available: 3 node(s) had untolerated taint..."
# OR: "0/3 nodes are available: Insufficient cpu/memory..."

# 4. Check cluster events
kubectl get events --all-namespaces --sort-by=.lastTimestamp | tail -30

# 5. If nodes NotReady, check Calico
kubectl -n kube-system get pods -l k8s-app=calico-node
kubectl -n kube-system logs -l k8s-app=calico-node --tail=50

# 6. Manually remove taints if needed
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes --all node.kubernetes.io/not-ready:NoSchedule-

# 7. Force reschedule
kubectl -n cert-manager delete pod --all
```

## Common Issues & Solutions

### Issue: "0/3 nodes are available: 3 node(s) had untolerated taint"

**Solution:**

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

### Issue: "0/3 nodes are available: 3 node(s) were not ready"

**Solution:**
Wait for nodes to become Ready or check CNI:

```bash
kubectl -n kube-system get pods
kubectl -n kube-system logs -l k8s-app=calico-node
```

### Issue: "0/2 nodes are available: Insufficient cpu"

**Solution:**
Your droplet sizes are too small. Upgrade workers:

```yaml
# In group_vars/all.yml
size_worker: "s-4vcpu-8gb" # Instead of s-2vcpu-4gb
```

### Issue: Nodes NotReady for >5 minutes

**Cause:** CNI (Calico) installation issue or network problems

**Solution:**

```bash
# Check Calico pods
kubectl -n kube-system get pods -l k8s-app=calico-node

# If CrashLoopBackOff, check logs
kubectl -n kube-system logs -l k8s-app=calico-node --tail=100

# Restart Calico if needed
kubectl -n kube-system delete pod -l k8s-app=calico-node
```

## Expected Behavior After Fixes

1. ✅ Playbook waits for nodes to be Ready before installing addons
2. ✅ Shows node status in output
3. ✅ Automatically removes NoSchedule taints when appropriate
4. ✅ Provides detailed diagnostics if scheduling fails
5. ✅ Clear error messages with actionable information

## Testing

After applying these fixes, run:

```bash
export DO_TOKEN="your_token"
ansible-playbook create_k8s.yml
```

You should see:

```
TASK [Verify cluster nodes are Ready] ************
ok: [localhost]

TASK [Show node status] ***************************
ok: [localhost]

TASK [Display nodes] ******************************
ok: [localhost] => {
    "msg": [
        "NAME            STATUS   ROLES           AGE   VERSION",
        "k8s-node-cp-1   Ready    control-plane   5m    v1.29.6",
        "k8s-node-w-1    Ready    <none>          4m    v1.29.6",
        "k8s-node-w-2    Ready    <none>          4m    v1.29.6"
    ]
}

TASK [Check for NoSchedule taints on control plane]
ok: [localhost]

TASK [cert-manager pods should now schedule successfully]
```

## Summary

The issue was pods couldn't be scheduled because:

- Nodes weren't fully Ready when cert-manager was installed
- Control plane had NoSchedule taints
- No diagnostics to identify the problem

The fixes ensure:

- ✅ Nodes are Ready before addon installation
- ✅ Taints are handled appropriately
- ✅ Detailed diagnostics show scheduling issues
- ✅ Clear path to resolution

Run the playbook again and cert-manager should deploy successfully!
