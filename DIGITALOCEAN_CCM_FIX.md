# DigitalOcean Cloud Controller Manager (CCM) Fix

## Problem

After deploying ingress-nginx with `type: LoadBalancer`, the LoadBalancer service never gets an external IP address assigned:

```bash
TASK [Wait for ingress-nginx external IP/hostname to be assigned]
FAILED - RETRYING: [localhost]: Wait for ingress-nginx external IP/hostname to be assigned (60 retries left).
...
fatal: [localhost]: FAILED!
```

Looking at the service status:

```json
"status": {
    "loadBalancer": {}
}
```

The `loadBalancer.ingress` field is missing - no IP address is assigned even after 10+ minutes (60 retries × 10 seconds).

## Root Cause

**DigitalOcean Cloud Controller Manager (CCM) is not installed in the cluster.**

### What is CCM?

The Cloud Controller Manager is a Kubernetes component that integrates with cloud provider APIs. For DigitalOcean specifically, the CCM:

1. **LoadBalancer provisioning**: Watches for Services with `type: LoadBalancer` and automatically creates DigitalOcean Load Balancers
2. **Node management**: Updates node information with provider-specific data (region, droplet ID, etc.)
3. **Taint removal**: Removes the `node.cloudprovider.kubernetes.io/uninitialized` taint after initialization
4. **Routes**: Manages network routes between nodes

### Why Kubespray Doesn't Install It

When you set `external_cloud_provider: digitalocean` in Kubespray's configuration:

```yaml
# k8s-cluster.yml
external_cloud_provider: digitalocean
cloud_provider: external
```

Kubespray **only configures the cluster to accept an external CCM** by:

- Setting `--cloud-provider=external` on kubelet
- Adding `--cloud-provider=external` to kube-controller-manager
- Applying the `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule` taint to nodes

**But Kubespray does NOT actually install the DigitalOcean CCM itself.** This is by design - Kubespray leaves cloud provider integration as a manual post-installation step.

### Impact

Without the CCM:

- ✅ The cluster runs fine
- ✅ Pods can be scheduled (after we manually remove the cloudprovider taint)
- ❌ **LoadBalancer services never get external IPs**
- ❌ Cannot create DigitalOcean Load Balancers from Kubernetes
- ❌ ingress-nginx, ArgoCD, Dashboard cannot be exposed via LoadBalancer

## Solution

Install the DigitalOcean CCM as a DaemonSet in the `kube-system` namespace.

### Implementation

Add these tasks to the playbook **after taint removal and before ingress-nginx installation**:

```yaml
# --- Install DigitalOcean Cloud Controller Manager (CCM) ---
- name: Create DigitalOcean CCM secret with API token
  kubernetes.core.k8s:
    kubeconfig: "{{ kubeconfig_path }}"
    state: present
    definition:
      apiVersion: v1
      kind: Secret
      metadata:
        name: digitalocean
        namespace: kube-system
      type: Opaque
      stringData:
        access-token: "{{ lookup('env', 'DO_AUTH_TOKEN') }}"

- name: Download DigitalOcean CCM manifest
  ansible.builtin.get_url:
    url: https://raw.githubusercontent.com/digitalocean/digitalocean-cloud-controller-manager/master/releases/v0.1.47.yml
    dest: /tmp/do-ccm.yaml
    mode: "0644"

- name: Apply DigitalOcean CCM
  kubernetes.core.k8s:
    kubeconfig: "{{ kubeconfig_path }}"
    state: present
    src: /tmp/do-ccm.yaml

- name: Wait for DigitalOcean CCM to be ready
  ansible.builtin.command: >
    /usr/local/bin/kubectl --kubeconfig {{ kubeconfig_path }}
    -n kube-system rollout status daemonset/digitalocean-cloud-controller-manager --timeout=180s
  register: ccm_rollout
  changed_when: false
  retries: 3
  delay: 10
  until: ccm_rollout.rc == 0
```

### How It Works

1. **Secret Creation**: The CCM needs your DigitalOcean API token to make API calls. We create a secret in `kube-system` with the `DO_AUTH_TOKEN` environment variable.

2. **CCM Deployment**: The official DigitalOcean CCM manifest creates:

   - ServiceAccount, ClusterRole, ClusterRoleBinding for RBAC
   - DaemonSet that runs the CCM on every node
   - The CCM pods watch for LoadBalancer services and create DigitalOcean LBs

3. **Rollout Wait**: We wait for the DaemonSet to be ready before proceeding to ingress-nginx installation.

### Verification

After the CCM is installed, verify it's running:

```bash
kubectl -n kube-system get daemonset digitalocean-cloud-controller-manager
kubectl -n kube-system get pods -l k8s-app=digitalocean-cloud-controller-manager
kubectl -n kube-system logs -l k8s-app=digitalocean-cloud-controller-manager
```

You should see logs like:

```
I1019 09:20:45.123456       1 controller.go:123] Successfully initialized DigitalOcean cloud provider
I1019 09:20:46.234567       1 service_controller.go:456] Processing service: ingress-nginx/ingress-nginx-controller
I1019 09:20:48.345678       1 service_controller.go:789] Created DigitalOcean Load Balancer: lb-abc123...
```

### Testing LoadBalancer Creation

After CCM is installed, create a test LoadBalancer service:

```bash
kubectl create service loadbalancer test-lb --tcp=80:80
kubectl get svc test-lb -w
```

Within 2-3 minutes, you should see an external IP assigned:

```
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)
test-lb   LoadBalancer   10.233.1.200   146.190.123.45    80:30123/TCP
```

The EXTERNAL-IP field will populate once the DigitalOcean Load Balancer is provisioned.

## Technical Deep Dive

