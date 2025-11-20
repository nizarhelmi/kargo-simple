# ArgoCD Patches Reference

## ✅ **Current Status: Already Applied**

The live cluster already has all required patches applied and working.  
**Use this reference for**: New deployments, disaster recovery, or understanding the setup.

## What Makes Multi-Cluster GitOps Work

### 1. Connect Gateway Authentication
```yaml
env:
- name: GOOGLE_SERVICE_ACCOUNT_NAME
  value: argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com
```
**Why needed**: Required by argocd-k8s-auth plugin for Connect Gateway access.

### 2. Network Restrictions Workaround
```yaml
nodeSelector:
  role: whitelist
tolerations:
- effect: NoSchedule
  key: role
  value: whitelist
```
**Why needed**: Control cluster blocks internet access except on whitelist nodes.

### 3. Proxy Configuration
```yaml
proxy: http://100.64.40.2:3128
noProxy: ".svc,.cluster.svc,100.64.36.1,argocd-*"
```
**Why needed**: Git repository access through corporate proxy.

## Files Included

- `argocd/patches/argocd-controller-patch.yaml` - Controller configuration
- `argocd/patches/proxy-repository-credentials.yaml` - Proxy settings

## Quick Verification

```bash
# Check patches are applied
kubectl get statefulset argocd-application-controller -n argocd -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="GOOGLE_SERVICE_ACCOUNT_NAME")].value}'

# Should return: argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com
```

## For New Deployments

If deploying from scratch:
```bash
kubectl patch statefulset argocd-application-controller -n argocd --patch-file argocd/patches/argocd-controller-patch.yaml
kubectl apply -f argocd/patches/proxy-repository-credentials.yaml
```

---
*These patches bridge the gap between standard ArgoCD and enterprise network restrictions.*