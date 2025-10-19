# Quick Reference: Simplified Setup

## Changes Made

### Removed ❌

- DigitalOcean Cloud Controller Manager (CCM) installation
- LoadBalancer type services
- Waiting for external IP assignment (10+ minutes of retries)

### Added ✅

- ingress-nginx with NodePort (simpler, more reliable)
- Manual Load Balancer configuration instructions
- Clear step-by-step prompts in playbook output

## What You Need to Do

### 1. Run the Playbook

```bash
ansible-playbook create_k8s.yml
```

It will be **much faster** now - no more waiting for LoadBalancers to provision!

### 2. When Playbook Pauses

It will show you something like:

```
HTTP NodePort:  30080
HTTPS NodePort: 30443
Load Balancer IP: 146.190.123.45

MANUAL CONFIGURATION REQUIRED:
1. Go to DigitalOcean -> Load Balancers
2. Edit your API Load Balancer
3. Add forwarding rules:
   - HTTP: 80 → 30443
   - HTTPS: 443 → 30443
```

### 3. Configure Load Balancer (2 minutes)

1. Open DigitalOcean console
2. Go to Networking → Load Balancers
3. Click on your load balancer
4. Settings → Forwarding Rules → Edit
5. Add:
   - **HTTP** port 80 → target port 30443
   - **HTTPS** port 443 → target port 30443
6. Save

### 4. Create DNS Records

In your DNS provider:

```
argocd.reacheasambath.com    A    146.190.123.45
dashboard.reacheasambath.com A    146.190.123.45
```

(Use the IP shown in playbook output)

### 5. Press Enter in Playbook

After configuring LB and DNS, press Enter to continue.

## Why This is Better

| Aspect          | Old Approach                       | New Approach              |
| --------------- | ---------------------------------- | ------------------------- |
| **Complexity**  | CCM + API tokens + secrets         | NodePort + manual config  |
| **Reliability** | LoadBalancer provisioning can fail | NodePort always works     |
| **Debug Time**  | Hard to troubleshoot CCM issues    | Easy - you control the LB |
| **Cost**        | $24/month (2 LBs)                  | $12/month (1 LB)          |
| **Setup Time**  | 1+ hour per run                    | ~30 min + 2 min manual    |

## Verification

After setup, test:

```bash
# Check NodePort service
kubectl -n ingress-nginx get svc ingress-nginx-controller

# Check DNS
nslookup argocd.reacheasambath.com

# Test access (after DNS propagates)
curl -k https://argocd.reacheasambath.com
curl -k https://dashboard.reacheasambath.com
```

## Need Help?

See `SIMPLIFIED_APPROACH.md` for detailed explanation and troubleshooting.

## Bottom Line

**You do 2 minutes of manual work to save yourself from:**

- Complex automation that takes 1 hour to debug
- Unreliable cloud provider integration
- Confusing error messages
- $12/month extra cost

**This is a deliberate trade-off: simplicity and reliability over full automation.** 🎯
