---
title: Setup environment for end-to-end testing
parent: Testing
layout: default
nav_order: 3
---

# Setup environment for end-to-end testing

When running end-to-end tests, you'll need to manage external dependencies like databases, message queues, and other services. We have these two options:

## Docker Compose

Using Docker Compose for test dependencies:

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  test-db:
    image: postgres:14
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: test_db
    ports:
      - '5432:5432'
```

**Pros:**

- Simple setup with declarative configuration
- Good for local development environments
- Can start all dependencies with a single command
- Easier to debug as containers persist between test runs

**Cons:**

- Manual cleanup required
- State can persist between test runs, leading to flaky tests
- Harder to manage in CI/CD pipelines
- Port conflicts can occur when running parallel tests

## [Testcontainers](https://testcontainers.com/)

Using Testcontainers for programmatic container management:

```typescript
describe('Users API (e2e)', () => {
  let postgres: PostgreSQLContainer;

  beforeAll(async () => {
    postgres = await new PostgreSQLContainer()
      .withDatabase('test_db')
      .withUsername('test')
      .withPassword('test')
      .start();

    // Configure your app to use the container's connection details
    process.env.DATABASE_URL = postgres.getConnectionUri();
  });

  afterAll(async () => {
    await postgres.stop();
  });
});
```

**Pros:**

- Containers are managed programmatically as part of the test lifecycle
- Automatic cleanup after tests
- Better parallel test execution support
- Works well in CI/CD environments

**Cons:**

- Slightly more complex setup
- May require more resources when running many parallel tests

### Recommendation

Consider using **Testcontainers** when:

- You need reliable, isolated test environments
- You're running tests in CI/CD pipelines
- You want to avoid test flakiness due to shared state
- You need to run tests in parallel

Use **Docker Compose** when:

- You're primarily running tests locally
- You need a persistent development environment
- You want to manually inspect the state of dependencies
- You have a simple setup with few dependencies
