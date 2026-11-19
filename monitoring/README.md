# Monitoring setup notes

Argo CD deploys kube-prometheus-stack, Loki, Alloy and Tempo from Helm charts. Apply `shop-servicemonitor.yaml` and `alerts.yaml` after the Prometheus Operator CRDs are installed, or add them to a dedicated Argo CD Application.

Create the `grafana-admin` Secret in namespace `monitoring` with keys `admin-user` and `admin-password` before deploying the chart.

The five alert rules cover high error rate, p99 latency, pod restarts, node readiness and low node disk space. Configure a real Alertmanager receiver and secret before expecting notifications. Validate the metric names/labels and disk mountpoint against the actual cluster.
