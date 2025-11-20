# API and IAM Requirements

This document lists all the Google Cloud APIs and IAM roles required for the multi-cluster GitOps setup.

## Google Cloud APIs

### Fleet Host Project (`sap-ems-central-monitoring-poc`)

The following APIs must be enabled in the fleet host project:

```bash
gcloud services enable container.googleapis.com \
  gkehub.googleapis.com \
  cloudresourcemanager.googleapis.com \
  iam.googleapis.com \
  connectgateway.googleapis.com \
  anthos.googleapis.com \
  --project=sap-ems-central-monitoring-poc
```

| API | Purpose | Required For |
|-----|---------|-------------|
| `container.googleapis.com` | GKE cluster management | Fleet registration, cluster operations |
| `gkehub.googleapis.com` | GKE Fleet management | Fleet membership, Connect Gateway |
| `cloudresourcemanager.googleapis.com` | Project resource management | Cross-project access |
| `iam.googleapis.com` | Identity and Access Management | Service account operations |
| `connectgateway.googleapis.com` | Connect Gateway service | Cross-project cluster access |
| `anthos.googleapis.com` | Anthos platform services | Fleet operations |

### Control Cluster Project (`sap-ems-gap-sandbox`)

```bash
gcloud services enable container.googleapis.com \
  gkehub.googleapis.com \
  artifactregistry.googleapis.com \
  --project=sap-ems-gap-sandbox
```

| API | Purpose | Required For |
|-----|---------|-------------|
| `container.googleapis.com` | GKE cluster management | Cluster hosting Kargo/ArgoCD |
| `gkehub.googleapis.com` | GKE Fleet management | Fleet operations |
| `artifactregistry.googleapis.com` | Container image storage | Image hosting for applications |

### Target Cluster Project (`sap-ems-gap-sandbox-net`)

```bash
gcloud services enable container.googleapis.com \
  gkehub.googleapis.com \
  connectgateway.googleapis.com \
  --project=sap-ems-gap-sandbox-net
```

| API | Purpose | Required For |
|-----|---------|-------------|
| `container.googleapis.com` | GKE cluster management | Target cluster operations |
| `gkehub.googleapis.com` | GKE Fleet management | Fleet membership |
| `connectgateway.googleapis.com` | Connect Gateway service | Receiving cross-project access |

## IAM Roles and Permissions

### Primary Service Account

**Service Account**: `argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com`

This service account is used by ArgoCD to access clusters via Connect Gateway.

#### Fleet Host Project Roles

```bash
# Grant roles in fleet host project
gcloud projects add-iam-policy-binding sap-ems-central-monitoring-poc \
  --member "serviceAccount:argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com" \
  --role "roles/gkehub.viewer"

gcloud projects add-iam-policy-binding sap-ems-central-monitoring-poc \
  --member "serviceAccount:argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com" \
  --role "roles/gkehub.gatewayEditor"

gcloud projects add-iam-policy-binding sap-ems-central-monitoring-poc \
  --member "serviceAccount:argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com" \
  --role "roles/container.clusterViewer"
```

| Role | Purpose | Permissions |
|------|---------|-------------|
| `roles/gkehub.viewer` | View Fleet resources | List fleet memberships, view cluster info |
| `roles/gkehub.gatewayEditor` | Connect Gateway operations | Access clusters via Connect Gateway |
| `roles/container.clusterViewer` | View cluster resources | Read cluster configurations |

#### Target Cluster Project Roles

```bash
# Grant roles in target cluster project
gcloud projects add-iam-policy-binding sap-ems-gap-sandbox-net \
  --member "serviceAccount:argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com" \
  --role "roles/container.developer"
```

| Role | Purpose | Permissions |
|------|---------|-------------|
| `roles/container.developer` | Manage cluster resources | Deploy, update, delete Kubernetes resources |

#### Workload Identity Binding