### CCM Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Kubernetes Cluster                                          │
│                                                             │
│  ┌────────────────┐         ┌─────────────────────────┐   │
│  │ LoadBalancer   │         │ kube-system namespace   │   │
│  │ Service        │────────▶│                         │   │
│  │ (ingress-nginx)│         │ ┌─────────────────────┐ │   │
│  └────────────────┘         │ │ digitalocean-ccm    │ │   │
│                             │ │ DaemonSet           │ │   │
│                             │ │                     │ │   │
│                             │ │ - Watches Services  │ │   │
│                             │ │ - Calls DO API      │ │   │
│                             │ │ - Updates Service   │ │   │
│                             │ └─────────────────────┘ │   │
│                             └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                                       │
                                       │ DO API Token
                                       ▼
                      ┌────────────────────────────────┐
                      │ DigitalOcean API               │
                      │                                │
                      │ - Create Load Balancer         │
                      │ - Configure backend droplets   │
                      │ - Assign external IP           │
                      │ - Health checks                │
                      └────────────────────────────────┘
                                       │
                                       ▼
                      ┌────────────────────────────────┐
                      │ Load Balancer Created          │
                      │ IP: 146.190.123.45             │
                      │ Backends: node1:30613, etc.    │
                      └────────────────────────────────┘
```

### Service Reconciliation Flow

1. **User creates LoadBalancer service** (e.g., ingress-nginx-controller)

   - Service spec: `type: LoadBalancer`, ports, selector

2. **CCM detects the service** via Kubernetes API watch

   - Controller loop runs every 30 seconds
   - Finds services with `type: LoadBalancer` and no external IP

3. **CCM calls DigitalOcean API** to create Load Balancer

   ```
   POST /v2/load_balancers
   {
     "name": "a1234567890abcdef1234567890abcde",
     "region": "sgp1",
     "forwarding_rules": [
       {"entry_port": 80, "target_port": 30613, "entry_protocol": "tcp"},
       {"entry_port": 443, "target_port": 30755, "entry_protocol": "tcp"}
     ],
     "health_check": {"port": 30596, "protocol": "tcp"},
     "droplet_ids": [123456789, 123456790, 123456791]
   }
   ```

4. **DigitalOcean provisions Load Balancer** (2-3 minutes)

   - Allocates public IP address
   - Configures forwarding rules
   - Adds droplets as backends
   - Starts health checks

5. **CCM polls DigitalOcean API** for LB status

   - Waits for `status: active`
   - Gets the public IP address

6. **CCM updates Service in Kubernetes**

   ```bash
   kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{
     "status": {
       "loadBalancer": {
         "ingress": [{"ip": "146.190.123.45"}]
       }
     }
   }'
   ```

7. **LoadBalancer is ready**
   - Service shows EXTERNAL-IP
   - Traffic flows: Internet → LB IP → NodePort → Pod

### CCM vs Manual Load Balancer

Without CCM, you could manually create a DigitalOcean Load Balancer and point it to NodePorts, but:

❌ **Manual approach problems:**

- Manual LB creation via DigitalOcean UI/API
- Manual configuration of backend droplets
- Manual health check setup
- No automatic updates when nodes change
- No integration with Kubernetes Service object
- Cannot use `kubectl get svc` to see LB IP
- No automatic cleanup when service is deleted

✅ **With CCM:**

- Fully automated LB provisioning
- Kubernetes-native workflow (`kubectl apply`)
- Automatic backend updates
- Service status shows external IP
- Automatic cleanup with `kubectl delete svc`
- Standard Kubernetes LoadBalancer experience

## Related Issues

This fix is related to but distinct from the **cloudprovider taint issue**:

1. **Cloudprovider taint** (`node.cloudprovider.kubernetes.io/uninitialized:NoSchedule`)

   - Blocks ALL pod scheduling
   - Requires manual removal before installing addons
   - Fixed by: `kubectl taint nodes --all node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-`

2. **Missing CCM** (this issue)
   - Pods can run fine
   - LoadBalancer services never get external IPs
   - Fixed by: Installing DigitalOcean CCM

Both are consequences of Kubespray's `external_cloud_provider: digitalocean` configuration:

- Kubespray applies the taint expecting CCM to remove it → **We remove it manually**
- Kubespray expects CCM to be installed separately → **We install it now**

## Cost Consideration

Each LoadBalancer service creates a DigitalOcean Load Balancer, which costs **$12/month**.

In our setup:

- **ingress-nginx-controller**: 1 Load Balancer ($12/month)
  - Handles ALL HTTP/HTTPS traffic via Ingress resources
  - ArgoCD Ingress → routes to ArgoCD via this LB
  - Dashboard Ingress → routes to Dashboard via this LB
  - Any future Ingress → routes via this same LB

**Total cost: $12/month** (one Load Balancer for all Ingress traffic)

This is much more cost-effective than exposing each service directly as LoadBalancer (which would create multiple LBs).

## References

- [DigitalOcean Cloud Controller Manager GitHub](https://github.com/digitalocean/digitalocean-cloud-controller-manager)
- [DigitalOcean CCM Releases](https://github.com/digitalocean/digitalocean-cloud-controller-manager/releases)
- [Kubernetes Cloud Controller Manager Docs](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)
- [DigitalOcean Load Balancer API](https://docs.digitalocean.com/reference/api/api-reference/#tag/Load-Balancers)
- [Kubespray External Cloud Provider](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/kubernetes-apps/external-cloud-controller.md)

## Fix Applied

✅ **Fixed in commit**: Added DigitalOcean CCM installation tasks

- Created secret with DO_AUTH_TOKEN
- Downloaded CCM manifest v0.1.47
- Applied CCM DaemonSet
- Added rollout wait with retries
- Positioned before ingress-nginx installation

This ensures LoadBalancer services can be provisioned automatically via DigitalOcean API integration.
