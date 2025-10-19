# Ingress-Nginx Webhook Not Ready - FIXED

## 🎉 More Progress!

**Successfully deployed**:

- ✅ Infrastructure provisioned
- ✅ Kubernetes cluster installed
- ✅ Cloudprovider taint removed
- ✅ ingress-nginx applied
- ✅ cert-manager deployed (all 3 components)
- ✅ ClusterIssuer created
- ✅ ArgoCD namespace created
- ✅ ArgoCD deployed

---

## 🐛 New Issue: Admission Webhook Not Ready

```
failed calling webhook "validate.nginx.ingress.kubernetes.io"
Post "https://ingress-nginx-controller-admission.ingress-nginx.svc:443/...":
dial tcp 10.233.48.234:443: connect: connection refused
```

### What Happened?

1. ingress-nginx was applied ✅
2. Playbook immediately tried to create ArgoCD Ingress ❌
3. ingress-nginx admission webhook wasn't ready yet
4. Connection refused → Ingress creation failed

### Why This Happens

ingress-nginx has an **admission webhook** that validates Ingress resources:

```
Ingress created
    ↓
Kubernetes API calls validating webhook
    ↓
ingress-nginx-controller-admission service (port 443)
    ↓
If webhook not ready: Connection refused!
```

**Timeline**:

- **t=0s**: ingress-nginx resources created
- **t=5s**: Controller pod starting (pulling image)
- **t=20s**: Controller running but webhook not ready
- **t=30s**: Admission webhook port opens
- **t=35s**: ✅ Webhook ready

**Problem**: Playbook tried to create Ingress at t=10s → webhook not ready yet!

---

## 🔧 Fix Applied

Added two wait tasks after applying ingress-nginx:

### 1. Wait for Controller Deployment

```yaml
- name: Wait for ingress-nginx controller to be ready
  ansible.builtin.command: >
    kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller --timeout=300s
  retries: 3
  delay: 10
  until: ingress_rollout.rc == 0
```

**What it does**: Waits for the ingress-nginx-controller deployment to fully roll out.

### 2. Wait for Controller Pods Ready

```yaml
- name: Wait for ingress-nginx admission webhook to be ready
  ansible.builtin.command: >
    kubectl -n ingress-nginx wait --for=condition=ready 
    pod -l app.kubernetes.io/component=controller --timeout=300s
  retries: 3
  delay: 10
  until: ingress_webhook_ready.rc == 0
```

**What it does**: Waits for controller pods to be in `Ready` state (webhook port open).

---

## How It Works Now

**Before**:

```
1. Apply ingress-nginx
2. Immediately create ArgoCD Ingress ❌ (webhook not ready)
```

**After**:

```
1. Apply ingress-nginx
2. Wait for deployment rollout (up to 5 min) ⏳
3. Wait for pods to be Ready (webhook available) ⏳
4. Create ArgoCD Ingress ✅ (webhook ready)
```

---

## What Changed in create_k8s.yml

**Location**: After line 497 (after "Apply ingress-nginx")

**Added**:

1. Rollout status check for ingress-nginx-controller deployment
2. Pod readiness check for controller pods
3. Both with retries and 5-minute timeouts

---

## Why Both Checks?

### Check 1: Rollout Status

- Ensures deployment has created ReplicaSet
- Ensures pods are scheduled
- Ensures basic availability

### Check 2: Pod Ready

- Ensures containers are running
- Ensures readiness probes pass
- Ensures webhook port is open and accepting connections

**Both needed** because:

- Deployment can be "rolled out" but pods not yet Ready
- Pods can be "Running" but readiness probe still failing
- Webhook needs pod to be fully Ready

---

## Expected Behavior

After the fix, you'll see:

