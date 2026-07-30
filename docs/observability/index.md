---
title: Observability
layout: default
nav_order: 3
---

# Observability

This section covers how to make a NestJS backend observable: infrastructure/application monitoring via **Grafana**, product analytics via **PostHog**, and how to instrument a NestJS service with **OpenTelemetry** so both are fed correctly. Infrastructure provisioning on AWS (how the observability backend is actually deployed) is covered in the [DevOps Guidelines](https://github.com/SpaceUY/devops-guidelines) — this section is the application-level standard.

1. [Observability backend: Grafana](./1_observability-backend.md)
2. [Product analytics: PostHog](./2_product-analytics.md)
3. [Telemetry: OpenTelemetry instrumentation in NestJS](./3_telemetry.md)

## Recommended stack

| Layer | Tool | Role |
|---|---|---|
| **Telemetry** | [OpenTelemetry](https://opentelemetry.io/) | Generates traces, logs, and metrics from inside the app in a vendor-neutral way. Decouples the code from whichever backend reads that data. |
| **Observability backend** | [Grafana Cloud](https://grafana.com/products/cloud/) (hybrid model) | Managed Tempo (traces) + Loki (logs) + Prometheus/Mimir (metrics), read through Grafana dashboards. |
| **Product analytics** | [PostHog](https://posthog.com/) | Events, funnels, session replay, feature flags, experiments, and web analytics for authenticated product usage. |

## Why this stack

This is not a decision made in a vacuum — see the full [Observability & Product Analytics research](https://claude.ai/code/artifact/5a3b6cdf-7cef-44ec-9de6-9b34bc883961) for the complete provider-by-provider comparison this summary is based on.
