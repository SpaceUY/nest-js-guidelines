---
title: Standard technology definitions
parent: Company’s Standard Technology Definitions
layout: default
nav_order: 1
---

# Company’s standard technology definitions

The following standards have been defined at the company level through chapter meetings, based on analyses and discussions presented by the team. These standards serve as the default technologies and tools for our projects, but may vary depending on specific project or client requirements.

- **Email Delivery:** [Resend](https://resend.com/) is our standard service for sending emails.
- **Relational ORM:** [TypeORM](https://typeorm.io/) is used for relational database management.
- **Non-Relational ORM:** [Mongoose](https://mongoosejs.com/) is the preferred choice for working with non-relational (NoSQL) databases.
- **Backend Framework:** [NestJS](https://nestjs.com/) is our standard backend framework for building scalable server-side applications.
- **Package Manager:** [pnpm](https://pnpm.io/) is our standard package manager.
- **HTTP Adapter:** The NestJS [`NodeAdapter`](https://docs.nestjs.com/http/performance) (`@nestjs/platform-node`) is our default HTTP adapter. Express is no longer the default. See [HTTP adapter](#http-adapter) below.
- **Infrastructure:** [AWS](https://aws.amazon.com/) is our standard platform for cloud infrastructure and services. See the [DevOps Guidelines](https://github.com/SpaceUY/devops-guidelines) for deployment and infrastructure documentation.
- **Observability:** [Grafana](https://grafana.com/) (self-hosted or Grafana Cloud) for infrastructure/application monitoring, [PostHog](https://posthog.com/) for product analytics, and [OpenTelemetry](https://opentelemetry.io/) for instrumenting the backend. See the [Observability section](../observability/index.html) for the full standard.
- **Chapter Board:** We use the [ClickUp chapter NestJS board](https://app.clickup.com/3117051/v/o/s/90142470417) to manage and track chapter-related activities and standards.
- **Authentication:** [Auth0](https://auth0.com/) is the standard solution for authentication and authorization.
- **Repository Hosting:** [Bitbucket](https://bitbucket.org/) is our standard platform for hosting code repositories.
- **Frontend Framework:** [React](https://react.dev/) is the standard for building web frontends.
- **Mobile Development:** [React Native](https://reactnative.dev/) is used for mobile application development.

> **Note:** These standards are established at the company level but may be adapted or changed according to the specific needs of a project or client.

## HTTP adapter

New NestJS projects at Space use the **`NodeAdapter`** (`@nestjs/platform-node`), which runs Nest directly on Node's `node:http` module. Express (`@nestjs/platform-express`) is no longer our default.

> Nest itself still falls back to Express when no adapter is passed to `NestFactory.create()`, so the adapter must always be set explicitly in `main.ts` and in e2e tests.

### Why

- **Space benchmark (2026-11-06):** we benchmarked the available adapters internally and the `NodeAdapter` came out ahead of Express. See the [full results](https://app.clickup.com/3117051/v/dc/2z3zv-155374/2z3zv-97354).
- **Official NestJS documentation:** the [HTTP adapters](https://docs.nestjs.com/http/performance) page recommends the `NodeAdapter` starting with NestJS v12.3 and states it will become Nest's own default in v13. Its benchmark reports about 50% more requests per second than Express and about 5% more than Fastify.
- The adapter only handles a small part of each request, so the gain matters most for lightweight routes at high request rates. Database-bound routes are dominated by waiting, so do not expect the same difference there.

### Setup

```bash
pnpm add @nestjs/platform-node
```

```ts
import { NestFactory } from '@nestjs/core';
import { NestNodeApplication, NodeAdapter } from '@nestjs/platform-node';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestNodeApplication>(
    AppModule,
    new NodeAdapter(),
  );
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

In e2e tests, pass the adapter as well. Otherwise the test app falls back to Express and fails if `@nestjs/platform-express` is not installed:

```ts
app = moduleRef.createNestApplication<NestNodeApplication>(new NodeAdapter());
```
