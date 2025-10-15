# HA Kubernetes Cluster on DigitalOcean with Kubespray

Automated infrastructure provisioning and HA K8s cluster deployment using Ansible + Kubespray.

## 🎯 Project Overview

This project automates:

- **Infrastructure**: DigitalOcean droplets (control-plane + workers) with load balancers
- **K8s Installation**: HA Kubernetes cluster via Kubespray
- **Addons**: ingress-nginx, cert-manager, ArgoCD, Kubernetes Dashboard
- **TLS**: Automated Let's Encrypt certificates for services

## 📋 Architecture

```
Local Machine (Ansible)
    ↓
    ↓ (creates infrastructure)
    ↓
DigitalOcean Cloud
    ├─ Control Plane Nodes (1-3)
    ├─ Worker Nodes (0-N)
    ├─ API Load Balancer (kube-apiserver on port 6443)
    └─ Ingress Load Balancer (auto-created by ingress-nginx)
```

## 📦 Prerequisites

1. **DigitalOcean Account**

   - API token with read/write access
   - SSH key added to your DO account (note the numeric ID)

2. **Local Machine**

   - Ubuntu/Debian (or WSL2 on Windows)
   - Python 3.8+
   - Ansible 2.9+ (installed automatically by playbook)

3. **DNS Access**
   - Ability to create A/CNAME records for your domain

## 🚀 Quick Start

### 1. Clone & Configure

```bash
cd /path/to/devops-project-2
```

### 2. Set Environment Variables

```bash
export DO_TOKEN="your_digitalocean_api_token_here"
```

### 3. Edit Configuration

Edit `group_vars/all.yml`:

```yaml
# Update these values:
ssh_key_id: YOUR_SSH_KEY_ID # From DigitalOcean → Settings → Security → SSH Keys
base_domain: "yourdomain.com"
dashboard_domain: "dashboard.yourdomain.com"
argocd_domain: "argocd.yourdomain.com"
email_acme: "your-email@yourdomain.com"

# Adjust cluster size:
cp_count: 1 # Control plane nodes (1 or 3 for HA)
worker_count: 2 # Worker nodes (0+ as needed)
```

### 4. Run the Playbook

```bash
ansible-playbook create_k8s.yml
```

**Duration**: ~20-30 minutes depending on cluster size and network speed.

### 5. Configure DNS

When prompted (or after the playbook finishes), create DNS records:

```
# Get the Ingress LB address from playbook output
A/CNAME records:
  dashboard.reacheasambath.com → <INGRESS_LB_IP>
  argocd.reacheasambath.com    → <INGRESS_LB_IP>
```

### 6. Access Your Services

**Kubernetes Dashboard**:

- URL: `https://dashboard.reacheasambath.com`
- Token: Printed at end of playbook output

**ArgoCD**:

- URL: `https://argocd.reacheasambath.com`
- Username: `admin`
- Password: Printed at end of playbook output

**kubectl**:

```bash
export KUBECONFIG=~/.kube/config
kubectl get nodes -o wide
```

## 🗑️ Cleanup

To destroy all infrastructure:

```bash
ansible-playbook destroy_infra.yml
```

To also delete local kubeconfig:

```bash
ansible-playbook destroy_infra.yml -e delete_local_kubeconfig=true
```

## 📁 Project Structure

```
.
├── create_k8s.yml          # Main provisioning playbook
├── destroy_infra.yml       # Cleanup/teardown playbook
├── group_vars/
│   └── all.yml             # Configuration variables
└── README.md               # This file
```

## 🔧 Customization

### Change Kubernetes Version

Edit `create_k8s.yml`:

```yaml
vars:
  k8s_version: "v1.29.6" # Change to desired version
```

### Change Node Sizes

Edit `group_vars/all.yml`:

```yaml
size_cp: "s-2vcpu-4gb" # Control plane droplet size
size_worker: "s-2vcpu-4gb" # Worker droplet size
```

Available sizes: `s-1vcpu-1gb`, `s-2vcpu-2gb`, `s-2vcpu-4gb`, `s-4vcpu-8gb`, etc.

### Change Region

Edit `group_vars/all.yml`:

```yaml
region: "sgp1" # Singapore
# Other options: nyc1, nyc3, sfo3, lon1, fra1, tor1, blr1, etc.
```

## 🛠️ What Gets Installed

### Infrastructure

- ✅ DigitalOcean droplets (Ubuntu 22.04)
- ✅ Load Balancer for kube-apiserver (HA)
- ✅ Load Balancer for Ingress (auto-provisioned)

### Kubernetes Components

- ✅ HA Kubernetes cluster via Kubespray
- ✅ Calico CNI for networking
- ✅ etcd (HA on control plane nodes)

### Add-ons

- ✅ **ingress-nginx**: Ingress controller with cloud LB
- ✅ **cert-manager**: Automated TLS certificates (Let's Encrypt)
- ✅ **ArgoCD**: GitOps continuous delivery
- ✅ **Kubernetes Dashboard**: Web UI for cluster management

## 🔐 Security Notes

- All services exposed via HTTPS with Let's Encrypt certificates
- SSH access: only via SSH keys (no passwords)
- Dashboard access: token-based authentication
- ArgoCD: change default admin password after first login

## 📊 Monitoring

DigitalOcean monitoring is enabled by default on all droplets. Access metrics via DO Console → Droplets → Monitoring.

## 🐛 Troubleshooting

### Playbook fails during cert-manager deployment

The playbook includes auto-remediation for ImagePullBackOff issues. If it still fails:

```bash
kubectl -n cert-manager get pods
kubectl -n cert-manager logs deploy/cert-manager
```

### Cannot access ArgoCD/Dashboard

1. Verify DNS records are correct: `nslookup dashboard.reacheasambath.com`
2. Check Ingress LB status: `kubectl -n ingress-nginx get svc`
3. Check cert-manager: `kubectl get certificate -A`

### kubectl connection issues

Verify kubeconfig points to correct LB address:

```bash
cat ~/.kube/config | grep server:
```

## 📚 References

- [Kubespray Documentation](https://kubespray.io/)
- [DigitalOcean Kubernetes](https://docs.digitalocean.com/products/kubernetes/)
- [cert-manager](https://cert-manager.io/)
- [ArgoCD](https://argo-cd.readthedocs.io/)

## 📝 License

MIT
