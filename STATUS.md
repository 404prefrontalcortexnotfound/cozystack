# Cozystack Infrastructure Status

**Updated**: 2026-01-22
**Cluster**: cozystack (Finland/HEL1)

## Cluster Health

| Node | Role | Status | IP | Tailscale Hostname |
|------|------|--------|----|--------------------|
| cozy-ctrl-1 | Control Plane | ✅ Running | 77.42.88.245 | talos-16aa3 |
| cozy-ctrl-2 | Control Plane | ✅ Running | 77.42.79.46 | talos-909a8 |
| cozy-ctrl-3 | Control Plane | ✅ Running | 77.42.85.170 | talos-7003c |
| cozy-ax41 | Control + Worker | ✅ Running | 65.21.129.221 | cozy-ctrl-4 |

**etcd**: 4-node quorum healthy
**KubeSpan**: Enabled (encrypted mesh)
**Services**: Linstor, Network, Metrics all healthy

## Access Configuration

```bash
# Set these in your shell
export KUBECONFIG=~/code/cozy-stack/hetzner-cluster/kubeconfig
export TALOSCONFIG=~/code/cozy-stack/hetzner-cluster/talosconfig
```

## Dashboard & Authentication

**Dashboard**: https://dashboard.cozy.homi.zone (Keycloak OIDC)
**Keycloak**: https://keycloak.cozy.homi.zone (admin console)
**Tailscale**: https://cozy-dashboard-cozystack-dashboard-ingress.beagle-danio.ts.net

**Keycloak Credentials**:
- User: admin
- Password: IKijfV5UtevFEdp7

## Completed Work

### Infrastructure
- [x] 3-node control plane cluster deployed
- [x] KubeSpan mesh networking enabled
- [x] Tailscale integration configured
- [x] Dashboard secured with Keycloak OIDC
- [x] DNS configured (*.cozy.homi.zone)
- [x] TLS certificates automated (Let's Encrypt)

### Repository Setup
- [x] Fork created: 404prefrontalcortexnotfound/cozystack
- [x] Local clone synced with fork
- [x] Arr stack Helm chart created

### Documentation
- [x] GitOps deployment plan created
- [x] Future Terraform strategy documented
- [x] Cluster access procedures documented

## Work in Progress

### Blocked: Platform Patch
**Issue**: Docker push failed due to missing `write:packages` scope on GitHub token

**Context**: Modified Cozystack platform source to remove "Universal Login" feature. Changes compiled locally (`cozystack-api` binary exists), but cannot push custom image to GitHub Container Registry.

**Resolution Path**:
1. Generate new GitHub token with `write:packages` scope
2. Authenticate Docker: `echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin`
3. Build and push: `make image` or manual `docker build/push`

**Alternative**: Skip platform modification, deploy Arr stack with standard Cozystack distribution.

### Ready: Arr Stack Deployment

**Status**: Chart created, awaiting push to fork

**Next Steps** (following GitOps plan):
1. Commit and push arr-stack package
2. Create tenant-media namespace
3. Create shared media PVC (500GB)
4. Register GitRepository + HelmRelease in cozy-public
5. Deploy arr-stack to tenant-media namespace
6. Configure DNS for sonarr/radarr/prowlarr.cozy.homi.zone

**Files**:
- Chart: `packages/apps/arr-stack/Chart.yaml`
- Values: `packages/apps/arr-stack/values.yaml`
- Plan: `gitops_plan.md`

## File Locations

| File | Path |
|------|------|
| Kubeconfig | `~/code/cozy-stack/hetzner-cluster/kubeconfig` |
| Talosconfig | `~/code/cozy-stack/hetzner-cluster/talosconfig` |
| Node configs | `~/code/cozy-stack/hetzner-cluster/nodes/` |
| Arr stack chart | `~/code/cozy-stack/packages/apps/arr-stack/` |
| GitOps plan | `~/code/cozy-stack/gitops_plan.md` |
| Terraform strategy | `~/code/cozy-stack/future_proofing.md` |

## Quick Commands

```bash
# Check cluster health
kubectl get nodes
kubectl get helmrelease -A | grep -v True

# Check Cozystack services
kubectl get pods -n cozy-system

# View Keycloak admin password
kubectl get secret keycloak-credentials -n cozy-keycloak \
  -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d

# Check Tailscale ingresses
kubectl get ingress -A | grep tailscale
```

## Recommended Next Actions

1. **Deploy Arr Stack** (GitOps approach):
   - Follow `gitops_plan.md`
   - Commit arr-stack to fork
   - Apply FluxCD manifests
   - Verify deployment

2. **Skip Platform Patch** (for now):
   - Standard Cozystack works fine
   - Custom dashboard modifications non-critical
   - Revisit when needed

3. **Future: Terraform Migration**:
   - Codify cluster configuration
   - Enable reproducible deployments
   - Follow `future_proofing.md`

## Resources

- **Project docs**: `~/code/cozy-stack/.planning/PROJECT.md`
- **Cozystack docs**: https://cozystack.io/docs/
- **GitOps example**: https://github.com/aenix-io/cozystack-gitops-example
- **External apps guide**: https://cozystack.io/docs/applications/external/
