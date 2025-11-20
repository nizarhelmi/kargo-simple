# Multi-Cluster GitOps with Kargo & ArgoCD

## What This Demonstrates

A working **multi-cluster GitOps pipeline** using:
- **Kargo** for GitOps promotion workflows  
- **ArgoCD** for cross-cluster deployment
- **GKE Fleet + Connect Gateway** for secure cluster access across Google Cloud projects
- **Workload Identity** for authentication

## Current Status: ✅ WORKING

All applications deployed and healthy across both clusters:

```
CLUSTER    APPLICATION              STATUS
use1       guestbook-dev-use1       Synced/Healthy  
use1       guestbook-staging-use1   Synced/Healthy
use1       guestbook-prod-use1      Synced/Healthy
use4       guestbook-dev-use4       Synced/Healthy  
use4       guestbook-staging-use4   Synced/Healthy
use4       guestbook-prod-use4      Synced/Healthy
```

## Architecture

```
┌─────────────────────────────────┐    ┌─────────────────────────────────┐
│     Fleet Host Project          │    │    Target Cluster Project       │
│  sap-ems-central-monitoring-poc │    │   sap-ems-gap-sandbox-net      │
│                                 │    │                                 │
│  ┌─────────────────────────────┐│    │ ┌─────────────────────────────┐ │
│  │   Control Cluster (use4)    ││    │ │   Target Cluster (use1)     │ │
│  │                             ││    │ │                             │ │
│  │  ┌─────────┐ ┌────────────┐ ││    │ │  ┌─────────────────────────┐│ │
│  │  │  Kargo  │ │   ArgoCD   │ ││    │ │  │    Applications        ││ │
│  │  │         │ │            │ ││────┼─┼─▶│  • guestbook-dev       ││ │
│  │  │ Promote │ │ Deploy     │ ││    │ │  │  • guestbook-staging   ││ │
│  │  └─────────┘ └────────────┘ ││    │ │  │  • guestbook-prod      ││ │
│  └─────────────────────────────┘│    │ │  └─────────────────────────┘│ │
└─────────────────────────────────┘    │ └─────────────────────────────┘ │
                                       └─────────────────────────────────┘
         Connect Gateway: connectgateway.googleapis.com/.../gap-staging-use1
```

## Key Configurations

### 1. Projects & Clusters
- **Fleet Host**: `sap-ems-central-monitoring-poc` (manages fleet)
- **Control Cluster**: `gap-staging-use4` in `sap-ems-gap-sandbox` (hosts ArgoCD/Kargo)  
- **Target Cluster**: `gap-staging-use1` in `sap-ems-gap-sandbox-net` (hosts apps)

### 2. Service Account & Authentication
```yaml
Service Account: argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com
Workload Identity: ✅ Enabled on both clusters
Environment Variable: GOOGLE_SERVICE_ACCOUNT_NAME set on ArgoCD controller
```

### 3. IAM Roles (split across projects)
```yaml
Fleet Host Project:
  - roles/container.clusterViewer
  - roles/gkehub.gatewayEditor
  - roles/gkehub.viewer

Target Cluster Project:
  - roles/container.clusterViewer  
  - roles/container.developer
```

### 4. Critical: Network Configuration
**Issue**: Control cluster has network restrictions  
**Solution**: ArgoCD pods scheduled on `whitelist` nodepool with internet access

```yaml
nodeSelector:
  role: "whitelist"
tolerations:
- effect: "NoSchedule"
  key: "role"
  value: "whitelist"
```

### 5. APIs Enabled
```yaml
Fleet Host Project: connectgateway.googleapis.com, gkehub.googleapis.com
Target Project: connectgateway.googleapis.com  
```

## Repository Structure

```
kargo-simple/
├── README.md                    # This overview
├── docs/
│   ├── ESSENTIAL-CONFIG.md      # Critical configuration reference
│   └── ARGOCD-PATCHES.md       # Required patches for network restrictions
├── argocd/
│   ├── applications/            # ArgoCD app definitions (use1 & use4)
│   ├── clusters/               # Cluster configs (reference only)
│   └── patches/                # Critical patches for multi-cluster setup
│       ├── argocd-controller-patch.yaml
│       └── proxy-repository-credentials.yaml
├── kargo/                      # Kargo pipeline (gke-fleet namespace)
│   ├── project.yaml           # Project: gke-fleet  
│   ├── warehouse.yaml         # Registry: us-east4-docker.pkg.dev/.../guestbook
│   ├── stages.yaml            # dev → staging → prod
│   └── promotiontask.yaml     # Git-based promotion
├── base/                       # Base manifests
└── env/                        # Environment overlays (dev/staging/prod)
```

## Quick Validation

```bash
# Patches are already applied! Just verify status:

# Check all apps (should show Healthy/Synced)
kubectl get applications -n argocd | grep use

# Test Connect Gateway
kubectl exec -it argocd-application-controller-0 -n argocd -- \
  curl -s "https://connectgateway.googleapis.com/v1beta1/projects/90257023985/locations/us-east1/gkeMemberships/gap-staging-use1/api/v1/nodes"

# Check Kargo pipeline
kubectl get stages,freight -n gke-fleet

# Test promotion
kargo promote --stage=staging --project=gke-fleet
```

## Key Learnings

1. **Connect Gateway requires APIs on both projects** (fleet host AND target)
2. **Critical patches required** for network restrictions (see `docs/ARGOCD-PATCHES.md`)
3. **Environment variable GOOGLE_SERVICE_ACCOUNT_NAME** required for argocd-k8s-auth plugin
4. **Whitelist nodepool** needed for ArgoCD pods in restricted networks  
5. **Workload Identity must be enabled on clusters** before fleet registration
6. **IAM roles can be split across projects** for better security
7. **Proxy configuration** required for Git repository access in corporate networks

## Production Considerations

- ✅ **Security**: No service account keys, Workload Identity only
- ✅ **Network**: Supports private clusters with restrictions  
- ✅ **Cross-project**: Works across organizational boundaries
- ✅ **Scalable**: Fleet-based management for multiple clusters
- ✅ **Auditable**: GitOps with full audit trail

---

**Status**: Production-ready multi-cluster GitOps pipeline with enterprise security features.