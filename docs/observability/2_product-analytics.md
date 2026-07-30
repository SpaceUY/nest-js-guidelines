---
title: Product analytics: PostHog
parent: Observability
layout: default
nav_order: 2
---

# Product analytics: PostHog

PostHog doesn't compete with Grafana — it competes with Amplitude, Mixpanel, Segment, GA4, and Heap on product analytics: events, funnels, session replay, feature flags, and A/B testing for **authenticated** product usage. It has no relationship to infrastructure health.

> **Before integrating PostHog on a project, talk to the client.** Its pricing is usage-based (per event, replay, feature flag request, etc.), so adopting it — and later deciding what to track — is a conversation to have upfront, not an implementation detail to decide unilaterally.

- **Model:** open-core (MIT), self-hosted or cloud.
- **NestJS/Node integration:** `posthog-node` for server-side events, `posthog-js` for the frontend.
- **All-in-one:** analytics + session replay + feature flags + experiments + error tracking + LLM observability + web analytics on a single product — every tool that isn't added separately is one less integration, login, and contract to maintain.

## Implementation in the NestJS Template

The [NestJS Template](https://github.com/SpaceUY/NestJS-Template) implements this as an abstract analytics module under `common/observability/analytics`, following the same [Adapter Pattern](../standard-definitions/2_nestjs-template.md) used for Email, Cloud Storage, and Push Notifications: a common abstract class defines the analytics interface, and provider-specific subclasses implement it.

- **`PostHogAdapterService`** — the real adapter, backed by `posthog-node`, sends events to PostHog.
- **`ConsoleAdapterService`** — logs events to the console instead of sending them anywhere. Used for local development and initial testing without needing a PostHog project configured.

Because both sit behind the same abstract class, switching between them (or adding a new provider later) is a matter of swapping which adapter gets injected — the rest of the application only depends on the abstract interface.

Inject the `AnalyticsService` token in other modules — never a concrete adapter class — so the adapter can be swapped without touching consumers. Its main methods:

- **`capture(input)`** — sends a product event (e.g. "order created", "user upgraded plan") tied to a `distinctId`.
- **`isFeatureEnabled(key, distinctId)`** — checks whether a boolean feature flag is on for a given user.
- **`getFeatureFlag(key, distinctId)`** — reads a flag's value, including non-boolean/multivariate flags.

## What to track from the backend

Backend events are for things that need to be reliable and can't depend on the frontend actually running (ad blockers, closed tabs, failed requests) — typically business-critical or lifecycle events, for example:

- Account/order lifecycle: `user signed up`, `order created`, `payment succeeded` / `payment failed`, `subscription upgraded` / `downgraded` / `cancelled`.
- Background/async outcomes: a queued job finishing (report generated, export ready, webhook received from a third party).
- Security-sensitive actions: `password reset requested`, `MFA enabled`, `API key created` / `revoked`.
- Server-side feature flag checks that gate business logic (not just UI), so you can see which variant actually ran for a given request.

Which events to track should be agreed with the client — PostHog's pricing is usage-based per event, so tracking everything by default can turn into a real cost driver. Decide with the client which events matter for their product/business rather than instrumenting every possible action.

## Web analytics

PostHog also offers Web Analytics — anonymous traffic/marketing metrics (pageviews, UTM sources, bounce rate), as opposed to the authenticated product usage covered above. It's instrumented on the frontend — see the [React Guidelines](https://spaceuy.github.io/react-guidelines) for the standard there.
