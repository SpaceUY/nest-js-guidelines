---
title: Project structure
parent: Architecture
layout: default
nav_order: 2
---

# Project structure

Defining rules for organizing projects is generally a good practice to avoid clutter accumulation as projects grow larger. That being said, not every project has the same level of complexity - so, in SpaceDev, we propose the usage of _two_ different folder organization patterns:

- For simple projects, we use the (simple) suggested NestJS folder structure.
- For more complex projects - especially those that are expected to scale and/or run for a long time, we propose the use of Domain Driven Design (DDD).

## Modules

Throughout the overview of the [NestJS docs](https://docs.nestjs.com/), folder structure is implicitly driven by _modules_.

Of course, modules are actual elements in the NestJS architecture - they incorporate various elements (controllers, providers, other modules) into a single unit, which can then be included in other parts of the application.

Thus, folder structure will look like this:

```
src
└───modules
│   └───module-a
│   └───module-b
│   └───module-c
└───common
└───...
```

Each module can be thought of as an autocontained _feature_. And since modules are to be composed via inclusion, it's useful to try and organize them in a way where dependencies flow _forward_, to avoid circular dependencies (which [can be resolved](https://docs.nestjs.com/openapi/types-and-parameters#circular-dependencies), but are not ideal).

> 👉 By "flow forward", what we mean is that the module dependency graph is actually a Directed Acyclic Graph (DAG).

Both design approaches will use this modular approach. The difference will lie in how we organize _each module_.

---

## Structure for simple projects

> 👉 By "simple project", we mean a project that is either not expected to scale too much, or that is expected to run for a limited amount of time, and then be discarded.

When a project meets these specifications, we can basically treat each module as a simple container where we place everything that belongs to it.

For a module with a single controller, a single service, and nothing but a single entity, the structure may look like this:

```
src
└───module-a
    └───module-a.module.ts
    └───module-a.controller.ts
    └───module-a.service.ts
    └───entity-1.entity.ts
```

> 👉 By convention, we use a file extension that incorporates the type of element into the filename, such as `.module.ts`.

Notice how everything lives at the same level. This is fine for modules that have this one-of-each behavior.

However, it's often the case that a module contains multiple entities, services, or controllers - or other types of providers. When that's the case, the suggestion is to group them into folders:

```
src
└───module-a
    └───module-a.module.ts
    └───controllers
    |   └───controller-1.controller.ts
    |   └───controller-2.controller.ts
    └───services
    |   └───service-1.service.ts
    |   └───service-2.service.ts
    └───entities
        └───entity-1.entity.ts
        └───entity-2.entity.ts
```

It's useful to add `index.ts` files as well in each of these subfolders.

Modules should be as self-contained as possible, so if there are other elements around them - helpers, constants, etc. -, consider including them here as well. This goes for both this approach, and the one following below.

---

## Structure for complex projects

More complex projects may see parts of their infrastructure or their presentation (API inputs and outputs) change through their lifecycle. In this context, it would be desirable to have an invariant _core_ that describes application business logic - and have the I/O and infrastructure layers operate at interface level, so that they can be switched in the future.

So, we take elements of [Domain Driven Design](https://en.wikipedia.org/wiki/Domain-driven_design) (DDD) and the [Hexagonal Architecture](<https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)>) to propose this folder structure.

> 💡 It's very useful to read about [abstract interfaces](./abstract-interfaces.html) in order to make the most out of this design choice.

Instead of being a simple container, each module is now separated into three very distinct parts:

- **Application layer**: Everything related to I/O - presentation, controllers, etc. This _consumes elements_ from the **domain layer** to handle business logic.
- **Domain layer**: Contains all the _business logic_ associated to a module. It also contains _models_ for the different entities handled and consumed by the application logic.
- **Infrastructure layer**: All the elements related to externally consumed services that serve as infrastructure. Things such as database models, mailing templates, etc. live here. They are not to be confused with the models in the **domain layer** - in fact, explicit mappings should be put in place.

At the coarsest level of organization, folder structure will look like this:

```
src
└───module-a
    └───application
    |   └─── ...
    └───domain
    |   └─── ...
    └───infrastructure
        └─── ...
```

Let's now explore what should go inside each of these locations.

### Application Layer

Crucially, this is where _controllers_ and _DTOs_ (input / output) should live.

> 🚨 DTOs (Data Transfer Objects) generate a bit of confusion in terms of naming. The [NestJS docs](https://docs.nestjs.com/controllers#request-payloads) correctly states that they are objects to be sent over a network, but then proceeds to show use cases where they are only used for _inputs_ - and dedicates an [entire section](https://docs.nestjs.com/techniques/serialization#overview) to _outputs_, which they call **Serializers**.
>
> We shall simplify and only talk about input and output DTOs.

Furthermore, if the application you're building is organized in different _clients_ (i.e. separate apps for admins, clients, operations, etc.), it's possible that client-specific modules live here as well.

A non-client-discriminated example would look like this, using `purchases` as an example module:

```
purchases
└───application
    └─── purchases.module.ts
    └─── controllers
    └─── dtos
         └─── input
         └─── output
```

While a complete, client-separated example would look like this:

```
purchases
└───application
    └─── admin
    |    └─── admin.purchases.module.ts
    |    └─── controllers
    |    └─── dtos
    |         └─── input
    |         └─── output
    └─── customer
    |    └─── customer.purchases.module.ts
    |    └─── controllers
    |    └─── dtos
    |         └─── input
    |         └─── output
    └─── ...
```

Note that entities and services do not live under `/application`.

### Domain Layer

The domain layer represents the core business logic and rules of an application, completely isolated from external concerns like presentation or data storage. This is where the actual business problems are solved.

> 👉 A good rule of thumb: if you can't explain a concept in the domain layer to a business stakeholder without mentioning technical terms, it probably doesn't belong here.

The standard structure for the domain layer looks like this:

```
purchases
└───domain
|   └─── events.ts
|   └─── constants.ts
|   └───enums
|   └───exceptions
└───models
|   └───purchase.model.ts
|   └───receipt.model.ts
└───services
|   └───purchase.service.ts
|   └───receipt.service.ts
└───interfaces
|   └───persistence
|   └───messaging
└───...
```

#### Models vs Entities

This is mentioned through this article, but still, being as important as it is:

> 🚨 Don't confuse domain models with infrastructure entities!!

Domain models represent business concepts and rules, while entities are typically ORM-specific implementations for persistence. Domain models may be rich in business logic and behavior, coupled with services at business logic level. Entities, on the other hand, are primarily focused on data structure and storage.

For example:

```ts
// Domain model
export class Purchase {
  private constructor(
    private readonly id: string,
    private readonly items: PurchaseItem[],
    private status: PurchaseStatus
  ) {}

  public calculateTotal(): Money {
    return this.items.reduce(
      (total, item) => total.add(item.price.multiply(item.quantity)),
      Money.zero()
    );
  }

  public canBeRefunded(): boolean {
    return (
      this.status === PurchaseStatus.Completed &&
      this.completedAt.isAfter(DateTime.now().minus({ days: 30 }))
    );
  }
}
```

#### Services

Domain services handle complex business operations that don't naturally fit within a single model. They:

- Orchestrate operations between multiple domain models
- Implement business rules that span multiple aggregates
- Handle domain events and complex workflows

#### Interfaces

**Interfaces**, or **ports**, are the strategies that adapters at the infrastrcture level should implement, so that they can be consumed by the domain layer without causing implementation-specific friction.

> 💡Said differently, domain interfaces define **contracts** that infrastructure implementations must fulfill.

### Infrastructure Layer

Under infrastructure, we find external providers that serve purposes such as persistence, caching, messaging, mailing, among others. The idea is that both external services or particular libraries that wrap those services live here, in a way that makes them switchable.

> 💡 This is the main idea and benefit behind [hexagonal architecture](<https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)>).

The bottom line is that data models for external providers, as well as services that interface with them live here. But the use of [adapters](./abstract-interfaces.html) is also encouraged, so that external providers and wrapping libraries can be switched and updated as the application evolves, without the need to rewrite business logic.

Thus, the standard folder structure will look like this:

```
purchases
└───infrastructure
    └─── projects.infrastructure.module.ts
    └─── adapters
         └─── persistence
         |    └─── typeorm
         |    └─── prisma
         |    └─── mongodb
         └─── blockchain
         |    └─── evm
         |    └─── solana
         └─── queues
         |    └─── amqp
         └─── cache
              └─── redis
              └─── memory
```

> 👉 It should be noted that under normal circumstances, only one adapter will be used at a given time. But it's generally a good practice not to discard other adapters. Other situations may call for multiple adapters (i.e. writing to multiple blockchains).

---

## Summary

Whether you choose to work with the simple or complex approach, following these organization principles is more likely to produce code that is both easier to follow, and easier to maintain.

Of course, certain situations may require a slightly different setup - that's left to the reader's judgement.

But, as a general rule of thumb, it's good to stick to these principles. Also, this is very helpful if team members have to **move from one project to another**, as they will avoid the time-consuming task of trying to understand how the code is organized, and can instead focus on the particular business logic.
