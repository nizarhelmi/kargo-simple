# ArgoCD Configuration Patch

The following environment variable configuration was applied to the ArgoCD application controller to enable Workload Identity with Connect Gateway:

## Environment Variable Patch

```yaml
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
          value: "argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com"
```

## How It Was Applied

This configuration can be applied using a patch:

```bash
kubectl patch statefulset argocd-application-controller -n argocd \
  --type='merge' \
  --patch='{"spec":{"template":{"spec":{"containers":[{"name":"application-controller","env":[{"name":"GOOGLE_SERVICE_ACCOUNT_NAME","value":"argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com"}]}]}}}}'
```

Or applied via ArgoCD Helm values or Kustomize patch during initial deployment.

## Service Account Annotation

The ArgoCD application controller service account must be annotated for Workload Identity:

```bash
kubectl annotate serviceaccount argocd-application-controller \
  -n argocd \
  iam.gke.io/gcp-service-account=argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com
```

## Verification

Verify the configuration is working:

```bash
# Check environment variable is set
kubectl get statefulset argocd-application-controller -n argocd \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="GOOGLE_SERVICE_ACCOUNT_NAME")].value}'

# Check service account annotation
kubectl get serviceaccount argocd-application-controller -n argocd \
  -o jsonpath='{.metadata.annotations.iam\.gke\.io/gcp-service-account}'
```

Both commands should return: `argocd-fleet-access@sap-ems-central-monitoring-poc.iam.gserviceaccount.com`