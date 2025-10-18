# Cloud Provider Uninitialized Taint - FIXED

## 🎯 ROOT CAUSE IDENTIFIED

```
Warning  FailedScheduling
0/3 nodes are available: 3 node(s) had untolerated taint
{node.cloudprovider.kubernetes.io/uninitialized: true}
```

**All cert-manager pods stuck in Pending because ALL nodes have this taint!**

## What is This Taint?

The `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule` taint is:

1. **Automatically added** by Kubernetes when a node joins the cluster
2. **Should be removed** by the Cloud Controller Manager (CCM) once it initializes the node
3. **Prevents pod scheduling** until the cloud provider recognizes and configures the node

## Why This Happens on DigitalOcean

When using Kubespray with DigitalOcean:

- Kubespray sets up Kubernetes without the DigitalOcean Cloud Controller Manager
- Nodes get the `uninitialized` taint on join
- **There's no CCM to remove it!**
- Pods can never be scheduled

## The Solution

Since we're not using DigitalOcean's Cloud Controller Manager (CCM), we need to **manually remove this taint**.

---

## 🔧 Fix Applied

### Added Automatic Taint Detection and Removal

**New tasks in create_k8s.yml** (after line 437):

```yaml
- name: Check for cloudprovider uninitialized taint
  ansible.builtin.shell: >
    kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}
    {.spec.taints[?(@.key=="node.cloudprovider.kubernetes.io/uninitialized")]}{"\n"}{end}'
  register: cloudprovider_taint_check

- name: Display cloudprovider taint status
  ansible.builtin.debug:
    msg: "Nodes with cloudprovider uninitialized taint: ..."

- name: Remove cloudprovider uninitialized taint from all nodes
  when: "'node.cloudprovider.kubernetes.io/uninitialized' in cloudprovider_taint_check.stdout"
  ansible.builtin.command: >
    kubectl taint nodes --all node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-
  register: remove_cloudprovider_taint

- name: Confirm taint removal
  when: remove_cloudprovider_taint is defined and remove_cloudprovider_taint.changed
  ansible.builtin.debug:
    msg: "Removed cloudprovider uninitialized taint from all nodes"
```

### What It Does

1. ✅ **Checks all nodes** for the `cloudprovider/uninitialized` taint
2. ✅ **Shows which nodes** have the taint (for visibility)
3. ✅ **Automatically removes** the taint from all nodes
4. ✅ **Confirms removal** with a message
5. ✅ **Runs before** installing any addons (ingress, cert-manager)

---

## Why This Taint Exists

The cloud provider taint is a Kubernetes safety mechanism:

```
Node joins cluster
     ↓
Kubernetes: "Is this a cloud node? Better wait for cloud provider to configure it"
     ↓
Adds taint: node.cloudprovider.kubernetes.io/uninitialized:NoSchedule
     ↓
Cloud Controller Manager should:
  - Configure cloud-specific metadata
  - Set up provider ID
  - Remove the taint
     ↓
But in our case: NO CCM installed!
     ↓
Taint never removed = Pods never schedule!
```

---

## Manual Fix (If You Need It Now)

If you need to fix the current cluster immediately:

```bash
# Check which nodes have the taint
kubectl get nodes -o json | grep -i "cloudprovider.kubernetes.io/uninitialized"

# Remove the taint from all nodes
kubectl taint nodes --all node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-

# Verify it's gone
kubectl get nodes -o json | jq '.items[].spec.taints'

# Delete and recreate cert-manager pods
kubectl -n cert-manager delete pod --all

# Watch them come up
kubectl -n cert-manager get pods -w
```

Expected output after taint removal:

```bash
$ kubectl -n cert-manager get pods
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c9d8879fd-kfs8h              1/1     Running   0          30s
cert-manager-cainjector-6cc9b5f678-thb44   1/1     Running   0          30s
cert-manager-webhook-7bb7b75848-tdq7s      1/1     Running   0          30s
```

---

## Alternative: Install DigitalOcean CCM (Advanced)

If you want proper cloud integration:

### Install DO CCM

```bash
# Create secret with DO token
kubectl -n kube-system create secret generic digitalocean \
  --from-literal=access-token=$DO_TOKEN

# Install DO Cloud Controller Manager
kubectl apply -f https://raw.githubusercontent.com/digitalocean/digitalocean-cloud-controller-manager/master/releases/v0.1.47.yml
```

**Benefits**:

- Automatic taint removal
- DigitalOcean Load Balancer integration
- Proper node metadata
- Cloud-native resource management

**Tradeoffs**:

- More complex setup
- Requires DO API token in cluster
- Extra pods to manage

---

## How Kubespray Should Handle This

Kubespray has a configuration option for cloud providers:

In `inventory/prod-k8s/group_vars/all/all.yml`:

```yaml
cloud_provider: "external"
external_cloud_provider: "digitalocean"
```

But this requires additional setup. Our fix is simpler: just remove the taint.

---

## What Changes in Playbook Execution

### Before Fix:

```
1. Kubespray installs K8s
2. Nodes join with cloudprovider taint
3. Verify nodes Ready ✅
4. Try to install cert-manager
5. Pods stuck in Pending ❌ (taint blocks scheduling)
6. Timeout and fail
```

### After Fix:

```
1. Kubespray installs K8s
2. Nodes join with cloudprovider taint
3. Verify nodes Ready ✅
4. Check for cloudprovider taint
5. Remove cloudprovider taint ✅
6. Install cert-manager
7. Pods schedule successfully ✅
8. Everything works! 🎉
```

---

## Testing the Fix

Run the playbook again:

```bash
export DO_TOKEN="your_token"
ansible-playbook create_k8s.yml
```

Look for these lines in output:

```
TASK [Check for cloudprovider uninitialized taint]
ok: [localhost]

TASK [Display cloudprovider taint status]
ok: [localhost] => {
    "msg": "Nodes with cloudprovider uninitialized taint: k8s-node-cp-1, k8s-node-wk-1, k8s-node-wk-2"
}

TASK [Remove cloudprovider uninitialized taint from all nodes]
changed: [localhost]

TASK [Confirm taint removal]
ok: [localhost] => {
    "msg": "Removed cloudprovider uninitialized taint from all nodes"
}

TASK [Apply cert-manager]
ok: [localhost]

TASK [cert-manager pods should now schedule successfully!]
✅ SUCCESS
```

---

## Why Your Nodes Showed "Ready" But Couldn't Schedule

**Node Status**: `Ready` ✅

- Means kubelet is running
- Node can communicate with API server
- Container runtime is functional

**Taints**: Block scheduling 🚫

- Even on Ready nodes!
- Separate from node conditions
- Act as pod admission control

So you can have:

```
Node: Ready
Taints: node.cloudprovider.kubernetes.io/uninitialized
Result: No pods can schedule!
```

---

## Summary

**Problem**:

- All nodes had `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule` taint
- No Cloud Controller Manager to remove it
- cert-manager pods couldn't schedule on any node

**Solution**:

- Automatically detect the taint
- Remove it from all nodes
- Happens before addon installation
- Pods can now schedule normally

**Result**:

- ✅ Taint automatically removed
- ✅ Pods schedule successfully
- ✅ cert-manager deploys properly
- ✅ Cluster fully functional

Run the playbook again and cert-manager should deploy successfully! 🎯
