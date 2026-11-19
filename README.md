# GitOps Shop Configuration

Kubernetes desired state for the GitOps capstone.

Replace `CHANGE_ME` with your GitHub username/organization in image paths and Argo CD repository URLs. If GHCR packages are private, create an imagePullSecret in the `shop` namespace.

Apply Argo CD Applications after installing Argo CD:
```bash
kubectl apply -f argocd/staging.yaml
kubectl apply -f argocd/production.yaml
kubectl apply -f argocd/monitoring-apps.yaml
kubectl apply -f argocd/logging-apps.yaml
kubectl apply -f argocd/tempo-app.yaml
```

Staging auto-syncs with prune and self-heal. Production has no automated sync; manually sync after promoting image tags in `apps/production/kustomization.yaml`.

Before deploying Grafana, create Secret `grafana-admin` in namespace `monitoring` with keys `admin-user` and `admin-password`. Configure an actual Alertmanager receiver and credentials before expecting notifications. Never commit credentials.
