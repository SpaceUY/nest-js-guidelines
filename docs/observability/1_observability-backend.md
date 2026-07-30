---
title: Observability backend: Grafana
parent: Observability
layout: default
nav_order: 1
---

# Observability backend: Grafana

Grafana isn't one piece — it's several, each covering a different signal, and they aren't interchangeable:

| Piece | Signal | Notes |
|---|---|---|
| **Grafana** | Visualization | The dashboard. Stores nothing itself — reads from Loki, Tempo, and Prometheus/Mimir and displays them in one place. |
| **Prometheus / Mimir** | Metrics | Numeric time series (CPU, requests/sec, latency). Prometheus scrapes a `/metrics` endpoint; Mimir is the scalable/multi-tenant version Grafana Cloud uses at volume. |
| **Loki** | Logs | Indexes only metadata/labels, not full text — much cheaper to operate than Elasticsearch at the same volume. |
| **Tempo** | Traces | The full path of a request across services, with per-hop timing. Stored on S3 to keep retention cheap. |
| **Alloy** | Collection | The agent that runs alongside the app/server and forwards logs/metrics/traces to Loki/Prometheus/Tempo — one agent instead of one per signal. |

Grafana is also not just for infrastructure: any simple business metric exposed as a counter/gauge (e.g. transactions processed) can be graphed on the same dashboard as infra metrics, with no separate BI tool needed for that kind of quantitative signal.

## Self-hosted vs. Grafana Cloud

Both are valid options — which one to use is a decision to make with the client, since pricing (self-hosted infra cost vs. Grafana Cloud's usage-based pricing) is the main factor between them and depends on the project's budget and expected volume.

| | Self-hosted (LGTM stack) | Grafana Cloud (hybrid) |
|---|---|---|
| **Who operates it** | We do — install Loki/Tempo/Prometheus/Grafana on EC2/ECS, scale, update, back up, manage IAM | Grafana — nothing installed on our side |
| **Effort** | High — four distributed systems to tune, scale, and upgrade | Low — same dashboards, no infrastructure of our own to operate |
| **Deployment options on AWS** | (a) single EC2 (min. `t3.large`) with Docker Compose, for MVPs; (b) ECS Fargate with Alloy as a sidecar | App still runs on our own AWS infra; only the observability backend is managed |

Grafana Cloud accepts OTLP directly from the app, with no Alloy collector in the middle — the first step into the hybrid model can be just changing two environment variables (endpoint + auth token), with zero new infrastructure. Alloy is the recommended path for production at real scale, but it isn't a requirement to get started.

Whichever option is used, it doesn't change a line of application code: the app is instrumented with OpenTelemetry the same way in both cases, and only the collector's destination changes — see [Choosing a backend](./3_telemetry.md#choosing-a-backend) in the telemetry guide.
