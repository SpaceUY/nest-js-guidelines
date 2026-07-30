---
title: "Telemetry: OpenTelemetry instrumentation in NestJS"
parent: Observability
layout: default
nav_order: 3
---

# Telemetry: OpenTelemetry instrumentation in NestJS

This is the practical, code-level piece of the observability standard: how to think about tracing with [OpenTelemetry](https://opentelemetry.io/) in a NestJS service, independent of which backend ([Grafana](./1_observability-backend.html), SigNoz, Datadog, etc.) ends up reading the data. It has already been implemented and validated in the [NestJS Template](https://github.com/SpaceUY/NestJS-Template), under `common/observability/telemetry`.

## The goal: instrument once, not again for the next incident

"Observable" means: when an unknown failure mode shows up in production, you can answer "why" from the data you're already collecting — not by SSH-ing in, adding a `console.log`, and redeploying. That means capturing high-cardinality context up front, because you can't predict which attribute will matter for a bug you haven't seen yet.

## Traces first, then logs, then metrics

**Logs** are events you decided in advance were worth recording, with whatever fields occurred to you at the time. **Traces** capture the actual shape of a request's execution — every hop, every dependency call, with timing — and context is attached as attributes you can filter on after the fact, not before.

For a team starting from "configurable logger, no tracing," the highest-impact order is: **traces first → wire the existing logger to emit trace context → metrics last.** Metrics without traces tell you something is wrong. Traces without metrics still let you find out why. The reverse is much less true.

## Auto-instrumentation is the floor, not the ceiling

Auto-instrumentation packages patch common libraries (HTTP in/out, `pg`, Redis, etc.) to emit spans automatically, with no code changes. It gives immediate infrastructure visibility: every HTTP call and DB query becomes a span. It's the first step, and the highest return for the effort. We only register the specific instrumentations our stack actually uses — see [Bootstrap implementation](#bootstrap-implementation) below.

It does not capture the shape of your business logic: it tells you "an HTTP call to the payments service took 400ms," not "we spent 300ms recalculating the user's discount tier for this edge case." That needs manual spans around meaningful units of work — service methods, use-case handlers, the seams of your domain.

## A method is a unit of work — but not every method deserves a span

A span represents a unit of work with a start, an end, and a meaningful name — something that deserves its own bar in a waterfall view. NestJS service methods are usually already factored this way: `OrdersService.createOrder()`, `PaymentService.chargeCard()`, `DiscountService.calculateTier()`.

Span a method if at least one of these is true:

- It crosses a process/IO boundary (DB, HTTP, queue publish) — though auto-instrumentation usually already covers this.
- It represents a distinct business step worth measuring on its own ("tier calculation took 280ms" is useful; "the private helper that adds two numbers took 0.001ms" is not).
- It's a place where you'll attach attributes only known/computed there (`discount.tier`, `payment.provider`, `order.itemCount`) — even if the method is fast, the attribute is the value, not the timing.
- It's a branch point you actually debug in practice — somewhere you'd ask "which path did this take?"

Don't span: pure functions/private helpers with no IO or decision-relevant state, trivial getters/mappers/DTOs, or anything called inside a tight loop (wrap the loop itself in one span with an `items.count` attribute instead of N spans).

## Two implementation layers

**Layer 1 — free:** a NestJS interceptor wraps handler execution (or any DI-managed provider) in a span automatically, named by class/method, with no per-method code. Gives trace boundaries for every entry point at zero ongoing effort. This layer is already covered by the official `@opentelemetry/instrumentation-nestjs-core` package (imported as `NestInstrumentation`) — no in-house interceptor needed.

**Layer 2 — intentional:** explicit instrumentation inside the methods chosen per the criteria above — grab the active span and add attributes/events, or open a child span if the method has sub-steps worth separating. In-house pattern: a `@Span()` decorator (~20 lines, depends only on `@opentelemetry/api`) that wraps the method in `tracer.startActiveSpan(...)`, records exceptions and error status, closes the span in a `finally`, and lets the method body call `trace.getActiveSpan()?.setAttribute(...)` to attach business data.

## No third-party NestJS wrapper

Two community wrappers were evaluated and discarded: `amplication/opentelemetry-nestjs` and `pragmaticivan/nestjs-otel`. Both show the same abandonment pattern as the forks they replaced (e.g. last release March 2024, fork-of-a-fork lineage). No NestJS OTel wrapper lives under the official `open-telemetry` GitHub org — that's a project design decision, not an oversight specific to NestJS.

Use only official packages:

- `@opentelemetry/instrumentation-nestjs-core` (`NestInstrumentation`) for automatic controller/guard/interceptor/pipe spans.
- A small in-house `@Span()` decorator for manual spans, depending only on `@opentelemetry/api` — the most stable package in the ecosystem.
- Explicitly-named instrumentations (`HttpInstrumentation`, `PgInstrumentation`, `IORedisInstrumentation`, etc.) instead of the `auto-instrumentations-node` meta-bundle, which pulls in ~50 instrumentations for libraries not in our stack (Kafka, Cassandra, GraphQL...).

## Bootstrap implementation

This is what `common/observability/telemetry` in the NestJS Template looks like — it builds the SDK explicitly instead of pulling in `@opentelemetry/auto-instrumentations-node`:

```ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { resourceFromAttributes } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME } from '@opentelemetry/semantic-conventions';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';
import { NestInstrumentation } from '@opentelemetry/instrumentation-nestjs-core';
import { PgInstrumentation } from '@opentelemetry/instrumentation-pg';
import { IORedisInstrumentation } from '@opentelemetry/instrumentation-ioredis';
import { getOtelConfig, OtelConfig } from './otel-env';

export function buildSdk(config: OtelConfig): NodeSDK | null {
  if (!config.enabled) return null;

  return new NodeSDK({
    resource: resourceFromAttributes({
      [ATTR_SERVICE_NAME]: config.serviceName,
    }),
    traceExporter: new OTLPTraceExporter({
      url: config.endpoint,
      headers: config.headers,
    }),
    instrumentations: [
      new HttpInstrumentation(),
      new NestInstrumentation(),
      new PgInstrumentation(),
      new IORedisInstrumentation(),
    ],
  });
}

const sdk = buildSdk(getOtelConfig());

if (sdk) {
  sdk.start();

  process.on('SIGTERM', () => {
    sdk
      .shutdown()
      .catch((err: unknown) => {
        console.error('Error shutting down OpenTelemetry SDK', err);
      })
      .finally(() => process.exit(0));
  });
}
```

`getOtelConfig()` derives `config.enabled` from whether `OTEL_EXPORTER_OTLP_ENDPOINT` is set — if it isn't, `buildSdk` returns `null` and the app starts exactly as it would without OTel. This file is the dedicated bootstrap entrypoint referenced below: it has to be imported before anything else.

## The most common mistake: initialization order

The Node OTel SDK relies on `AsyncLocalStorage` to track the active span across async boundaries, which works fine with Nest's DI and lifecycle. The real gotcha is different: auto-instrumentation patches modules at `require`/`import` time, so **the SDK must initialize before anything else is imported**. The typical bug is initializing OTel inside `main.ts` after other imports have already loaded. Use a dedicated bootstrap file, loaded first (via `--require`, or as the literal first import of the entrypoint) — e.g. `tracing.bootstrap.ts` imported as the first line of `main.ts`.

## Logging becomes the span's narrator

Two concrete changes:

1. **Inject `trace_id`/`span_id` into every log line** — pull the active span from context in your logger (interceptor, middleware, or a log processor) and attach its IDs to every entry. This is the single biggest upgrade over logging blind: instead of loose lines, you open the trace and see every log emitted during that span.
2. **Prefer span attributes/events over log lines for structured data.** Logs are better for unstructured narrative ("retrying after backoff"); attributes are better for anything you'll later filter or group by (`user.tier`, `payment.amount`, `cache.hit`). A log line like "user 4821 hit the rate limit" is far less useful than a span attribute `user.id=4821` you can pivot on across thousands of traces in the backend's UI.

In the template, this is implemented as a `TraceContextLoggerDecorator` that wraps the configured logger adapter (Nest/Pino/Winston, unchanged) to stamp `traceId`/`spanId` on every log line and forward the log as `span.addEvent()`.

## Sampling: a deliberate decision, not a default

**Head-based:** decide at request start (e.g. keep 10%). Simple, but you can silently drop the one trace you needed. **Tail-based:** decide after the request finishes (e.g. keep all errors and slow requests, sample the rest). Closer to "never need to instrument again," but requires an OTel Collector in the path instead of exporting straight from the SDK to the backend.

To start: export everything during the initial rollout, introduce sampling once volume becomes a cost problem. Don't optimize this ahead of time.

## Choosing a backend

The OTel SDK is vendor-neutral (OTLP exporter) — instrumentation code only ever talks to `OTEL_EXPORTER_OTLP_ENDPOINT`/`OTEL_EXPORTER_OTLP_HEADERS`, never to a specific vendor's SDK. That's the practical payoff of standardizing on OpenTelemetry: which backend to use is a separate, genuinely reversible decision — see [Observability backend](./1_observability-backend.html) for how we choose and set it up (self-hosted Grafana vs. Grafana Cloud). If that choice changes later, moving to it is a config change (updating the endpoint and headers), not a rewrite of the instrumentation.

To start locally: [Jaeger](https://www.jaegertracing.io/) (`jaegertracing/all-in-one`) added to `docker-compose.yml` — UI on `:16686`, OTLP HTTP on `:4318`. Moving from local Jaeger to the chosen backend later doesn't touch a line of instrumentation code — only the collector's destination changes.

## What's already done vs. what's next

**Done** (`common/observability/telemetry` in the NestJS Template): SDK bootstrap with explicitly-named official instrumentations, no third-party wrapper, the `@Span()` decorator, two-way log correlation via `TraceContextLoggerDecorator`, and a local Jaeger backend in `docker-compose.yml`.

**Next, when a project needs it:**

- Apply `@Span()` to paths that would page someone if they broke, using the criteria above — the decorator exists and is tested, but deliberately isn't applied to any real method yet without a concrete use case.
- Add span attributes liberally on those manual spans — over-attributing now is cheaper than guessing later.
- Migrate the backend from local Jaeger to Grafana Cloud or Amazon Managed Grafana + Prometheus — config-only change.
- Add metrics only for what you'll actually alert on (latency SLOs, error rates) — last, not first.
- Revisit sampling once real volume/cost shape is understood.
- Audit PostHog event volume against the active pricing tier — usage-based pricing at scale is the most concrete cost risk identified.
