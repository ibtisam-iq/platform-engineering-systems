# ArgoCD installation on EKS + Gateway API

```bash
# Install ArgoCD
kubectl create namespace argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update argo

helm install argocd argo/argo-cd \
  --namespace argocd \
  -f patch-values.yaml \
  --version 9.5.21

# After the Service exists, apply the TargetGroupConfiguration
kubectl apply -f target-grp-config.yaml
```


## Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Access ArgoCD UI

https://argocd.ibtisam.qzz.io
