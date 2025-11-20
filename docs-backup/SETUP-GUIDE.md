# Multi-Cluster Setup Guide

This guide provides step-by-step instructions to reproduce the complete multi-cluster GitOps setup with Kargo, ArgoCD, and GKE Fleet.

## Prerequisites

- Google Cloud SDK (`gcloud`) installed and authenticated
- `kubectl` installed and configured
- Access to three Google Cloud projects:
  - Fleet Host Project (for managing fleet)
  - Control Cluster Project (for hosting Kargo/ArgoCD)  
  - Target Cluster Project (for hosting target applications)
- GKE clusters already created in the respective projects

## Step 1: Enable Required APIs

### Fleet Host Project
```bash
export FLEET_PROJECT_ID="your-fleet-project-id"
export FLEET_PROJECT_NUMBER="your-fleet-project-number"

gcloud config set project $FLEET_PROJECT_ID
gcloud services enable container.googleapis.com \
  gkehub.googleapis.com \
  cloudresourcemanager.googleapis.com \
  iam.googleapis.com \
  connectgateway.googleapis.com \
  anthos.googleapis.com
```

### Target Cluster Project
```bash
export TARGET_PROJECT_ID="your-target-project-id"

gcloud services enable container.googleapis.com \
  gkehub.googleapis.com \
  connectgateway.googleapis.com \
  --project=$TARGET_PROJECT_ID
```

## Step 5: Create IAM Service Account

### Create Service Account
```bash
gcloud iam service-accounts create argocd-fleet-access \
  --project=$FLEET_PROJECT_ID \
  --description="ArgoCD service account for Fleet access" \
  --display-name="ArgoCD Fleet Access"
```

### Grant Required Roles
```bash
# Fleet Host Project Roles
gcloud projects add-iam-policy-binding $FLEET_PROJECT_ID \
  --member "serviceAccount:argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com" \
  --role "roles/container.clusterViewer"

gcloud projects add-iam-policy-binding $FLEET_PROJECT_ID \
  --member "serviceAccount:argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com" \
  --role "roles/gkehub.viewer"

gcloud projects add-iam-policy-binding $FLEET_PROJECT_ID \
  --member "serviceAccount:argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com" \
  --role "roles/gkehub.gatewayEditor"

# Target Cluster Project Roles  
gcloud projects add-iam-policy-binding $TARGET_PROJECT_ID \
  --member "serviceAccount:argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com" \
  --role "roles/container.clusterViewer"

gcloud projects add-iam-policy-binding $TARGET_PROJECT_ID \
  --member "serviceAccount:argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com" \
  --role "roles/container.developer"
```

### Grant Target Project Roles
```bash
gcloud projects add-iam-policy-binding $TARGET_PROJECT_ID \
  --member "serviceAccount:argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com" \
  --role "roles/container.developer"
```

## Step 3: Enable Workload Identity on Clusters

### Control Cluster (Fleet Host Project)
```bash
export CONTROL_CLUSTER_NAME="gap-staging-use4"
export CONTROL_CLUSTER_LOCATION="us-east4" 
export CONTROL_PROJECT_ID="sap-ems-gap-sandbox"

gcloud container clusters update $CONTROL_CLUSTER_NAME \
  --workload-pool=$CONTROL_PROJECT_ID.svc.id.goog \
  --region=$CONTROL_CLUSTER_LOCATION \
  --project=$CONTROL_PROJECT_ID
```

### Target Cluster 
```bash
export TARGET_CLUSTER_NAME="gap-staging-use1"
export TARGET_CLUSTER_LOCATION="us-east1"

gcloud container clusters update $TARGET_CLUSTER_NAME \
  --workload-pool=$TARGET_PROJECT_ID.svc.id.goog \
  --region=$TARGET_CLUSTER_LOCATION \
  --project=$TARGET_PROJECT_ID
```

## Step 4: Register Target Cluster with Fleet

### Register Cluster
```bash
export TARGET_CLUSTER_NAME="your-target-cluster-name"
export TARGET_CLUSTER_LOCATION="your-target-cluster-location"

gcloud container fleet memberships register $TARGET_CLUSTER_NAME \
  --gke-cluster=projects/$TARGET_PROJECT_ID/locations/$TARGET_CLUSTER_LOCATION/clusters/$TARGET_CLUSTER_NAME \
  --enable-workload-identity \
  --project=$FLEET_PROJECT_ID
```

### Enable Connect Agent
```bash
gcloud container fleet memberships update $TARGET_CLUSTER_NAME \
  --location=$TARGET_CLUSTER_LOCATION \
  --project=$FLEET_PROJECT_ID \
  --update-labels=connect-agent=enabled
```

## Step 4: Configure Control Cluster

