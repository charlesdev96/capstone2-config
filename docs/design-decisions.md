# Design Decisions

- Two repositories separate application delivery from Kubernetes desired state.
- Node.js and Express provide lightweight HTTP microservices.
- Gateway, Users, Orders and Payments provide multiple service boundaries and downstream calls.
- The multi-stage Dockerfile separates builder and production runner stages.
- Kustomize base/overlays reuse Kubernetes manifests across staging and production.
- Argo CD auto-syncs staging with self-heal; production requires manual sync.
- Prometheus collects metrics; Grafana visualizes; Loki stores logs; Alloy collects pod logs; Tempo is the trace backend.
- `x-trace-id` is propagated and logged as `trace_id`. This provides log correlation, not full distributed tracing by itself. Add OpenTelemetry SDK/exporter instrumentation before claiming trace ingestion.
- Keep all credentials in GitHub Actions Secrets or Kubernetes Secrets, never in Git.
