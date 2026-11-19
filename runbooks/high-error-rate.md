# High error rate

## Response

1. Identify the service in Grafana; query Loki for errors; inspect recent GitOps changes and downstream service health; revert the GitOps commit if a release caused the issue.

## Recovery verification

Confirm the alert clears, affected workloads are healthy, and record the cause and remediation.
