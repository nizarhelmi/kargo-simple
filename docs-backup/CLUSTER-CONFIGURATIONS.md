# Cluster Configurations

This document details the specific configurations required for each cluster and project in the multi-cluster setup.

## Overview

Our setup consists of:
- **Fleet Host Project**: `sap-ems-central-monitoring-poc` (90257023985)
- **Control Cluster Project**: `sap-ems-gap-sandbox` (525135499313) 
- **Target Cluster Project**: `sap-ems-gap-sandbox-net` (90608020739)

## Fleet Host Project Configuration

**Project**: `sap-ems-central-monitoring-poc`  
**Purpose**: Hosts the GKE Fleet and manages Connect Gateway access

### Required Configurations:

#### 1. APIs Enabled
```bash
gcloud services enable container.googleapis.com \
  gkehub.googleapis.com \
  cloudresourcemanager.googleapis.com \
  iam.googleapis.com \
  connectgateway.googleapis.com \
  --project=sap-ems-central-monitoring-poc
```

#### 2. IAM Service Account
```yaml
Service Account: argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com
Fleet Host Project Roles:
  - roles/container.clusterViewer  # Basic cluster viewing
  - roles/gkehub.gatewayEditor     # Connect Gateway access
  - roles/gkehub.viewer           # Fleet visibility
Target Cluster Project Roles:
  - roles/container.clusterViewer  # Basic cluster viewing
  - roles/container.developer      # Cluster management capabilities
```

#### 3. Workload Identity Bindings
```bash
# Current Workload Identity bindings (control cluster project)
gcloud iam service-accounts add-iam-policy-binding \
  argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:sap-ems-gap-sandbox.svc.id.goog[argocd/argocd-application-controller]" \
  --project=sap-ems-central-monitoring-poc

gcloud iam service-accounts add-iam-policy-binding \
  argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:sap-ems-gap-sandbox.svc.id.goog[argocd/argocd-server]" \
  --project=sap-ems-central-monitoring-poc
```

#### 4. Fleet Memberships
```bash
# Target cluster registered as fleet member
Fleet Member: gap-staging-use1
Location: us-east1
Project: sap-ems-gap-sandbox-net
Connect Agent: Enabled
```

## Control Cluster Configuration  

**Cluster**: `gap-staging-use4`  
**Project**: `sap-ems-gap-sandbox`
**Purpose**: Hosts Kargo and ArgoCD controllers

### Required Configurations:

#### 1. Cluster Features
```bash
# Workload Identity enabled (REQUIRED for Connect Gateway)
Workload Pool: sap-ems-gap-sandbox.svc.id.goog
Status: ✅ Enabled

# Enable with:
gcloud container clusters update gap-staging-use4 \
  --workload-pool=sap-ems-gap-sandbox.svc.id.goog \
  --region=us-east4
```

#### 2. Network Configuration & Whitelist Nodepool
**Issue**: This cluster has network restrictions that block internet access from default nodes.

**ArgoCD Requirements**:
- Connect Gateway API communication  
- Git repository access
- Container image pulling from Artifact Registry

**Solution**: ArgoCD pods scheduled on `whitelist` nodepool with internet access.

```yaml
# Whitelist Nodepool Configuration
Nodepool Name: whitelist
Node Labels:
  - role=whitelist
  - cloud.google.com/gke-nodepool=whitelist
Node Taints:
  - role=whitelist:NoSchedule
Internet Access: ✅ Enabled
```

#### 3. ArgoCD Scheduling Configuration

**Critical**: ArgoCD components must run on whitelist nodes for internet connectivity.

**ArgoCD Application Controller**:
```yaml
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: "linux"
        role: "whitelist"
      tolerations:
      - effect: "NoSchedule"
        key: "role"
        operator: "Equal"
        value: "whitelist"
      containers:
      - name: application-controller
        env:
        - name: GOOGLE_SERVICE_ACCOUNT_NAME
          value: "argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com"
```

**ArgoCD Server**:
```yaml
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: "linux"
        role: "whitelist"
      tolerations:
      - effect: "NoSchedule"
        key: "role"
        operator: "Equal"
        value: "whitelist"
```

**Service Account Annotation**:
```bash
kubectl annotate serviceaccount argocd-application-controller \
  --namespace argocd \
  iam.gke.io/gcp-service-account=argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com
```

