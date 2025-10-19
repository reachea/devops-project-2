# Simplified Approach: NodePort + Manual Load Balancer

## Why the Change?

The previous approach tried to automate everything using DigitalOcean Cloud Controller Manager (CCM), but it was:

- ❌ Too complex (CCM installation, API token secrets, etc.)
- ❌ Error-prone (LoadBalancer provisioning failures)
- ❌ Slow to debug (1 hour per playbook run!)
- ❌ Hard to troubleshoot when things go wrong

## New Simplified Approach

Instead of fighting with automatic LoadBalancer provisioning:

1. **ingress-nginx uses NodePort** (not LoadBalancer type)
2. **Reuse the existing API Load Balancer** (already created for kube-apiserver)
3. **Manually add forwarding rules** (one-time, 2 minutes in DigitalOcean console)

### Benefits

✅ **Much simpler** - no CCM, no API integration  
✅ **More reliable** - manual configuration is predictable  
✅ **Easier to debug** - you control the Load Balancer directly  
✅ **Same cost** - reusing existing LB ($12/month total)  
✅ **Faster** - no waiting for CCM to provision LBs

## How It Works

### Architecture

```
Internet
    │
    ▼
┌─────────────────────────────┐
│ DigitalOcean Load Balancer  │ ◄── Reuse the one you already have!
│ IP: 146.190.123.45          │
│                             │
│ Forwarding Rules:           │
│  - 80  → 30080 (HTTP)       │ ◄── You add these manually
│  - 443 → 30443 (HTTPS)      │
└─────────────────────────────┘
              │
              ▼
    ┌─────────────────┐
    │  All 3 Nodes    │
    │  (backends)     │
    └─────────────────┘
              │
              ▼
┌──────────────────────────────┐
│ ingress-nginx-controller     │
│ Type: NodePort               │
│  - 30080 (HTTP)              │ ◄── NodePorts (fixed ports on all nodes)
│  - 30443 (HTTPS)             │
└──────────────────────────────┘
              │
              ▼
    ┌─────────────────┐
    │  Ingress Rules  │
    └─────────────────┘
          │       │
          ▼       ▼
    ┌───────┐ ┌──────────┐
    │ArgoCD │ │Dashboard │
    └───────┘ └──────────┘
```

### Traffic Flow

1. **User accesses** `https://argocd.reacheasambath.com`
2. **DNS resolves** to Load Balancer IP: `146.190.123.45`
3. **LB forwards** port 443 → NodePort 30443 on any node
4. **NodePort routes** to ingress-nginx pod
5. **Ingress rule** matches `argocd.reacheasambath.com` → forwards to ArgoCD service
6. **ArgoCD service** routes to ArgoCD pods

## Manual Steps Required

### Step 1: Get NodePort Information

After the playbook runs, it will show you:

```
HTTP NodePort:  30080
HTTPS NodePort: 30443
Load Balancer IP: 146.190.123.45
```

### Step 2: Configure Load Balancer

1. **Go to DigitalOcean Console**

   - Navigate to: Networking → Load Balancers
   - Click on your API Load Balancer (the one with IP shown in playbook output)

2. **Add Forwarding Rules**

   - Click "Settings" → "Forwarding Rules" → "Edit"
   - Add two new rules:

     ```
     Protocol: HTTP
     Port: 80
     Target Protocol: HTTP
     Target Port: 30443  ← (yes, redirect HTTP to HTTPS NodePort)

     Protocol: HTTPS
     Port: 443
     Target Protocol: HTTPS
     Target Port: 30443
     ```

   - Click "Save"

3. **Verify Backend Health**
   - Load Balancer should show all 3 nodes as healthy backends
   - Health check on the existing API port (6443) is fine

### Step 3: Create DNS Records

In your DNS provider (where reacheasambath.com is hosted):

```
Type: A
Name: argocd
Value: 146.190.123.45  ← (your LB IP)
TTL: 300

Type: A
Name: dashboard
Value: 146.190.123.45  ← (same LB IP)
TTL: 300
```

### Step 4: Wait for DNS Propagation

```bash
# Test DNS resolution
nslookup argocd.reacheasambath.com
nslookup dashboard.reacheasambath.com
```

Should return your Load Balancer IP.

### Step 5: Test Access

After DNS propagates (5-10 minutes):

```bash
# Test ArgoCD
curl -k https://argocd.reacheasambath.com

# Test Dashboard
curl -k https://dashboard.reacheasambath.com
```

Both should return HTTP responses (might be redirects or HTML, but not connection errors).

## Comparison: Old vs New Approach

### Old Approach (CCM-based)

```yaml
# 1. Install CCM with API token secret
- Create Secret with DO_AUTH_TOKEN
- Download CCM manifest
- Apply CCM DaemonSet
- Wait for CCM to be ready

# 2. Create LoadBalancer service
- Apply ingress-nginx (type: LoadBalancer)
- Wait for CCM to detect it
- CCM calls DigitalOcean API
- DigitalOcean provisions new LB ($12/month)
- Wait 2-5 minutes for LB to be ready
- CCM updates Service with external IP
- Retry loop waiting for IP assignment
# RESULT: Complex, slow, error-prone
```

### New Approach (NodePort)

```yaml
# 1. Create NodePort service
- Apply ingress-nginx (type: NodePort)
- Get NodePort numbers

# 2. Manual LB configuration (one-time, 2 minutes)
- Add forwarding rules in DO console
- Create DNS A records
# RESULT: Simple, fast, reliable
```

## Cost Analysis

### Old Approach

- API Load Balancer: $12/month (for kube-apiserver)
- **Ingress Load Balancer: $12/month** (auto-created by CCM)
- **Total: $24/month**

### New Approach

- API Load Balancer: $12/month (for kube-apiserver **AND** ingress)
- **Total: $12/month** (saves $12/month!)

## Troubleshooting

### NodePort Not Accessible

Check if ingress-nginx pods are running:

```bash
kubectl -n ingress-nginx get pods
```

### Load Balancer Health Checks Failing

The LB might health-check port 6443 (API) - this is OK. As long as nodes are reachable, traffic will route to NodePorts.

Alternatively, change health check to use the HTTPS NodePort:

```
Protocol: TCP
Port: 30443
```

### DNS Not Resolving

Wait 5-10 minutes for propagation, or:

```bash
# Test directly with IP (bypass DNS)
curl -k -H "Host: argocd.reacheasambath.com" https://146.190.123.45
```

### HTTPS Certificate Errors

Expected initially! cert-manager needs DNS to work:

1. DNS must resolve correctly
2. Let's Encrypt will verify domain ownership
3. Certificates will be issued (2-5 minutes)

Check certificate status:

```bash
kubectl get certificate -A
kubectl describe certificate -n argocd argocd-tls
```

## Why This is Better for You

Given your feedback:

- **"does it has to be this complicated?"** → No! This is much simpler
- **"everytime I run the scripts it took almost 1 hour"** → No more waiting for CCM/LB provisioning

The 2-minute manual configuration is worth it to avoid:

- Complex CCM setup
- API token management
- Debugging LoadBalancer provisioning failures
- Long retry loops waiting for external IPs
- Cloud provider integration issues

## Next Steps

1. **Run the playbook** - it will complete faster now (no LoadBalancer waiting)
2. **Follow the output instructions** - add LB forwarding rules and DNS records
3. **Access your services** - ArgoCD and Dashboard should work!

The playbook will pause and show you exactly what to do. Just follow the instructions, and you're done! 🎉
