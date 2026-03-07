# k3s Homelab Ansible Playbook

Deploys a 3-node k3s cluster (all control-plane, embedded etcd) with:
- **Calico** CNI (eBPF + BGP ready, off by default)
- **Longhorn** distributed storage on dedicated aux disk (/dev/sdb)
- **ingress-nginx** (DaemonSet, hostNetwork — no LoadBalancer required)
- **Kubernetes Dashboard**

## Cluster topology

| Node   | Internal (cluster)   | Storage (Longhorn/NFS) | Proxmox host |
|--------|----------------------|------------------------|--------------|
| k3s211 | 192.168.x.211       | 192.168.1xx.211        | pve201       |
| k3s212 | 192.168.x.212       | 192.168.1xx.212        | pve202       |
| k3s213 | 192.168.x.213       | 192.168.1xx.213        | pve203       |

- **vmbr42**  → Internal (cluster traffic, API, Calico, pod network)
- **vmbr142** → Storage (Longhorn replication, NFS/iSCSI to TrueNAS)

## Prerequisites

### On your Ansible controller
```bash
pip install ansible
ansible-galaxy collection install community.general
```

### Verify SSH access
```bash
ansible -i inventory/hosts.yaml k3s_cluster -m ping
```

### Confirm /dev/sdb is present and unformatted on each VM
```bash
ansible -i inventory/hosts.yaml k3s_cluster -m command -a "lsblk /dev/sdb" --become
```

## Deployment

### Full deploy (recommended first run)
```bash
ansible-playbook -i inventory/hosts.yaml site.yaml
```

### Individual phases (useful for re-runs or troubleshooting)
```bash
ansible-playbook -i inventory/hosts.yaml site.yaml --tags prereqs
ansible-playbook -i inventory/hosts.yaml site.yaml --tags k3s
ansible-playbook -i inventory/hosts.yaml site.yaml --tags calico
ansible-playbook -i inventory/hosts.yaml site.yaml --tags longhorn
ansible-playbook -i inventory/hosts.yaml site.yaml --tags ingress
ansible-playbook -i inventory/hosts.yaml site.yaml --tags dashboard
```

## Post-deploy

### Set up kubeconfig
```bash
export KUBECONFIG=/tmp/k3s-homelab.yaml
# or merge into your existing config:
cp /tmp/k3s-homelab.yaml ~/.kube/config
```

### Verify cluster
```bash
kubectl get nodes -o wide
kubectl get pods -A
```

### Access the dashboard
Point `dashboard.internal` at any of `192.168.x.211-213` in your OPNsense
host overrides (or local DNS), then open `https://dashboard.internal`.
The login token is printed at the end of the playbook run. To retrieve it later:
```bash
kubectl get secret dashboard-admin-token \
  -n kubernetes-dashboard \
  -o jsonpath='{.data.token}' | base64 -d
```

### Access Longhorn UI (via kubectl proxy)
```bash
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
# open http://localhost:8080
```

## Next steps (not in this playbook)

### Enable Calico eBPF dataplane
1. Disable kube-proxy: `kubectl -n kube-system patch ds kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico":"true"}}}}}'`
2. Edit the Installation CR: set `spec.calicoNetwork.linuxDataplane: BPF`
3. See Calico docs: https://docs.tigera.io/calico/latest/operations/ebpf/enabling-ebpf

### Enable Calico BGP peering with OPNsense
1. Install FRR plugin on OPNsense
2. Configure BGP on OPNsense (AS 65000 suggested)
3. Apply BGPPeer and BGPConfiguration resources (templates in roles/calico/tasks/main.yaml comments)
4. Set `calico_bgp_enabled: true` in group_vars/all.yaml and re-run `--tags calico`

## Vault setup (required before first run)

Fill in your secrets, then optionally encrypt:
```bash
# Edit vault/secrets.yaml and fill in real values, then:
ansible-vault encrypt vault/secrets.yaml

# Run playbooks with vault:
ansible-playbook -i inventory/hosts.yaml site.yaml --ask-vault-pass
# or with a password file:
echo "yourpassword" > ~/.vault_pass && chmod 600 ~/.vault_pass
ansible-playbook -i inventory/hosts.yaml site.yaml --vault-password-file ~/.vault_pass
```

The `.gitignore` blocks `vault/secrets.yaml` from being committed unencrypted.
Once encrypted with `ansible-vault`, it is safe to commit.

## Cloudflare scoped API token

Create at https://dash.cloudflare.com/profile/api-tokens → "Create Custom Token":
- Zone > DNS > Edit — zone: homelab.internal
- Zone > Zone > Read — zone: homelab.internal

Paste the token into `vault/secrets.yaml` as `cloudflare_api_token`.

## OPNsense API credentials

OPNsense GUI → System → Access → Users → your user → API keys.
User needs the **Unbound DNS** privilege minimum.
Paste key/secret into `vault/secrets.yaml`.

## cert-manager / TLS workflow

1. Run `--tags cert-manager` after Calico is healthy
2. Deploy services with annotation: `cert-manager.io/cluster-issuer: letsencrypt-staging`
3. Check cert status: `kubectl get certificate -A`
4. Once staging cert issues successfully, switch to `letsencrypt-production`
5. Delete old staging secret to force re-issue: `kubectl delete secret <name>-tls -n <namespace>`

## Maintenance

Rolling updates across all nodes (one at a time, cluster-safe):
```bash
ansible-playbook -i inventory/hosts.yaml maintain.yaml
```

Single node only:
```bash
ansible-playbook -i inventory/hosts.yaml maintain.yaml --limit k3s211
```

The maintain role handles three states automatically:
- **Node has a broken/partial k3s install** → runs `k3s-uninstall.sh`, reboots, continues
- **Node is in a running cluster with updates** → cordons, drains, upgrades, reboots, uncordons
- **Node is up to date** → apt housekeeping only, no reboot

---

### Add TrueNAS NFS/iSCSI storage class
Mount NFS shares from TrueNAS on the storage network (192.168.1xx.x).
Consider the `nfs-subdir-external-provisioner` Helm chart for dynamic NFS provisioning.

### Add Caddy reverse proxy in front of ingress
Run Caddy on OPNsense or a small VM to handle TLS termination and forward to
`192.168.x.211-213:443`. This gives you automatic ACME certs without
exposing the cluster ingress directly.
