# Architecture

```text
gitops-shop-app -> GitHub Actions (lint/test/audit)
                -> build each service image
                -> Trivy scan
                -> push scanned images to GHCR
                -> update staging tags in gitops-shop-config

gitops-shop-config -> Argo CD
                   -> staging (auto-sync/self-heal)
                   -> production (manual sync)

Kubernetes workloads -> Prometheus metrics
                     -> Alloy logs -> Loki
                     -> trace export -> Tempo (after SDK instrumentation)
Prometheus alerts -> Alertmanager -> configured notification receiver
Grafana visualizes metrics, logs and traces.
```
