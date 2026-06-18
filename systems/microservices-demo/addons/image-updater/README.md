```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/config/install.yaml

sleep 15  

kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater  

kubectl apply -f image-updater.yaml

sleep 20

kubectl get imageupdater -n argocd

kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater --tail=5
```
