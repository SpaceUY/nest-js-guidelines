---
title: Serverless Backend Architecture
parent: Architecture
layout: default
nav_order: 4
---

# Serverless Backend Architecture

## Summary

Our default backend deployment is a container on ECS Fargate behind an Application Load Balancer. It is a good default, but it bills for compute around the clock, including for APIs that sit idle most of the day.

This document looks at an alternative for low-traffic backends: **one Lambda function per bounded domain module**, each running a trimmed application, fronted by a **single API Gateway HTTP API** that does pure request routing.

The pattern is not hypothetical, as one of our projects already runs this way in a development environment, with six functions behind one gateway. This document writes down what that deployment actually is, explains why the model works mechanically, and lays out the tradeoffs so a team can decide per project.

## Starting point

### The default

Each backend is an always-on container service on ECS Fargate, behind an ALB, provisioned with SST. Capacity is fixed by the service definition and autoscaling policy, and cost does not follow request volume.

### A reference deployment

A Lambda-based project is set up like this:

- One API Gateway HTTP API with multiple `AWS_PROXY` integrations, one per Lambda function.
- Each function is packaged as a container image with its own ECR repository, all pinned to the same release tag, all in the same VPC, subnet and security group.
- None of the IAM roles carry ECS permissions. The gateway invokes Lambda directly and nothing proxies to ECS.
- Async work goes through SQS to a separate worker, not through another Lambda-to-service hop.

```text
 Client
   |
   v
  +-----------------------------+
  | API Gateway (HTTP API)      |
  | path + method routing only  |
  +-----------------------------+
    |    |    |    |    |    |      AWS_PROXY, direct invoke
    v    v    v    v    v    v
  +----+----+----+----+----+----+
  | fn | fn | fn | fn | fn | fn |   own container image + ECR repo each
  | 1  | 2  | 3  | 4  | 5  | 6  |   same release tag, VPC, subnet, SG
  +----+----+----+----+----+----+
                 |
                 | enqueue async work
                 v
            +---------+     +--------------+
            |   SQS   |---->| batch worker |
            +---------+     +--------------+
```

## Lambda execution

To fully understand this strategy and its tradeoffs, it's very instructive to look what happens under the hood when a Lambda function is executed.

Lambda invocation does not start a fresh process per request. AWS provisions an **execution environment** (a microVM) and runs the module-level init code **once**.

> That is what's called the **cold start**.

Between invocations the environment is **frozen**, not destroyed. A later request that lands on the same frozen environment is a **warm start**: only the handler runs, and all initialization code is skipped. AWS eventually tears the environment down after an idle period it controls.

For NestJS in particular, this has one hard rule. `NestFactory.create(...)` must be called once at module scope and the resulting application context cached. If it is called inside the handler, every invocation re-bootstraps the whole DI container and the warm path loses most of its benefit.

Concurrency also works differently from a container. N simultaneous requests create up to N environments in parallel, each handling one invocation at a time. On Fargate, a single process handles many concurrent connections itself, which is an advantage at **sustained load**.

## The pattern

We now discuss the serverless architecture in itself, and some important technical details.

### Entrypoint

Instead of calling Lambdas directly, we use **API Gateway as the entry point**. The gateway handles TLS termination, path and method routing to a specific function, marshalling to the Lambda event shape, and optionally throttling, request validation and JWT authorizers. It runs no business logic.

**It fully replaces the ALB**. The ALB routes a path to a target that is a listening process. The gateway routes a path to a function it invokes directly. Whatever ECS service, target group and listener definitions a module had are no longer needed once it moves.

Authentication could also move out of the application and **into a gateway authorizer**. That is an option, not a requirement of the pattern, but it allows for a clean separation of responsibilities.

### Granularity

A finer option we might consider is to have **one Lambda per route**. That would provide the tightest isolation and the most precise scaling, since a hot endpoint never competes with a cold one and a bad deploy touches only one route.

However, this has a hidden cost: the application bootstrap (especially in the case of NestJS, where we have module resolution, DI wiring, guards, pipes) is roughly fixed per execution environment however few routes it serves. Splitting to single handlers **pays that cost more often for the same amount of logic**.

> It also multiplies everything that scales with function count: artifacts, IAM roles, route mappings, log groups, alarms.

One function per **bounded domain module** is a better compromise solution.

> In the case of NestJS, it matches its own architectural boundary, because we work with modules! The bootstrap cost is paid **once per module**, and amortized over every route inside it on warm invocations, so a module with more routes gets cheaper per route.