```bash
# Allow ArgoCD application controller to impersonate the Google service account
gcloud iam service-accounts add-iam-policy-binding \
  argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:sap-ems-central-monitoring-poc.svc.id.goog[argocd/argocd-application-controller]"
```

### Target Cluster Node Service Account

The GKE node service account needs access to pull container images.

```bash
# Grant Artifact Registry access to target cluster nodes
gcloud projects add-iam-policy-binding sap-ems-gap-sandbox \
  --member "serviceAccount:90608020739-compute@developer.gserviceaccount.com" \
  --role "roles/artifactregistry.reader"
```

| Role | Purpose | Permissions |
|------|---------|-------------|
| `roles/artifactregistry.reader` | Pull container images | Read images from Artifact Registry |

### Kubernetes RBAC

#### Target Cluster RBAC

ArgoCD service account needs cluster-admin permissions on target clusters:

```yaml
# File: argocd/patches/connect-gateway-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-fleet-access
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: User
  name: argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com
```

#### Control Cluster RBAC

ArgoCD application controller service account annotation:

```bash
kubectl annotate serviceaccount argocd-application-controller \
  --namespace argocd \
  iam.gke.io/gcp-service-account=argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com
```

## Required Permissions Matrix

| Operation | Service Account | Project | Role/Permission |
|-----------|----------------|---------|-----------------|
| View Fleet memberships | argocd-fleet-access | Fleet Host | roles/gkehub.viewer |
| Access Connect Gateway | argocd-fleet-access | Fleet Host | roles/gkehub.gatewayEditor |
| Deploy to target cluster | argocd-fleet-access | Target Cluster | roles/container.developer |
| Pull container images | Node SA (target) | Image Registry | roles/artifactregistry.reader |
| Impersonate Google SA | ArgoCD K8s SA | Fleet Host | roles/iam.workloadIdentityUser |

## Validation Commands

### Verify API Enablement

```bash
# Check APIs in fleet host project
gcloud services list --enabled --project=sap-ems-central-monitoring-poc \
  --filter="name:gkehub OR name:connectgateway OR name:container"

# Check APIs in target cluster project  
gcloud services list --enabled --project=sap-ems-gap-sandbox-net \
  --filter="name:gkehub OR name:connectgateway OR name:container"
```

### Verify IAM Roles

```bash
# Check service account roles in fleet host project
gcloud projects get-iam-policy sap-ems-central-monitoring-poc \
  --flatten="bindings[].members" \
  --format="table(bindings.role)" \
  --filter="bindings.members:argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com"

# Check service account roles in target cluster project
gcloud projects get-iam-policy sap-ems-gap-sandbox-net \
  --flatten="bindings[].members" \
  --format="table(bindings.role)" \
  --filter="bindings.members:argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com"
```

### Test Workload Identity

```bash
# Test from ArgoCD pod
kubectl exec -it argocd-application-controller-0 -n argocd -- \
  curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email
```

### Test Connect Gateway Access

```bash
# Test Connect Gateway connectivity
kubectl exec -it argocd-application-controller-0 -n argocd -- \
  curl -s "https://connectgateway.googleapis.com/v1beta1/projects/90257023985/locations/us-east1/gkeMemberships/gap-staging-use1/api/v1/nodes" \
  | jq '.items[0].metadata.name'
```

## Common Issues and Solutions

### Issue: "Failed to get access token"
**Solution**: Check Workload Identity configuration and service account binding

### Issue: "403 Forbidden" from Connect Gateway
**Solution**: Verify `roles/gkehub.gatewayEditor` role is granted

### Issue: "Image pull failed"  
**Solution**: Grant `roles/artifactregistry.reader` to cluster node service account

### Issue: "Unknown error synchronizing cache"
**Solution**: Enable `connectgateway.googleapis.com` API on target cluster project

### Issue: "Cannot watch resource X"
**Solution**: Grant `roles/container.developer` instead of just `roles/container.clusterViewer`