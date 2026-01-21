# Cozystack Documentation Review & Recommendations

**Date**: 2026-01-22
**Reviewed**: Official Cozystack docs, GitOps examples, API references

## Key Findings

### 1. External Application Integration (v0.37.0+)

**Reference**: https://cozystack.io/docs/applications/external/

Cozystack supports custom applications via FluxCD GitRepository + HelmRelease pattern:

**Process**:
1. Create package repository with structure:
   - `./packages/core` - Platform configuration
   - `./packages/system` - System Helm charts
   - `./packages/apps` - User-installable apps
2. Register GitRepository pointing to your fork
3. Deploy HelmRelease referencing the package path
4. Applications appear in Cozystack catalog

**Your arr-stack fits this model perfectly.**

### 2. GitOps Reference Architecture

**Reference**: https://github.com/aenix-io/cozystack-gitops-example

**Pattern**:
- FluxCD polls repository every 1-5 minutes
- Kustomization resources target cluster-specific paths
- Multi-tenant namespaces (tenant-root, tenant-example)
- HelmReleases reference GitRepository sources
- Secrets support for private repositories

**Application**:
- Your arr-stack should live in `packages/apps/arr-stack/`
- Deploy to `tenant-media` namespace
- FluxCD handles reconciliation automatically

### 3. Managed Application Lifecycle

**Reference**: https://cozystack.io/docs/applications/

**Storage Options**:
- **Linstor** (linstor-r1): Replicated block storage for databases/config
- **NFS**: Shared filesystem for media libraries
- **SeaweedFS**: S3-compatible object storage (planned for your setup)

