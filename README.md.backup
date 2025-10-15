# DO + Kubespray — Single Playbook

Run:
```bash
export DO_TOKEN=your_token
ansible-playbook create_k8s.yml
```

Notes:
- The playbook **does not** create DNS automatically anymore.
- After ingress gets an external IP, you will be prompted to add:
  - `argocd_domain` -> <ingress_ip>
  - `dashboard_domain` -> <ingress_ip>
- Then press Enter to continue (or it will wait for `dns_wait_seconds` if `dns_wait_mode: seconds`).

You’ll also see printed lists of master/worker IPs right after provisioning.
