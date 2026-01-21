# GitOps Deployment Plan: Arr Stack

## Overview

Deploy Sonarr, Radarr, and Prowlarr using Cozystack's GitOps pattern via FluxCD. This approach provides version-controlled, declarative infrastructure and automatic reconciliation.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  GitHub Repository: 404prefrontalcortexnotfound/cozystack│
│  └── packages/apps/arr-stack/                           │
│      ├── Chart.yaml                                     │
│      ├── values.yaml                                    │
│      └── templates/                                     │
│                                                         │
│  Applied to Cluster:                                    │
│  ├── GitRepository (cozy-public namespace)              │
│  ├── HelmRelease (cozy-system namespace)                │
│  └── Tenant Applications (tenant-media namespace)       │
└─────────────────────────────────────────────────────────┘
```

## Prerequisites

- [x] Cozystack cluster running (3 control nodes healthy)
- [x] Fork created: `404prefrontalcortexnotfound/cozystack`
- [x] Arr stack Helm chart created: `packages/apps/arr-stack/`
- [ ] Chart published to fork
- [ ] Tenant namespace created
- [ ] Shared media PVC created

## Step 1: Commit and Push Arr Stack Package

```bash
cd ~/code/cozy-stack

# Stage arr-stack package
git add packages/apps/arr-stack/

# Commit
git commit -m "feat: add arr-stack managed application

- Sonarr, Radarr, Prowlarr umbrella chart
- Linstor storage for config
- Shared media PVC mount

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"

# Push to fork
git push origin main
```

## Step 2: Create Tenant Namespace

Create `manifests/tenant-media.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-media
  labels:
    cozystack.io/tenant: "true"
```

Apply:
```bash
kubectl apply -f manifests/tenant-media.yaml
```

## Step 3: Create Shared Media PVC

Create `manifests/media-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: media-pvc
  namespace: tenant-media
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: linstor-r1  # Replicated Linstor storage
  resources:
    requests:
      storage: 500Gi  # Adjust based on media library size
```

Apply:
```bash
kubectl apply -f manifests/media-pvc.yaml
```

## Step 4: Register External Application Repository

Create `manifests/arr-stack-gitops.yaml`:

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: external-arr-stack
  namespace: cozy-public
spec:
  interval: 5m0s
  ref:
    branch: main
  timeout: 60s
  url: https://github.com/404prefrontalcortexnotfound/cozystack.git
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-arr-stack
  namespace: cozy-system
spec:
  interval: 10m
  targetNamespace: cozy-system
  chart:
    spec:
      chart: ./packages/apps/arr-stack
      sourceRef:
        kind: GitRepository
        name: external-arr-stack
        namespace: cozy-public
      version: '*'
```

Apply:
```bash
kubectl apply -f manifests/arr-stack-gitops.yaml
```

## Step 5: Deploy Arr Stack to Tenant

Once the application appears in the catalog, deploy via HelmRelease:

Create `manifests/arr-stack-deployment.yaml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: arr-stack
  namespace: tenant-media
spec:
  interval: 5m
  chart:
    spec:
      chart: arr-stack
      reconcileStrategy: Revision
      sourceRef:
        kind: GitRepository
        name: external-arr-stack
        namespace: cozy-public
      version: '0.1.0'
  values:
    global:
      ingress:
        main:
          enabled: true
          className: nginx
          annotations:
            cert-manager.io/cluster-issuer: letsencrypt-prod

    sonarr:
      ingress:
        main:
          hosts:
            - host: sonarr.cozy.homi.zone
              paths:
                - path: /
                  pathType: Prefix
      persistence:
        config:
          enabled: true
          storageClass: linstor-r1
          size: 1Gi
        media:
          enabled: true
          existingClaim: media-pvc
          mountPath: /media

    radarr:
      ingress:
        main:
          hosts:
            - host: radarr.cozy.homi.zone
              paths:
                - path: /
                  pathType: Prefix
      persistence:
        config:
          enabled: true
          storageClass: linstor-r1
          size: 1Gi
        media:
          enabled: true
          existingClaim: media-pvc
          mountPath: /media

    prowlarr:
      ingress:
        main:
          hosts:
            - host: prowlarr.cozy.homi.zone
              paths:
                - path: /
                  pathType: Prefix
      persistence:
        config:
          enabled: true
          storageClass: linstor-r1
          size: 1Gi
```

Apply:
```bash
kubectl apply -f manifests/arr-stack-deployment.yaml
```

## Step 6: Verify Deployment

```bash
# Check GitRepository sync
kubectl get gitrepository -n cozy-public external-arr-stack

# Check HelmRelease status
kubectl get helmrelease -n cozy-system external-arr-stack
kubectl get helmrelease -n tenant-media arr-stack

# Check pods
kubectl get pods -n tenant-media

# Check ingresses
kubectl get ingress -n tenant-media
```

## Access URLs

After DNS propagation:
- Sonarr: https://sonarr.cozy.homi.zone
- Radarr: https://radarr.cozy.homi.zone
- Prowlarr: https://prowlarr.cozy.homi.zone

## GitOps Workflow Benefits

**Declarative Configuration**: All changes committed to git
**Automatic Reconciliation**: FluxCD syncs every 5 minutes
**Version Control**: Full audit trail of infrastructure changes
**Rollback Support**: Revert git commits to rollback deployments
**Single Source of Truth**: Repository defines desired state

## Future Enhancements

- Add qBittorrent as dependency in arr-stack chart
- Configure Prowlarr indexers via ConfigMap
- Add Grafana dashboard for *arr metrics
- Implement backup CronJob for config PVCs