### Switch to Control Cluster
```bash
export CONTROL_CLUSTER_NAME="your-control-cluster-name"
export CONTROL_CLUSTER_LOCATION="your-control-cluster-location"
export CONTROL_PROJECT_ID="your-control-project-id"

gcloud container clusters get-credentials $CONTROL_CLUSTER_NAME \
  --region=$CONTROL_CLUSTER_LOCATION \
  --project=$CONTROL_PROJECT_ID
```

### Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Configure Workload Identity for ArgoCD
```bash
# Annotate ArgoCD service account
kubectl annotate serviceaccount argocd-application-controller \
  --namespace argocd \
  iam.gke.io/gcp-service-account=argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com

# Grant Workload Identity binding
gcloud iam service-accounts add-iam-policy-binding \
  argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:$FLEET_PROJECT_ID.svc.id.goog[argocd/argocd-application-controller]"
```

### Apply ArgoCD Environment Patch
```bash
# Create environment patch
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: argocd-application-controller
  namespace: argocd
spec:
  template:
    spec:
      containers:
      - name: application-controller
        env:
        - name: GOOGLE_SERVICE_ACCOUNT_NAME
          value: "argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com"
EOF

# Patch the StatefulSet
kubectl patch statefulset argocd-application-controller -n argocd --patch-file /dev/stdin <<EOF
spec:
  template:
    spec:
      containers:
      - name: application-controller
        env:
        - name: GOOGLE_SERVICE_ACCOUNT_NAME
          value: "argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com"
EOF
```

## Step 6: Configure Workload Identity

### Switch Back to Control Cluster  
```bash
gcloud container clusters get-credentials $CONTROL_CLUSTER_NAME \
  --region=$CONTROL_CLUSTER_LOCATION \
  --project=$CONTROL_PROJECT_ID
```

### Bind Google Service Account to Kubernetes Service Account
```bash
gcloud iam service-accounts add-iam-policy-binding \
  argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com \
  --project=$FLEET_PROJECT_ID \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:$CONTROL_PROJECT_ID.svc.id.goog[argocd/argocd-application-controller]"

# Annotate Kubernetes service account  
kubectl annotate serviceaccount argocd-application-controller \
  -n argocd \
  iam.gke.io/gcp-service-account=argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com
```

### Configure Environment Variable for Connect Gateway
```bash
# Set environment variable for Workload Identity
kubectl patch statefulset argocd-application-controller -n argocd \
  --type='merge' \
  --patch='{"spec":{"template":{"spec":{"containers":[{"name":"application-controller","env":[{"name":"GOOGLE_SERVICE_ACCOUNT_NAME","value":"argocd-fleet-access@'$FLEET_PROJECT_ID'.iam.gserviceaccount.com"}]}]}}}}'
```

## Step 7: Configure ArgoCD for Network-Restricted Environment

**Important**: If your control cluster has network restrictions (private cluster with limited internet access), ArgoCD components must be scheduled on nodes with internet connectivity.

### Check if Whitelist Nodepool Exists
```bash
kubectl get nodes --show-labels | grep whitelist
```

### Configure ArgoCD Application Controller for Whitelist Nodes
```bash
kubectl patch statefulset argocd-application-controller -n argocd \
  --type='merge' \
  --patch='{"spec":{"template":{"spec":{"nodeSelector":{"role":"whitelist"},"tolerations":[{"effect":"NoSchedule","key":"role","operator":"Equal","value":"whitelist"}]}}}}'
```

### Configure ArgoCD Server for Whitelist Nodes  
```bash
kubectl patch deployment argocd-server -n argocd \
  --type='merge' \
  --patch='{"spec":{"template":{"spec":{"nodeSelector":{"role":"whitelist"},"tolerations":[{"effect":"NoSchedule","key":"role","operator":"Equal","value":"whitelist"}]}}}}'
```

### Verify Pods are Scheduled Correctly
```bash
kubectl get pods -n argocd -o wide | grep -E "(argocd-application-controller|argocd-server)"
```

All ArgoCD pods should be running on nodes with names containing "whitelist".

## Step 9: Configure Container Image Access

### Grant Registry Access to Target Cluster
```bash
# Get target cluster project number
export TARGET_PROJECT_NUMBER=$(gcloud projects describe $TARGET_PROJECT_ID --format="value(projectNumber)")

# Grant Artifact Registry read access to cluster nodes
export REGISTRY_PROJECT_ID="your-registry-project-id"
gcloud projects add-iam-policy-binding $REGISTRY_PROJECT_ID \
  --member "serviceAccount:$TARGET_PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role "roles/artifactregistry.reader"
```

## Step 10: Install and Configure Kargo

### Switch Back to Control Cluster
```bash
gcloud container clusters get-credentials $CONTROL_CLUSTER_NAME \
  --region=$CONTROL_CLUSTER_LOCATION \
  --project=$CONTROL_PROJECT_ID
```

### Install Kargo
```bash
# Install Kargo
kubectl apply -f https://raw.githubusercontent.com/akuity/kargo/release-1.3/install.yaml

# Wait for Kargo to be ready
kubectl wait --for=condition=Available --timeout=300s deployment/kargo-controller-manager -n kargo
```