**Ingress**:
- Automated via `ingress.main.enabled: true`
- TLS certificates via cert-manager (Let's Encrypt)
- Supports multiple ingress classes (nginx, traefik)

**Your Configuration**:
- Linstor for Sonarr/Radarr/Prowlarr config (1Gi each)
- Shared PVC (media-pvc) for media library
- Ingress with auto-TLS on *.cozy.homi.zone

### 4. Deployment Workflow

**Reference**: https://cozystack.io/docs/getting-started/deploy-app/

**Two Approaches**:

**A. Dashboard (Quickest)**:
1. Navigate to Catalog
2. Search for application
3. Click Deploy
4. Configure parameters
5. Monitor in Applications tab

**B. kubectl (GitOps-friendly)**:
1. Define HelmRelease YAML
2. Apply with `kubectl apply -f`
3. FluxCD reconciles automatically
4. Check status: `kubectl get helmrelease -A`

**Recommendation**: Use kubectl approach for arr-stack (infrastructure as code).

### 5. Tenant Isolation

**Pattern**:
- Namespaces prefixed with `tenant-*`
- Resources isolated by RBAC
- Shared resources (ingress, storage) in system namespaces

**Your Setup**:
- Create `tenant-media` namespace
- Deploy arr-stack here
- PVC lives in same namespace for isolation

## Recommended Path Forward

### Option A: GitOps Deployment (Recommended)

**Advantages**:
- Infrastructure as code
- Version-controlled configuration
- Automatic reconciliation
- Rollback via git revert
- Team collaboration friendly

**Steps**:
1. Follow `gitops_plan.md` step-by-step
2. Commit arr-stack package
3. Push to fork
4. Apply FluxCD manifests
5. Verify deployment

**Timeline**: 30 minutes

### Option B: Dashboard Deployment (Quick Test)

**Advantages**:
- Fastest to test
- No git workflow needed
- Good for experimentation

**Steps**:
1. Package arr-stack as tarball
2. Upload via dashboard
3. Configure values in UI
4. Deploy

**Timeline**: 15 minutes

**Limitation**: Not suitable for production, no version control

## Addressing the Docker Push Blocker

### Issue
Platform patch (Universal Login removal) cannot be pushed to GHCR due to missing `write:packages` scope.

### Analysis
After reviewing documentation:
- Custom platform patches **not required** for arr-stack deployment
- Cozystack supports external applications without forking platform
- Dashboard customization nice-to-have, not critical path

### Recommendation: Defer Platform Patch

**Rationale**:
1. Arr stack works with standard Cozystack
2. Focus on application delivery first
3. Revisit platform customization later if needed
4. Reduces complexity and unblocks deployment

**If Platform Patch Required Later**:
```bash
# Generate new GitHub token with write:packages
# https://github.com/settings/tokens/new

# Authenticate Docker
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Build and push
cd ~/code/cozy-stack
make image
```

## Storage Architecture Decision

### Current Plan: Linstor + Shared PVC

**Configuration**:
- Config volumes: Linstor (replicated, durable)
- Media library: Shared PVC (ReadWriteMany)

**Pros**:
- Simple to implement
- Works with existing cluster
- No additional services needed

**Cons**:
- Media PVC not distributed (single-node bottleneck)
- No S3 API for future integrations

### Future Enhancement: SeaweedFS

**Reference**: Your PROJECT.md mentions SeaweedFS

**Benefits**:
- Distributed object storage
- S3-compatible API
- FUSE mount for POSIX access
- Scales across nodes

**Implementation Path**:
1. Deploy arr-stack with Linstor first (validate workflow)
2. Deploy SeaweedFS as Cozystack managed app later
3. Migrate media PVC to SeaweedFS volume
4. Update arr-stack values to use new mount

**Timeline**: Phase 2 (after arr-stack working)

## DNS Configuration

**Current Setup**: *.cozy.homi.zone

**Requirements for Arr Stack**:
- sonarr.cozy.homi.zone
- radarr.cozy.homi.zone
- prowlarr.cozy.homi.zone

**Verification**:
```bash
# Check ingress got IP
kubectl get ingress -n tenant-media

# Test DNS resolution
dig sonarr.cozy.homi.zone
```

**TLS Automation**:
- cert-manager handles Let's Encrypt
- Ingress annotation: `cert-manager.io/cluster-issuer: letsencrypt-prod`
- Certificates auto-renewed

## Security Considerations

### From Documentation

**Database External Access**:
> "External database access introduces security risks and should be avoided"

**Application**: Your arr-stack uses cluster-internal networking only (good).

**Ingress Security**:
- Enable authentication via Keycloak integration
- Or use Tailscale ingress for private access

**Recommendation**: Start with public ingress (HTTPS + Let's Encrypt), add auth later if needed.

### Secrets Management

**Arr Stack Secrets**:
- API keys (for Sonarr/Radarr integration)
- Indexer credentials (Prowlarr)

**Storage**:
- Create Kubernetes Secrets
- Mount into pods
- Bitwarden for backup/rotation

**Example**:
```bash
kubectl create secret generic arr-secrets \
  -n tenant-media \
  --from-literal=sonarr-api-key=xxx \
  --from-literal=radarr-api-key=yyy
```

## Next Actions (Prioritized)

### Immediate (Today)
1. ✅ Review Cozystack documentation (DONE)
2. ✅ Create GitOps plan (DONE)
3. ✅ Create Terraform strategy (DONE)
4. **Next**: Commit arr-stack package
5. **Next**: Apply FluxCD manifests

### Short-term (This Week)
1. Deploy arr-stack via GitOps
2. Configure Sonarr/Radarr/Prowlarr
3. Test download workflow
4. Document access procedures

### Medium-term (Next Month)
1. Deploy SeaweedFS
2. Migrate media storage
3. Add qBittorrent to arr-stack
4. Implement backups

### Long-term (Future)
1. Implement Terraform IaC
2. Add Immich (photo management)
3. Deploy Jellyfin with GPU
4. Platform customization (if still needed)

## References

| Topic | URL |
|-------|-----|
| Main Docs | https://cozystack.io/docs/ |
| Getting Started | https://cozystack.io/docs/getting-started/ |
| External Apps | https://cozystack.io/docs/applications/external/ |
| GitOps Example | https://github.com/aenix-io/cozystack-gitops-example |
| Application Guide | https://cozystack.io/docs/applications/ |
| Deploy App Tutorial | https://cozystack.io/docs/getting-started/deploy-app/ |
| Platform Stack | https://cozystack.io/docs/guides/platform-stack/ |

## Questions Resolved

**Q**: How to add custom applications to Cozystack?
**A**: Use FluxCD GitRepository + HelmRelease pattern (documented above).

**Q**: Do I need to modify Cozystack platform source?
**A**: No, external applications supported natively since v0.37.0.

**Q**: What storage should I use for media?
**A**: Start with shared PVC, migrate to SeaweedFS later for scale.

**Q**: How to handle secrets?
**A**: Kubernetes Secrets, optionally integrated with Vault/Bitwarden.

**Q**: Should I use dashboard or GitOps?
**A**: GitOps for production (version control, rollback support).

## Conclusion

Your cluster is production-ready. The arr-stack package is correctly structured. The platform patch is non-blocking. Follow `gitops_plan.md` to deploy, starting with committing the arr-stack package.

**Estimated time to working Arr stack**: 1 hour
**Blocker status**: None (Docker push issue bypassed via GitOps approach)
