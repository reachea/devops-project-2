# ArgoCD Namespace Issue - FIXED

## 🎉 SUCCESS: cert-manager Deployed!

First, **congratulations!** The cloudprovider taint fix worked perfectly:

```
✅ Removed cloudprovider uninitialized taint from all nodes
✅ cert-manager deployed successfully
✅ All 3 cert-manager deployments rolled out
✅ No ImagePullBackOff issues
✅ ClusterIssuer created
```

---

## 🐛 New Issue: ArgoCD Namespace Error

```
fatal: [localhost]: FAILED! =>
  "msg": "Failed to create object: Namespace is required for v1.ServiceAccount"
```

### Root Cause

The ArgoCD official manifest (`install.yaml`) from GitHub:

- Contains resources without explicit `namespace` fields
- Expects to be applied with `kubectl apply -n argocd -f install.yaml`
- The `kubernetes.core.k8s` module can't infer the namespace when using `src:`

**Why it failed**:

```yaml
# Using kubernetes.core.k8s with src:
- name: Apply ArgoCD
  kubernetes.core.k8s:
    src: ./argocd.yaml # Resources without namespace field = error!
```

---

## 🔧 Fix Applied

### Changed from kubernetes.core.k8s to kubectl command

**Before**:

```yaml
- name: Apply ArgoCD
  kubernetes.core.k8s:
    kubeconfig: "{{ kubeconfig_path }}"
    state: present
    src: ./argocd.yaml # ❌ Can't specify namespace for all resources
```

**After**:

```yaml
- name: Create argocd namespace
  kubernetes.core.k8s:
    kubeconfig: "{{ kubeconfig_path }}"
    state: present
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: argocd

- name: Apply ArgoCD
  ansible.builtin.command: >
    kubectl --kubeconfig {{ kubeconfig_path }}
    apply -n argocd -f ./argocd.yaml  # ✅ Applies all resources to argocd namespace
  register: argocd_apply
  changed_when: "'created' in argocd_apply.stdout or 'configured' in argocd_apply.stdout"
```

### Added ArgoCD Readiness Check

```yaml
- name: Wait for ArgoCD server deployment to be ready
  ansible.builtin.command: >
    kubectl -n argocd rollout status deploy/argocd-server --timeout=300s
  retries: 3
  delay: 10
  until: argocd_rollout.rc == 0
```

**Why**: Ensures ArgoCD is ready before trying to read the admin password secret.

---

## What Changed

| Line | Change                               | Reason                         |
| ---- | ------------------------------------ | ------------------------------ |
| ~826 | Added "Create argocd namespace" task | Ensures namespace exists first |
| ~834 | Changed to `kubectl apply -n argocd` | Proper namespace handling      |
| ~838 | Added rollout status check           | Wait for ArgoCD to be ready    |

---

## How It Works Now

```
1. Download ArgoCD manifest ✅
2. Create argocd namespace explicitly ✅ NEW!
3. Apply manifest with namespace flag: kubectl apply -n argocd ✅ FIXED!
4. Wait for argocd-server to be ready ✅ NEW!
5. Create ArgoCD Ingress ✅
6. Read admin password ✅
7. Continue with Dashboard installation ✅
```

---

## Why kubectl Instead of kubernetes.core.k8s?

### kubernetes.core.k8s Limitations

When using `src:` with a file containing multiple resources:

- ❌ Can't specify namespace for all resources at once
- ❌ Resources without namespace field fail
- ❌ No way to set default namespace

### kubectl Command Benefits

```bash
kubectl apply -n argocd -f file.yaml
```

- ✅ Sets default namespace for all resources in the file
- ✅ Resources without namespace go to specified namespace
- ✅ Standard Kubernetes behavior
- ✅ Works with official manifests as-is

---

## Alternative Solutions (Not Used)

### Option 1: Patch the Manifest

```yaml
- name: Add namespace to ArgoCD manifest
  ansible.builtin.replace:
    path: ./argocd.yaml
    regexp: "^metadata:"
    replace: "metadata:\n  namespace: argocd"
```

**Why not**: Complex regex, error-prone, might break valid YAML

### Option 2: Use kubernetes.core.k8s with namespace parameter

```yaml
- name: Apply ArgoCD
  kubernetes.core.k8s:
    kubeconfig: "{{ kubeconfig_path }}"
    state: present
    src: ./argocd.yaml
    namespace: argocd # Doesn't work with 'src'!
```

**Why not**: The `namespace` parameter is ignored when using `src`

### Option 3: Parse and apply each resource separately

```yaml
- name: Parse ArgoCD manifest
  ansible.builtin.shell: kubectl create --dry-run=client -f argocd.yaml -o json
  # Then loop through each resource...
```

**Why not**: Overly complex, reinventing kubectl

---

## Testing

Run the playbook again:

```bash
export DO_TOKEN="your_token"
ansible-playbook create_k8s.yml
```

Or continue from where it failed:

```bash
# If you want to start from ArgoCD installation:
ansible-playbook create_k8s.yml --start-at-task="Create argocd namespace"
```

Expected output:

```
TASK [Create argocd namespace]
ok: [localhost]

TASK [Apply ArgoCD]
changed: [localhost]

TASK [Wait for ArgoCD server deployment to be ready]
ok: [localhost]

TASK [Expose ArgoCD with Ingress + TLS]
changed: [localhost]

TASK [Download Kubernetes Dashboard manifest]
ok: [localhost]

... (continues successfully)
```

---

## Quick Manual Fix (Current Cluster)

If you want to fix the current cluster without re-running the playbook:

```bash
# Create namespace
kubectl create namespace argocd

# Apply ArgoCD
kubectl apply -n argocd -f ./argocd.yaml

# Wait for it to be ready
kubectl -n argocd rollout status deploy/argocd-server

# Create the Ingress manually (or re-run playbook from this task)
```

---

## Summary

**Issue**: ArgoCD manifest resources lacked namespace specifications, causing kubernetes.core.k8s module to fail.

**Fix**:

1. Create `argocd` namespace explicitly
2. Use `kubectl apply -n argocd` instead of kubernetes.core.k8s with src
3. Wait for ArgoCD to be ready before accessing secrets

**Result**:

- ✅ ArgoCD deploys to correct namespace
- ✅ All resources properly namespaced
- ✅ Standard kubectl behavior
- ✅ Works with official manifests

Run the playbook again and it should complete successfully! 🚀
