# Essential Configuration Reference

## Current Deployment Status ✅

All applications are **Synced/Healthy** in both clusters:

### use1 Cluster (Remote - gap-staging-use1)
- `guestbook-dev-use1`: Synced/Healthy
- `guestbook-staging-use1`: Synced/Healthy  
- `guestbook-prod-use1`: Synced/Healthy

### use4 Cluster (Local - gap-staging-use4) 
- `guestbook-dev-use4`: Synced/Healthy
- `guestbook-staging-use4`: Synced/Healthy
- `guestbook-prod-use4`: Synced/Healthy

## Critical Configurations

### Service Account
```
Name: argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com
Type: Google Service Account (no JSON keys)
Authentication: Workload Identity
```

### **REQUIRED PATCHES** ⚠️
**STATUS: Already applied in current cluster ✅**

For reference, ArgoCD Controller has these patches applied:
```yaml
# Environment Variable (for Connect Gateway)
env:
- name: GOOGLE_SERVICE_ACCOUNT_NAME
  value: argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com

# Network Configuration (for restricted networks)
nodeSelector:
  role: "whitelist"
tolerations:
- effect: "NoSchedule"  
  key: "role"
  value: "whitelist"
```

**For new deployments, see**: `docs/ARGOCD-PATCHES.md`

### IAM Roles by Project

**Fleet Host Project** (`sap-ems-central-monitoring-poc`):
- `roles/container.clusterViewer`
- `roles/gkehub.gatewayEditor`  
- `roles/gkehub.viewer`

**Target Cluster Project** (`sap-ems-gap-sandbox-net`):
- `roles/container.clusterViewer`
- `roles/container.developer`

### Connect Gateway Configuration
```
Endpoint: connectgateway.googleapis.com/v1beta1/projects/90257023985/locations/us-east1/gkeMemberships/gap-staging-use1
Fleet Project: 90257023985 (sap-ems-central-monitoring-poc)
Membership: gap-staging-use1
```

### Network Requirements ⚠️
**HANDLED BY PATCHES** - Control cluster has network restrictions:
- Default nodes: No internet access
- Whitelist nodes: Internet access via proxy (`http://100.64.40.2:3128`)
- ArgoCD scheduled on whitelist nodepool via patches

### Proxy Configuration
```yaml
# Applied via argocd/patches/proxy-repository-credentials.yaml
proxy: http://100.64.40.2:3128
noProxy: ".svc,.cluster.svc,100.64.36.1,argocd-*"
```

### Required APIs
**Fleet Host Project**: `connectgateway.googleapis.com`, `gkehub.googleapis.com`  
**Target Project**: `connectgateway.googleapis.com`

## Quick Validation Commands

```bash
# Check ArgoCD apps
kubectl get applications -n argocd | grep guestbook

# Test Connect Gateway access  
kubectl exec -it argocd-application-controller-0 -n argocd -- \
  curl -s "https://connectgateway.googleapis.com/v1beta1/projects/90257023985/locations/us-east1/gkeMemberships/gap-staging-use1/api/v1/nodes"

# Check Kargo pipeline
kubectl get stages,freight -n gke-fleet

# Test promotion
kargo promote --stage=staging --project=gke-fleet
```

---
*Last validated: Working deployment confirmed across both clusters*