### Apply Kargo Configuration
```bash
kubectl apply -f kargo/
```

## Step 11: Configure ArgoCD Applications

### Create Connect Gateway Applications
```bash
# Calculate Connect Gateway URL
export CONNECT_GATEWAY_URL="https://connectgateway.googleapis.com/v1beta1/projects/$FLEET_PROJECT_NUMBER/locations/$TARGET_CLUSTER_LOCATION/gkeMemberships/$TARGET_CLUSTER_NAME"

# Apply ArgoCD applications
kubectl apply -f argocd/applications/
```

## Step 12: Validation

### Test Connect Gateway Connectivity
```bash
kubectl exec -it argocd-application-controller-0 -n argocd -- \
  curl -s "$CONNECT_GATEWAY_URL/api/v1/nodes" | jq '.items[0].metadata.name'
```

### Check ArgoCD Application Status
```bash
kubectl get applications -n argocd
kubectl get application guestbook-dev-use1 -n argocd -o jsonpath='{.status.health.status}'
```

### Verify Kargo Stages
```bash
kubectl get stages -n gke-fleet
kubectl get freight -n gke-fleet
```

## Step 13: Test Promotion Workflow

### Create Test Promotion
```bash
# Get latest freight
export LATEST_FREIGHT=$(kubectl get freight -n gke-fleet -o jsonpath='{.items[0].metadata.name}')

# Create promotion
cat <<EOF | kubectl apply -f -
apiVersion: kargo.akuity.io/v1alpha1
kind: Promotion
metadata:
  name: test-promotion-$(date +%s)
  namespace: gke-fleet
spec:
  stage: staging
  freight: $LATEST_FREIGHT
EOF
```

### Monitor Promotion
```bash
kubectl get promotions -n gke-fleet -w
kubectl logs -f deployment/kargo-controller-manager -n kargo -c manager
```

## Troubleshooting

### Common Issues and Solutions

#### 1. "Error synchronizing cache state: unknown"
```bash
# Check if Connect Gateway API is enabled on target project
gcloud services list --enabled --project=$TARGET_PROJECT_ID --filter="name:connectgateway"

# Enable if missing
gcloud services enable connectgateway.googleapis.com --project=$TARGET_PROJECT_ID
```

#### 2. "Failed to get access token"  
```bash
# Check Workload Identity configuration
kubectl describe serviceaccount argocd-application-controller -n argocd

# Verify annotation is present
kubectl get serviceaccount argocd-application-controller -n argocd -o yaml | grep iam.gke.io
```

#### 3. "403 Forbidden" errors
```bash
# Verify service account has required roles
gcloud projects get-iam-policy $FLEET_PROJECT_ID \
  --flatten="bindings[].members" \
  --format="table(bindings.role)" \
  --filter="bindings.members:argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com"
```

#### 4. Image pull failures
```bash
# Check if registry access is granted
gcloud projects get-iam-policy $REGISTRY_PROJECT_ID \
  --flatten="bindings[].members" \
  --format="table(bindings.role)" \
  --filter="bindings.members:$TARGET_PROJECT_NUMBER-compute@developer.gserviceaccount.com"
```

### Debug Commands

```bash
# Check ArgoCD logs
kubectl logs argocd-application-controller-0 -n argocd --tail=50

# Check Kargo logs
kubectl logs deployment/kargo-controller-manager -n kargo -c manager --tail=50

# Check Fleet membership
gcloud container fleet memberships describe $TARGET_CLUSTER_NAME \
  --location=$TARGET_CLUSTER_LOCATION \
  --project=$FLEET_PROJECT_ID

# Test Connect Gateway directly
kubectl exec -it argocd-application-controller-0 -n argocd -- \
  curl -v "$CONNECT_GATEWAY_URL/api/v1/nodes"
```

## Security Considerations

1. **Least Privilege**: Grant only necessary permissions to service accounts
2. **Network Security**: Use private clusters and firewall rules where possible
3. **Workload Identity**: Always use Workload Identity instead of service account keys
4. **RBAC**: Apply principle of least privilege for Kubernetes RBAC
5. **Audit**: Enable audit logging on all clusters

## Cleanup (Optional)

To remove the setup:

```bash
# Delete ArgoCD applications
kubectl delete -f argocd/applications/

# Delete Kargo resources
kubectl delete -f kargo/

# Unregister cluster from fleet
gcloud container fleet memberships unregister $TARGET_CLUSTER_NAME \
  --project=$FLEET_PROJECT_ID

# Remove IAM bindings
gcloud projects remove-iam-policy-binding $FLEET_PROJECT_ID \
  --member "serviceAccount:argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com" \
  --role "roles/gkehub.viewer"

# Delete service account
gcloud iam service-accounts delete argocd-fleet-access@$FLEET_PROJECT_ID.iam.gserviceaccount.com \
  --project=$FLEET_PROJECT_ID
```