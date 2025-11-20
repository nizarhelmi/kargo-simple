# Network Configuration & Whitelist Nodepool Setup

This document explains the critical network configuration required for ArgoCD to work in a restricted GKE environment.

## Problem Statement

The control cluster (`gap-staging-use4`) is a **private GKE cluster with network restrictions** that prevent most nodes from accessing the internet. However, ArgoCD requires internet access for:

1. **Connect Gateway API calls** - Cross-project cluster communication
2. **Git repository access** - Fetching application manifests  
3. **Container registry access** - Pulling container images
4. **Google Cloud APIs** - Workload Identity token exchange

## Solution: Whitelist Nodepool

### Nodepool Configuration

**Whitelist nodepool** with special network configuration allowing internet access:

```yaml
Nodepool Name: whitelist
Node Count: 3 (across zones)
Machine Type: e2-standard-8
Network Access: ✅ Internet enabled
Node Labels:
  - role=whitelist
  - cloud.google.com/gke-nodepool=whitelist
Node Taints:
  - role=whitelist:NoSchedule
```

### Why Taints and Tolerations?

The `role=whitelist:NoSchedule` taint ensures:
- **Security**: Prevents regular workloads from accidentally scheduling on internet-enabled nodes
- **Resource isolation**: Reserves whitelist nodes for components that actually need internet access
- **Cost control**: Limits internet-enabled resources to necessary components only

## ArgoCD Scheduling Configuration

### ArgoCD Application Controller

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: argocd-application-controller
  namespace: argocd
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: "linux"
        role: "whitelist"                    # Schedule only on whitelist nodes
      tolerations:
      - effect: "NoSchedule"
        key: "role"
        operator: "Equal"
        value: "whitelist"                   # Tolerate the whitelist taint
      containers:
      - name: application-controller
        env:
        - name: GOOGLE_SERVICE_ACCOUNT_NAME  # Required for Workload Identity
          value: "argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com"
```

### ArgoCD Server

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-server
  namespace: argocd
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: "linux"
        role: "whitelist"                    # Schedule only on whitelist nodes
      tolerations:
      - effect: "NoSchedule"
        key: "role" 
        operator: "Equal"
        value: "whitelist"                   # Tolerate the whitelist taint
```

## Verification Commands

### Check Nodepool Configuration
```bash
# List whitelist nodes
kubectl get nodes -l role=whitelist

# Check node labels and taints
kubectl describe node <whitelist-node-name>
```

### Verify Pod Scheduling
```bash
# Check ArgoCD pods are on whitelist nodes
kubectl get pods -n argocd -o wide | grep -E "(argocd-application-controller|argocd-server)"

# Expected output: All pods should be on nodes with "whitelist" in the name
```

### Test Internet Connectivity
```bash
# Test from ArgoCD application controller pod
kubectl exec -it argocd-application-controller-0 -n argocd -- curl -s https://www.google.com

# Test Connect Gateway access
kubectl exec -it argocd-application-controller-0 -n argocd -- curl -s "https://connectgateway.googleapis.com/v1beta1/projects/90257023985/locations/us-east1/gkeMemberships/gap-staging-use1/api/v1/nodes"
```

## Troubleshooting Network Issues

### Common Symptoms
1. **"error synchronizing cache state : unknown"** - Usually network connectivity issues
2. **Pod stuck in Pending state** - Missing tolerations for whitelist taint
3. **Connect Gateway timeouts** - Pod scheduled on restricted node without internet access
4. **Git sync failures** - No internet access to GitHub

### Diagnosis Steps

1. **Check pod scheduling**:
   ```bash
   kubectl get pods -n argocd -o wide
   kubectl describe pod <argocd-pod-name> -n argocd
   ```

2. **Test network connectivity**:
   ```bash
   kubectl exec -it <argocd-pod-name> -n argocd -- nslookup google.com
   kubectl exec -it <argocd-pod-name> -n argocd -- curl -v https://connectgateway.googleapis.com
   ```

3. **Check node configuration**:
   ```bash
   kubectl get nodes -l role=whitelist
   kubectl describe node <node-name>
   ```

### Fix Commands

**Reschedule ArgoCD application controller to whitelist nodes**:
```bash
kubectl patch statefulset argocd-application-controller -n argocd \
  --type='merge' \
  --patch='{"spec":{"template":{"spec":{"nodeSelector":{"role":"whitelist"},"tolerations":[{"effect":"NoSchedule","key":"role","operator":"Equal","value":"whitelist"}]}}}}'
```

**Reschedule ArgoCD server to whitelist nodes**:
```bash
kubectl patch deployment argocd-server -n argocd \
  --type='merge' \
  --patch='{"spec":{"template":{"spec":{"nodeSelector":{"role":"whitelist"},"tolerations":[{"effect":"NoSchedule","key":"role","operator":"Equal","value":"whitelist"}]}}}}'
```

## Current Status

✅ **Working Configuration**:
- All ArgoCD pods running on whitelist nodepool
- Internet connectivity verified
- Connect Gateway communication working
- Git repository access working
- Container image pulling working

## Security Considerations

1. **Principle of Least Privilege**: Only ArgoCD components scheduled on internet-enabled nodes
2. **Taint Protection**: Prevents accidental scheduling of non-essential workloads
3. **Network Segmentation**: Whitelist nodes are the only internet-enabled nodes in the cluster
4. **Monitoring**: Should monitor internet usage and access patterns on whitelist nodes

This network configuration is essential for the multi-cluster GitOps setup to function in enterprise environments with network security restrictions.