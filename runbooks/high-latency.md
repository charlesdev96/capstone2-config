# High p99 latency

## Response

1. Identify the affected service; inspect CPU/memory and downstream latency; search Loki for timeouts; revert the change if a recent release caused the regression.

## Recovery verification

Confirm the alert clears, affected workloads are healthy, and record the cause and remediation.
