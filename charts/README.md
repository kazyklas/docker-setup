# Helm charts

## ArgoCD 

### ArgoCDupdate

This is the way how we do updates and cloning the repository

```bash
git subtree add \
  --prefix=charts/argo-helm \
  https://github.com/argoproj/argo-helm.git \
  main \
  --squash
```

```bash
git subtree pull \
  --prefix=charts/argo-helm \
  https://github.com/argoproj/argo-helm.git \
  main \
  --squash
```

### Deployment

And this is how we deploy the ArgoCD

```bash
# we need redis-ha
helm repo add dandydeveloper https://dandydeveloper.github.io/charts/
helm repo update
helm dependency build 

# then we can install the argocd
cd charts/argo-helm/charts/argo-cd
helm upgrade --install argocd \
  . \
  -n argocd \
  --create-namespace \
  -f values-local.yaml
  
# We can get password afterwards with
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---
