# GitOps Shop Configuration

Kubernetes desired state for the GitOps Shop capstone. Argo CD watches this repository and keeps the cluster in sync with it.

Application code and CI/CD live in [capstone2-app](https://github.com/charlesdev96/capstone2-app).

## How it works

```text
push to capstone2-app
  -> CI: lint, test, audit, build, Trivy scan, push images to GHCR
  -> CI bot commits the new image tag to apps/staging and apps/production (this repo)
  -> Argo CD auto-syncs staging                       (namespace: shop-staging)
  -> production shows OutOfSync -> you click Sync     (namespace: shop-production)
```

| Environment | Namespace | Sync |
|---|---|---|
| Staging | `shop-staging` | Automatic (prune + self-heal) |
| Production | `shop-production` | Manual |
| Monitoring | `monitoring` | Automatic |

## Repository layout

```text
apps/
  base/                  Deployments + Services for gateway, users, orders, payments
  staging/               Overlay: namespace shop-staging, image tags (updated by CI)
  production/            Overlay: namespace shop-production, 2 gateway replicas
argocd/                  Argo CD namespace + Application manifests
monitoring/              Helm values, ServiceMonitor, alert rules, namespace, secret template
runbooks/                Response steps for each alert
docs/                    Architecture and design decisions
```

## Tools

### Install on your machine

| Tool | Purpose | Required |
|---|---|---|
| Docker Desktop (Kubernetes enabled) | Local Kubernetes cluster | Yes |
| kubectl | Apply manifests, inspect the cluster | Yes |
| git | Clone and push this repository | Yes |
| Helm CLI | Inspect charts locally (`helm template`); not needed to deploy | Optional |
| argocd CLI | Manage Argo CD from the terminal; the web UI is enough | Optional |

### Installed into the cluster by these steps

You do **not** install these by hand. Step 1 installs Argo CD; Argo CD then installs everything else from Helm charts, using the values files in `monitoring/`.

| Component | Purpose | Helm chart | Values file | Argo CD app file |
|---|---|---|---|---|
| Argo CD | GitOps sync engine | — (official install manifest) | — | — |
| Prometheus + Alertmanager + Grafana | Metrics, alerting, dashboards | `kube-prometheus-stack` 68.3.0 | `prometheus-values.yaml` | `monitoring-apps.yaml` |
| Loki | Log storage | `loki` 6.29.0 | `loki-values.yaml` | `logging-apps.yaml` |
| Alloy | Collects pod logs, ships to Loki | `alloy` 0.12.6 | `alloy-values.yaml` | `logging-apps.yaml` |
| Tempo | Trace storage | `tempo` 1.24.4 | `tempo-values.yaml` | `tempo-app.yaml` |

To change a component, edit its values file and push. Argo CD upgrades the Helm release automatically.

## Prerequisites

- A Kubernetes cluster (these steps use Docker Desktop's built-in Kubernetes) and `kubectl` pointing at it.
- The four GHCR packages (`gitops-shop-gateway`, `-users`, `-orders`, `-payments`) set to **Public**.
- The `CONFIG_REPO_TOKEN` secret set in the app repository.

Check that kubectl talks to the right cluster:

```powershell
kubectl config current-context    # should print docker-desktop
kubectl get nodes                 # STATUS should be Ready
```

All commands below are run from the root of this repository.

## Step 1 — Install Argo CD

```powershell
kubectl apply -f argocd/namespace.yaml
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until every pod is `Running`:

```powershell
kubectl get pods -n argocd -w
```

## Step 2 — Open the Argo CD UI

In a separate terminal (leave it running):

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the initial `admin` password:

```powershell
$p = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p))
```

Open https://localhost:8080 (accept the certificate warning) and log in as `admin`.

## Step 3 — Create the monitoring namespace and Grafana secret

```powershell
kubectl apply -f monitoring/namespace.yaml
Copy-Item monitoring/grafana-admin-secret.example.yaml monitoring/grafana-admin-secret.yaml
```

Edit `monitoring/grafana-admin-secret.yaml` and set a real password, then:

```powershell
kubectl apply -f monitoring/grafana-admin-secret.yaml
```

`grafana-admin-secret.yaml` is git-ignored. Never commit it.

## Step 4 — (Optional) Set the alert webhook

To receive critical alert notifications, replace `http://REPLACE_WITH_REAL_NOTIFICATION_WEBHOOK` in `monitoring/prometheus-values.yaml` with a real URL (for testing, a free URL from https://webhook.site works). Commit and push. Everything deploys without this; only notifications are affected.

## Step 5 — Deploy the monitoring stack

```powershell
kubectl apply -f argocd/monitoring-apps.yaml     # Prometheus, Alertmanager, Grafana
kubectl apply -f argocd/logging-apps.yaml        # Loki + Alloy
kubectl apply -f argocd/tempo-app.yaml           # Tempo
kubectl apply -f argocd/monitoring-config-app.yaml   # ServiceMonitor + alert rules
```

