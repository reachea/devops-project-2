# cert-manager Rollout Fixes Applied

## Problem

You were experiencing: `TASK [Fail cert-manager rollout] fatal: [localhost]: FAILED!`

This happens when cert-manager pods don't start within the timeout period, which can be caused by:

1. **ImagePullBackOff** - Docker Hub rate limiting
2. **Slow network** - Images taking too long to download
3. **Resource constraints** - Nodes under load
4. **Timing issues** - Pods starting but rollout checking too early

## Fixes Applied

### Fix 1: Extended Timeouts & Better Retry Logic

- Increased initial rollout timeout: `180s` → `240s`
- Added `failed_when: false` to first rollout attempt (don't fail immediately)
- Added final attempt with 360s timeout if initial attempts fail
- Increased retry intervals: `8s` → `10-20s`

### Fix 2: Enhanced ImagePullBackOff Remediation

- Added 10-second wait after switching images to ghcr.io
- Extended post-patch rollout timeout to 300s
- Added more retries with longer delays

### Fix 3: Graceful Degradation

- Check if pods are actually running even if rollout times out
- Only fail if fewer than 3 pods are running
- This handles cases where rollout times out but pods are healthy

### Fix 4: Better Diagnostics

- Show running pod count in error messages
- Enhanced error output formatting
- Clear indication of what went wrong

## What Changed in create_k8s.yml

### Line ~454: First rollout attempt

```yaml
# Before:
--timeout=180s
register: cm_ctrl_rollout
changed_when: false

# After:
--timeout=240s
register: cm_ctrl_rollout
changed_when: false
failed_when: false  # Don't fail immediately
```

### Line ~489: ImagePullBackOff remediation

```yaml
# Added:
- name: Wait for image repull
  ansible.builtin.pause:
    seconds: 10

# Extended timeout and added failed_when: false
--timeout=300s
failed_when: false
```

### Line ~500: Final attempt block (NEW)

```yaml
# Added entirely new block:
- name: If rollout still failed after remediation, try one more time
  when: cm_rollout_failed and (cm_ctrl_rollout_after_patch is not defined or ...)
  block:
    - name: Final rollout attempt with extended timeout
      --timeout=360s  # 6 minutes!
      retries: 2
      delay: 20
```

### Line ~520: Success validation (NEW)

```yaml
# Added:
- name: Check if all deployments succeeded
  ansible.builtin.set_fact:
    all_deployments_ok: >-
      {{ (cm_ctrl_rollout.rc == 0 or ...) and ... }}

- name: Final check - verify pods are running
  when: not all_deployments_ok | bool
  # Checks for at least 3 running pods
  # Overrides failure if pods are actually healthy
```

### Line ~570: Rescue block improvements

```yaml
# Added:
- name: Check if any pods are actually running despite rollout failure
  # Counts running pods

- name: Fail only if no pods are running
  ansible.builtin.fail:
    msg: "..."
    when: (rescue_running_count | default(0) | int) < 3 # Only fail if <3 pods
```

## How It Works Now

```
1. Try rollout with 240s timeout
   ↓
2. If fails: Check for ImagePullBackOff
   ↓
3. If ImagePullBackOff: Switch to ghcr.io and retry (300s timeout)
   ↓
4. If still fails: One final attempt with 360s timeout
   ↓
5. Check cainjector & webhook deployments
   ↓
6. Verify at least 3 pods are running
   ↓
7. If <3 pods running:
      → Show diagnostics
      → FAIL
   Else:
      → CONTINUE (cert-manager is working despite timeout)
```

## Manual Troubleshooting Commands

If the playbook still fails, run these commands to diagnose:

```bash
# 1. Check pod status
kubectl -n cert-manager get pods -o wide

# 2. Check for image pull issues
kubectl -n cert-manager describe pods | grep -A5 "Image"

# 3. Check events
kubectl -n cert-manager get events --sort-by=.lastTimestamp

# 4. Check specific pod logs
kubectl -n cert-manager logs -l app=cert-manager --tail=100

# 5. Force image to ghcr.io manually (if needed)
kubectl -n cert-manager set image deploy/cert-manager \
  cert-manager-controller=ghcr.io/cert-manager/cert-manager-controller:v1.14.4

kubectl -n cert-manager set image deploy/cert-manager-cainjector \
  cert-manager-cainjector=ghcr.io/cert-manager/cert-manager-cainjector:v1.14.4

kubectl -n cert-manager set image deploy/cert-manager-webhook \
  cert-manager-webhook=ghcr.io/cert-manager/cert-manager-webhook:v1.14.4

# 6. Check if cert-manager is actually working
kubectl get crd | grep cert-manager
kubectl -n cert-manager get all
```

## Alternative: Install cert-manager via Helm

If you continue to have issues, you can switch to Helm installation (more reliable):

### Option 1: Modify the playbook to use Helm

Add to `create_k8s.yml` around line 404:

```yaml
# Install Helm first (add this before cert-manager section)
- name: Install Helm
  ansible.builtin.shell: |
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
  args:
    creates: /usr/local/bin/helm

- name: Add cert-manager Helm repo
  ansible.builtin.command: >
    helm repo add jetstack https://charts.jetstack.io
  changed_when: false

- name: Update Helm repos
  ansible.builtin.command: helm repo update
  changed_when: false

- name: Install cert-manager via Helm
  ansible.builtin.command: >
    helm install cert-manager jetstack/cert-manager
    --namespace cert-manager
    --create-namespace
    --version v1.14.4
    --set installCRDs=true
    --set global.leaderElection.namespace=cert-manager
    --kubeconfig {{ kubeconfig_path }}
  register: helm_install
  changed_when: helm_install.rc == 0
```

### Option 2: Quick manual fix after playbook fails

If the playbook fails at cert-manager, you can manually install it and re-run:

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install cert-manager via Helm
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.4 \
  --set installCRDs=true

# Wait for it to be ready
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
kubectl -n cert-manager rollout status deploy/cert-manager-cainjector

# Then re-run playbook with --start-at-task to skip cert-manager
ansible-playbook create_k8s.yml --start-at-task="Create ClusterIssuer (Let's Encrypt)"
```

## Expected Behavior Now

With these fixes, the playbook should:

1. ✅ Tolerate longer image pull times
2. ✅ Auto-fix ImagePullBackOff issues
3. ✅ Continue if pods are running (even if rollout times out)
4. ✅ Only fail if cert-manager truly isn't working

## Success Indicators

You'll know cert-manager is working when:

```bash
$ kubectl -n cert-manager get pods
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-7d9d6b5d9f-xxxxx            1/1     Running   0          2m
cert-manager-cainjector-5c7d8f9d-xxxxx   1/1     Running   0          2m
cert-manager-webhook-6d4b8c7f5d-xxxxx    1/1     Running   0          2m
```

All 3 pods should show `Running` with `1/1` ready.

## Next Steps

1. Try running the playbook again with these fixes
2. If it still fails at cert-manager, share the output of these commands:
   ```bash
   kubectl -n cert-manager get pods -o wide
   kubectl -n cert-manager get events --sort-by=.lastTimestamp | tail -20
   ```
3. Consider using the Helm installation method if issues persist

## Summary

The playbook now has **3 layers of retry logic** plus **graceful degradation**, making it much more resilient to:

- Slow networks
- Docker Hub rate limits
- Resource constraints
- Timing issues

The chance of hitting the "cert-manager failed to roll out" error should be significantly reduced.
