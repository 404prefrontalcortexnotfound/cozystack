# Quick Reference: Cozystack Operations

## Environment Setup

```bash
# Set these in every shell session
export KUBECONFIG=~/code/cozy-stack/hetzner-cluster/kubeconfig
export TALOSCONFIG=~/code/cozy-stack/hetzner-cluster/talosconfig
```

## Cluster Health Checks

```bash
# Node status
kubectl get nodes

# System pods
kubectl get pods -n cozy-system

# All HelmReleases (show failures)
kubectl get helmrelease -A | grep -v True

# Talos node health
talosctl health

# etcd status
talosctl -n 77.42.88.245 etcd status
```

## Access URLs

| Service | URL |
|---------|-----|
| Dashboard | https://dashboard.cozy.homi.zone |
| Keycloak | https://keycloak.cozy.homi.zone |
| Tailscale Dashboard | https://cozy-dashboard-cozystack-dashboard-ingress.beagle-danio.ts.net |

## Credentials

### Keycloak Admin
```bash
kubectl get secret keycloak-credentials -n cozy-keycloak \
  -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d && echo
```

**User**: admin
**Password**: IKijfV5UtevFEdp7

## Common Operations

### Deploy Application (GitOps)

```bash
# 1. Create HelmRelease manifest
cat > app.yaml <<EOF
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: myapp
  namespace: tenant-media
spec:
  interval: 5m
  chart:
    spec:
      chart: myapp
      sourceRef:
        kind: GitRepository
        name: external-arr-stack
        namespace: cozy-public
  values:
    # ... configuration
EOF

# 2. Apply
kubectl apply -f app.yaml

# 3. Watch status
kubectl get helmrelease -n tenant-media myapp -w
```

### Check Application Logs

```bash
# List pods in namespace
kubectl get pods -n tenant-media

# View logs
kubectl logs -n tenant-media <pod-name> -f

# Previous crashed container
kubectl logs -n tenant-media <pod-name> --previous
```

### Storage Operations

```bash
# List PVCs
kubectl get pvc -n tenant-media

# Check PVC details
kubectl describe pvc media-pvc -n tenant-media

# List storage classes
kubectl get storageclass
```

### Ingress & DNS

```bash
# List ingresses
kubectl get ingress -A

# Check ingress details
kubectl describe ingress -n tenant-media arr-stack-sonarr

# Test DNS
dig sonarr.cozy.homi.zone

# Check certificate
kubectl get certificate -n tenant-media
```

### FluxCD Operations

```bash
# Force GitRepository sync
flux reconcile source git external-arr-stack -n cozy-public

# Force HelmRelease reconcile
flux reconcile helmrelease arr-stack -n tenant-media

# Check FluxCD logs
kubectl logs -n cozy-fluxcd deploy/source-controller -f
kubectl logs -n cozy-fluxcd deploy/helm-controller -f
```

### Troubleshooting

```bash
# Describe resource for events
kubectl describe pod <pod-name> -n tenant-media

# Get all events in namespace
kubectl get events -n tenant-media --sort-by='.lastTimestamp'

# Check resource usage
kubectl top nodes
kubectl top pods -n tenant-media

# Restart deployment
kubectl rollout restart deployment <name> -n tenant-media

# Delete stuck pod
kubectl delete pod <pod-name> -n tenant-media --force --grace-period=0
```

## Talos Operations

```bash
# Get kubeconfig
talosctl kubeconfig

# Node dashboard
talosctl -n 77.42.88.245 dashboard

# Service status
talosctl -n 77.42.88.245 services

# Container logs
talosctl -n 77.42.88.245 containers
talosctl -n 77.42.88.245 logs <container-id>

# Disk usage
talosctl -n 77.42.88.245 df

# Memory usage
talosctl -n 77.42.88.245 memory

# Upgrade Talos
talosctl upgrade --nodes 77.42.88.245 \
  --image ghcr.io/cozystack/cozystack/talos:v1.10.6
```

## Backup & Restore

### etcd Backup

```bash
# Create snapshot
talosctl -n 77.42.88.245 etcd snapshot backup.db

# List snapshots
talosctl -n 77.42.88.245 etcd snapshot ls
```

### Application Backup

```bash
# Export HelmRelease manifests
kubectl get helmrelease -n tenant-media -o yaml > helmreleases-backup.yaml

# Backup PVC data
kubectl exec -n tenant-media <pod-name> -- tar czf - /config > config-backup.tar.gz
```

## Git Operations

```bash
cd ~/code/cozy-stack

# Check status
git status

# Commit changes
git add packages/apps/arr-stack/
git commit -m "feat: arr-stack updates"

# Push to fork
git push origin main

# Sync from upstream
git fetch upstream
git merge upstream/main
git push origin main
```

## Node Information

| Node | Role | IP | Tailscale |
|------|------|----|-----------|
| cozy-ctrl-1 | Control | 77.42.88.245 | talos-16aa3 |
| cozy-ctrl-2 | Control | 77.42.79.46 | talos-909a8 |
| cozy-ctrl-3 | Control | 77.42.85.170 | talos-7003c |
| cozy-ax41 | Control+Worker | 65.21.129.221 | cozy-ctrl-4 |

## Useful Aliases

```bash
# Add to ~/.zshrc or ~/.bashrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes'
alias kga='kubectl get all -A'
alias kd='kubectl describe'
alias kl='kubectl logs -f'

alias tc='talosctl'
alias tcn='talosctl -n 77.42.88.245'

# Cozystack context
alias cozy-env='export KUBECONFIG=~/code/cozy-stack/hetzner-cluster/kubeconfig && export TALOSCONFIG=~/code/cozy-stack/hetzner-cluster/talosconfig'
```

## Emergency Procedures

### Cluster Unresponsive

```bash
# Check control plane nodes
talosctl -n 77.42.88.245,77.42.79.46,77.42.85.170 health

# Check etcd
talosctl -n 77.42.88.245 etcd status

# Restart Kubernetes API server
talosctl -n 77.42.88.245 service kube-apiserver restart

# Bootstrap etcd (DESTRUCTIVE - last resort)
talosctl bootstrap -n 77.42.88.245
```

### Pod Stuck Terminating

```bash
# Force delete
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0

# If still stuck, remove finalizers
kubectl patch pod <pod-name> -n <namespace> -p '{"metadata":{"finalizers":null}}'
```

### PVC Stuck Deleting

```bash
# Remove finalizers
kubectl patch pvc <pvc-name> -n <namespace> -p '{"metadata":{"finalizers":null}}'
```

## Documentation Links

- This repo: `~/code/cozy-stack/`
- GitOps plan: `gitops_plan.md`
- Terraform plan: `future_proofing.md`
- Status: `STATUS.md`
- Doc review: `DOCUMENTATION-REVIEW.md`

## Support Channels

- Telegram: @cozystack
- GitHub Issues: https://github.com/cozystack/cozystack/issues
- CNCF Slack: #cozystack