But as they say, nothing is free! The cost here is **shared fate**. Routes in the same function share its concurrency, so a spike or a slow dependency on one route can throttle its neighbours, and a bad deploy hits every route in the module at once. At low per-tenant traffic, this rarely matters. It does argue however for keeping modules cohesive and not bundling unrelated routes just to reduce the function count.

### Grouping modules into functions

Do not map one NestJS module folder to one function by default. The reference deployment groups related concerns by call pattern (for example a business-logic function, a backoffice function, a catalog function, etc). A useful grouping considers which modules share heavy dependencies such as signing keys or external SDKs, and which have very different traffic profiles. This needs real call-pattern data. Folder names alone are not enough.

> Scheduled work with no HTTP surface, such as cron-style jobs and sweepers, is the easiest first candidate. It needs no gateway at all!

Cross-cutting code (auth, notifications, task runners, shared providers) should usually ship as an internal package imported by whichever function needs it, not as its own function.

### Multi-tenant setups

If each tenant today gets its own backend, database and domain, the Lambda version should keep that isolation. The natural unit is then **one API Gateway per tenant**, mirroring a per-tenant target group and listener rule behind a shared ALB. Several apps can sit behind the same tenant gateway through base path mapping on the tenant's domain. That means fewer resources to manage, at the cost of coupling those apps to one routing and deploy surface. It is the same shared-fate tradeoff as grouping routes into one function, one level up.

Tenant onboarding changes shape. Instead of adding one DNS record and one service behind an existing load balancer, it adds gateway, custom domain, certificate and base path wiring per tenant.

## Cost

### What goes away

- **Idle compute.** No task runs or bills between requests.
- **The ALB.** It has a fixed hourly charge plus LCU usage regardless of load. HTTP APIs have no base hourly charge.
- **Autoscaling policy and service lifecycle** for the modules that move.

### What replaces it

- **Usage-based billing.** Lambda per invocation and API Gateway per request. Cost now tracks traffic instead of staying flat, which is the point at low and bursty volume and a liability at steady high volume.
- **Cold starts.** A latency cost an always-on service does not have.
- **Connection pooling.** Every warm environment can hold its own database connection, so concurrency multiplies connections against a fixed Postgres limit. RDS Proxy or an equivalent is new infrastructure to build and run, not a config change.
- **Per-tenant gateway and domain management**, where tenants are isolated as above.
- **VPC networking, unchanged.** Functions that reach a private database or cache still need VPC attachment, so NAT and ENI costs carry over. This is not a new cost, but it is also not a saving.

## Consequences

- **Cost** is meaningful only for domains with genuinely low, bursty traffic. For steady load it is worth nothing, and cold starts make it worse.
- **Database connections** need an explicit answer (RDS Proxy, or a small pool `max` per function plus IAM auth) before any real concurrency.
- **Operational surface** trades one cluster, service and ALB for N functions plus the gateway(s), each function with its own role, log group and artifact. The idle footprint is smaller and there are more discrete pieces to wire. SST supports both models, so this is configuration and not a new toolchain.
- **Bootstrap discipline** decides whether the warm path is fast. Caching the Nest context at module scope is a correctness requirement.
- **Persistent connections do not fit.** Anything built on WebSockets or long-lived streams needs a different design (API Gateway WebSocket API with connection IDs stored externally and server push via `postToConnection`) or stays on containers. A backend that mixes REST domains with a WebSocket surface becomes a hybrid, and that seam is permanent.

## When it fits

Use Lambda-per-module when most of these are true:

- Traffic is low or bursty, with real idle periods.
- The API is request/response, with no persistent connections.
- Occasional cold-start latency is acceptable for the affected endpoints.
- The modules are cohesive and can be grouped into a handful of functions.
- A database pooling answer exists, or the database can tolerate the connection pattern.

Stay on ECS Fargate when traffic is steady or high, when latency must be consistently low, or when the service depends on WebSockets or other long-lived connections.

## Decisions to make per project

- How should a large module set group into functions? Real traffic and dependency data per feature area is needed.
- Which pooling approach per function: RDS Proxy in front of Postgres, or a smaller pool plus IAM database auth?
- How is shared code (DTOs, guards, interceptors, common providers) packaged so each function bundles only what it needs? A workspace package, a layer, or something else.
- Container images (parity with existing Docker-based pipelines, matches the reference deployment) or esbuild zip bundles (smaller artifacts and faster cold starts, different build pipeline)?
- Where tenants are isolated: one gateway per tenant, or one per tenant shared across apps via base path mapping? Prototype both against a real onboarding to see how much stays automated.
- Should this be proven as a proof of concept on one scheduled-job module before it becomes standing practice?