#### 3. Network Configuration

**Node Constraints** (if network restricted):
```yaml
# For ArgoCD pods requiring internet access
nodeSelector:
  role: whitelist
tolerations:
- key: "role"
  operator: "Equal"
  value: "whitelist"
  effect: "NoSchedule"
```

#### 4. Kargo Configuration

**Namespace**: `kargo-simple`

**Image Repository**: `us-east4-docker.pkg.dev/sap-ems-gap-sandbox/use4-misc-images/guestbook`

## Target Cluster Configuration

**Cluster**: `gap-staging-use1`  
**Project**: `sap-ems-gap-sandbox-net`
**Purpose**: Hosts target applications deployed via ArgoCD

### Required Configurations:

#### 1. Workload Identity Configuration
```bash
# Workload Identity must be enabled for Fleet registration
Workload Pool: sap-ems-gap-sandbox-net.svc.id.goog
Status: ✅ Enabled

# Enable with:
gcloud container clusters update gap-staging-use1 \
  --workload-pool=sap-ems-gap-sandbox-net.svc.id.goog \
  --region=us-east1 \
  --project=sap-ems-gap-sandbox-net
```

#### 2. APIs Enabled
```bash
gcloud services enable container.googleapis.com \
  gkehub.googleapis.com \
  connectgateway.googleapis.com \
  --project=sap-ems-gap-sandbox-net
```

#### 3. Fleet Registration
```bash
# Register cluster with Fleet in fleet host project
gcloud container fleet memberships register gap-staging-use1 \
  --gke-cluster=projects/sap-ems-gap-sandbox-net/locations/us-east1/clusters/gap-staging-use1 \
  --enable-workload-identity \
  --project=sap-ems-central-monitoring-poc
```

#### 3. Connect Agent
```bash
# Enable Connect agent for Connect Gateway access
gcloud gke hub memberships update gap-staging-use1 \
  --location=us-east1 \
  --connect-agent \
  --project=sap-ems-central-monitoring-poc
```

#### 4. RBAC Configuration
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

#### 5. Container Registry Access
```bash
# Grant target cluster node service account access to pull images from registry
gcloud projects add-iam-policy-binding sap-ems-gap-sandbox \
  --member "serviceAccount:90608020739-compute@developer.gserviceaccount.com" \
  --role "roles/artifactregistry.reader" \
  --project=sap-ems-gap-sandbox
```

#### 6. Cross-Project IAM
```bash
# Grant ArgoCD service account access to target cluster project  
gcloud projects add-iam-policy-binding sap-ems-gap-sandbox-net \
  --member "serviceAccount:argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com" \
  --role "roles/container.clusterViewer" \
  --project=sap-ems-gap-sandbox-net

gcloud projects add-iam-policy-binding sap-ems-gap-sandbox-net \
  --member "serviceAccount:argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com" \
  --role "roles/container.developer" \
  --project=sap-ems-gap-sandbox-net
```

## Connect Gateway URLs

### Format
```
https://connectgateway.googleapis.com/v1beta1/projects/{FLEET_HOST_PROJECT_NUMBER}/locations/{FLEET_LOCATION}/gkeMemberships/{CLUSTER_NAME}
```

### Actual URLs Used
```yaml
gap-staging-use1: https://connectgateway.googleapis.com/v1beta1/projects/90257023985/locations/us-east1/gkeMemberships/gap-staging-use1
```

## Application Namespaces

Target cluster applications are deployed to these namespaces:
- `guestbook-simple-dev` - Development environment
- `guestbook-simple-staging` - Staging environment  
- `guestbook-simple-prod` - Production environment

## Verification Commands

### Check Fleet Membership
```bash
gcloud container fleet memberships describe gap-staging-use1 \
  --location=us-east1 \
  --project=sap-ems-central-monitoring-poc
```

### Test Connect Gateway Access
```bash
kubectl exec -it argocd-application-controller-0 -n argocd -- \
  curl -s "https://connectgateway.googleapis.com/v1beta1/projects/90257023985/locations/us-east1/gkeMemberships/gap-staging-use1/api/v1/nodes"
```

### Check ArgoCD Application Status
```bash
kubectl get application -n argocd
kubectl get application guestbook-dev-use1 -n argocd -o jsonpath='{.status.health.status}'
```