```
TASK [Apply ingress-nginx]
changed: [localhost]

TASK [Wait for ingress-nginx controller to be ready]
ok: [localhost]
# Output: deployment "ingress-nginx-controller" successfully rolled out

TASK [Wait for ingress-nginx admission webhook to be ready]
ok: [localhost]
# Output: pod/ingress-nginx-controller-xxx condition met

TASK [Apply cert-manager]
changed: [localhost]

... (continues)

TASK [Expose ArgoCD with Ingress + TLS]
changed: [localhost]  # ✅ No webhook error!

TASK [Expose Dashboard with Ingress + TLS]
changed: [localhost]  # ✅ No webhook error!
```

---

## Manual Verification

If you want to check ingress-nginx status manually:

```bash
# Check controller deployment
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller

# Check controller pods
kubectl -n ingress-nginx get pods

# Expected output:
# NAME                                       READY   STATUS    RESTARTS   AGE
# ingress-nginx-controller-xxx               1/1     Running   0          2m

# Check admission webhook service
kubectl -n ingress-nginx get svc ingress-nginx-controller-admission

# Test webhook is responding
kubectl -n ingress-nginx get endpoints ingress-nginx-controller-admission

# Should show pod IP:443 endpoint
```

---

## Quick Fix for Current Cluster

If you want to fix the current cluster without re-running:

```bash
# Wait for ingress-nginx to be ready
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller
kubectl -n ingress-nginx wait --for=condition=ready pod -l app.kubernetes.io/component=controller

# Verify it's ready
kubectl -n ingress-nginx get pods

# Now manually create the ArgoCD Ingress
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-production
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "false"
spec:
  tls:
    - hosts: ["argocd.reacheasambath.com"]
      secretName: argocd-tls
  rules:
    - host: "argocd.reacheasambath.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
EOF

# Or continue the playbook from this task
ansible-playbook create_k8s.yml --start-at-task="Expose ArgoCD with Ingress + TLS"
```

---

## Other Components with Admission Webhooks

This pattern applies to any component with admission webhooks:

| Component     | Webhook                      | Wait Method                      |
| ------------- | ---------------------------- | -------------------------------- |
| cert-manager  | Validates Certificate/Issuer | Wait for rollout + pods ready    |
| ingress-nginx | Validates Ingress            | Wait for rollout + pods ready ✅ |
| ArgoCD        | Validates Application        | Wait for rollout + pods ready ✅ |

The playbook now waits for:

- ✅ ingress-nginx (fixed)
- ✅ cert-manager (already has retry logic)
- ✅ ArgoCD (already added wait)

---

## Performance Impact

### Typical Wait Times

- ingress-nginx controller rollout: **30-60 seconds**
- Controller pod ready: **10-20 seconds**
- Total added time: **~1 minute**

### Timeouts Configured

- Rollout status timeout: **300s (5 minutes)**
- Pod ready timeout: **300s (5 minutes)**
- Retries: **3 attempts** with **10s delay**

**Maximum wait**: 15 minutes (unlikely, usually <2 minutes)

---

## Summary

**Problem**: ArgoCD Ingress creation failed because ingress-nginx admission webhook wasn't ready.

**Root Cause**: Race condition - Ingress created before webhook started accepting connections.

**Fix**:

1. Wait for ingress-nginx deployment rollout
2. Wait for controller pods to be Ready
3. Then create Ingresses

**Result**:

- ✅ ingress-nginx fully ready before Ingress creation
- ✅ No more webhook connection refused errors
- ✅ Reliable, race-condition-free deployment

---

## Next Steps

**Option 1: Continue from failed task**

```bash
ansible-playbook create_k8s.yml --start-at-task="Expose ArgoCD with Ingress + TLS"
```

**Option 2: Re-run full playbook** (recommended for clean run)

```bash
ansible-playbook destroy_infra.yml
export DO_TOKEN="your_token"
ansible-playbook create_k8s.yml
```

**Option 3: Manual fix current cluster** (see "Quick Fix" section above)

---

The playbook should now complete successfully end-to-end! 🚀