Wait for the pods (this can take a few minutes):

```powershell
kubectl get pods -n monitoring -w
```

In the Argo CD UI all four monitoring apps should become **Synced** and **Healthy**.

## Step 6 — Deploy staging and production

```powershell
kubectl apply -f argocd/staging.yaml
kubectl apply -f argocd/production.yaml
```

Staging syncs on its own. Production stays **OutOfSync** until you sync it:

- Argo CD UI: open `gitops-shop-production` → **Sync** → **Synchronize**.

Check the pods:

```powershell
kubectl get pods -n shop-staging
kubectl get pods -n shop-production
```

## Step 7 — Test the application

```powershell
kubectl port-forward svc/gateway -n shop-staging 3001:3001
```

In another terminal:

```powershell
Invoke-RestMethod http://localhost:3001/health
Invoke-RestMethod http://localhost:3001/api/users
$body = @{ user_id = "u-1001"; item = "keyboard"; amount = 45 } | ConvertTo-Json
Invoke-RestMethod -Uri http://localhost:3001/api/orders -Method Post -ContentType "application/json" -Headers @{ "x-trace-id" = "demo-123" } -Body $body
```

Use `-n shop-production` (and a different local port such as `3101:3001`) to test production.

## Step 8 — Open Grafana

```powershell
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```

Open http://localhost:3000 and log in with the credentials from Step 3.

- **Metrics:** Explore → Prometheus → `sum by (service) (rate(http_requests_total[5m]))`
- **Logs:** Explore → Loki → `{namespace="shop-staging"} |= "demo-123"`
- **Alerts:** Alerting → Alert rules, or Prometheus targets at `kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090` → http://localhost:9090/targets (the `gitops-shop-services` targets should be UP)

## Step 9 — Verify the end-to-end GitOps flow

1. Push any small change to `capstone2-app`.
2. Watch the pipeline in the app repo's **Actions** tab.
3. A `chore(staging): deploy <sha>` commit appears in this repo.
4. Argo CD syncs `gitops-shop-staging` automatically (within ~3 minutes; click **Refresh** to speed it up).
5. Confirm the new image tag:

```powershell
kubectl get deploy -n shop-staging -o jsonpath="{range .items[*]}{.metadata.name}{'  '}{.spec.template.spec.containers[0].image}{'\n'}{end}"
```

## Deploying to production

No manual tag editing is needed. Every push to `capstone2-app` updates the image tag in both overlays:

1. Staging deploys the new version automatically.
2. `gitops-shop-production` shows **OutOfSync** in Argo CD (click **Refresh** if it has not appeared yet).
3. Check staging, then open `gitops-shop-production` → **Sync** → **Synchronize**.

To roll back, revert the bot's `chore: deploy <sha>` commit in this repository and sync production again.

> The CI bot commits to this repository. If you edit it locally, run `git pull` first so your push is not rejected.

## Alerts and runbooks

| Alert | Runbook |
|---|---|
| HighErrorRate | [runbooks/high-error-rate.md](runbooks/high-error-rate.md) |
| HighP99Latency | [runbooks/high-latency.md](runbooks/high-latency.md) |
| ShopPodCrashLooping | [runbooks/pod-crash-loop.md](runbooks/pod-crash-loop.md) |
| KubernetesNodeNotReady | [runbooks/node-not-ready.md](runbooks/node-not-ready.md) |
| KubernetesNodeDiskSpaceLow | [runbooks/disk-space-low.md](runbooks/disk-space-low.md) |

## Troubleshooting

| Symptom | Fix |
|---|---|
| Pods in `ImagePullBackOff` | Make sure the GHCR packages are Public and the tag exists. |
| Prometheus app fails with "annotations: Too long" | `ServerSideApply=true` must be in `argocd/monitoring-apps.yaml`. |
| `monitoring-config` app fails on ServiceMonitor/PrometheusRule kinds | Wait for the Prometheus app to install its CRDs, then sync again. |
| Grafana pod not starting | The `grafana-admin` secret is missing in `monitoring` (Step 3). |
| Loki pod stuck `Pending` | Check `kubectl describe pod -n monitoring <pod>`; reduce resources or persistence size. |
| node-exporter `CrashLoopBackOff` on Docker Desktop | Known Docker Desktop mount issue; add `prometheus-node-exporter: { hostRootFsMount: { enabled: false } }` to `monitoring/prometheus-values.yaml`. |
| Argo CD app stuck OutOfSync | Click **Refresh**, then check the app's events for the error. |

## Notes

- Requests carry an `x-trace-id` header that is logged as `trace_id`, so logs can be correlated across services. Tempo is installed as a trace backend, but the services do not yet export OpenTelemetry traces.
- Never commit credentials. Keep them in GitHub Actions Secrets or Kubernetes Secrets.
