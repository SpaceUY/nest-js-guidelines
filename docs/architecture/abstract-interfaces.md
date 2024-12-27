---
title: Abstract providers & interfaces
parent: Parent
layout: default
nav_order: 1
---

# Abstract providers

When implementing some types of services, a contradiction of sorts happens. Said service may have very clearly defined responsibilities, but when it comes to implementing a solution, we tightly couple it with a specific implementation.

To better understand the above statement, picture an _mailing service_. All we want to do is send emails, so we can imagine that we need a method like `sendEmail`, and that's it. We don't really care _how_ the service does it, as long as the job is done.

Suppose we were using some external provider `A` for sending emails. Now, we want to try a different provider `B`, and we go and create some custom implementation for it. But we decide to name the method `sendOneEmail` instead, and make it have a different signature than `sendEmail`. How does this impact our code?

The answer is that we need to _refactor_ to accomodate to this new signature. We left the implementation _interface_ for the developer to decide upon, and this leads to inconsistencies. Can we do better?

## Using Interfaces

What we can do is _define an interface_ in advance, and require any implementation to conform to it. Custom implementations are called _adapters_ or _strategies_ in this context.

To do so, we can use _[abstract classes](https://www.tutorialsteacher.com/typescript/abstract-class)_ instead of pure typescript interfaces. This has the advantage of allowing some methods to be predefined or have a default implementation in the abstract class.

For example, for a mailing service, we may do something like this:

```ts
export interface SendEmailParams<T extends Template> {
  template: {
    name: LocalizedTemplate<T>;
    params: TemplateParams[T];
  };
  to: string;
  from?: string;
  subject?: string;
}

@Injectable()
export abstract class MailingProviderService {
  abstract sendEmail<T extends Template, R>(
    params: SendEmailParams<T>
  ): Promise<R>;
}
```

Abstract classes are meant to be extended - they should not be directly instantiated. A _strategy_ or _adapter_ is then created by extending this class, and providing the particular implementation of `sendEmail` for the specific provider:

```ts
@Injectable()
export class SendgridAdapter extends MailingProviderService {
  sendEmail<T extends Template, SendgridResponse>(
    params: SendEmailParams<T>
  ): Promise<SendgridResponse> {
    // ...
    // Actual implementation.
    // ...
  }
}
```

## Interfaces as injection tokens

So we have our custom adapter, and we want another service to consume it. We could set this adapter as a provider, and inject it into services normally:

```ts
export class SomeService {
  constructor(private readonly mailing: SendgridAdapter) {}

  // ...
}
```

This has a drawback: if we ever need to switch the adapter for _another implementation_ (for instance, AWS SES), we'd have to change all the injection tokens across our application.

To avoid this, we can again use the abstract interface, this time as an _injection token_. The trick is to create the provider with the interface as the token, and the custom implementation as the value, by using the `useClass` directive:

```ts
@Module({
  imports: [
    /* ... */
  ],
  providers: [
    // ...
    {
      provide: MailingProviderService,
      useClass: SendgridAdapter,
    },
  ],
  exports: [
    // ...
    MailingProviderService,
  ],
})
export class MailingModule {}
```

By doing this, we can use `MailingProviderService` as our injection token:

```ts
export class SomeService {
  constructor(private readonly mailing: MailingProviderService) {}
  // ...
}
```

And the underlying implementation will be determined by the module configuration.

## Advanced patterns

Some scenarios require a more specific setup. In particular, imagine these two situations:

1. You want to support multiple providers (adapters) for the same interface at once.
2. You want to switch providers on a per-module basis (suppose you're doing progressive changes).

Let's analyze how these two cases could be implemented.

### 1. Multiple simultaneous providers

This is as easy as using _custom injection tokens_. Instead of relying solely on the abstract class as our token, we could introduce new ones:

```ts
export const AWS_SES_TOKEN = 'AWS_SES_TOKEN'

@Module({
    // ...
    providers: [
        {
            provide: MailingProviderService,
            useClass: SendgridAdapter,
        },
        {
            provide: AWS_SES_TOKEN,
            useClass: AwsSesAdapter,
        }
    ],
})
```

> 💡 TIP: By keeping the abstract class as an injection token, you give sort of a "default provider" behavior.

The only caveat is that injecting the custom token requires explicit declaration with `@Inject()`:

```ts
export class SomeService {
  constructor(
    @Inject(AWS_SES_TOKEN) private readonly mailing: MailingProviderService
  ) {}
  // ...
}
```

Maintaining this code suffers from the same problem as before (we need to keep track of where we use `AWS_SES_TOKEN`, in this case), so it's generally discouraged. The second approach is recommended.

### 2. Dynamic modules

An alternative to hard coupling a specific adapter to the abstract interface is to _make it parametric_. [Dynamic modules](https://docs.nestjs.com/fundamentals/dynamic-modules) are a feature in NestJS that allows modules to receive arguments at creation time, and modify their structure based on the provided parameters.

The general structure is:

```ts
@Module({})
export class MailingModule {
  static register(params?: ApplicationParams) {
    const { mailingProvider = DEFAULT_MAILING_PROVIDER } =
      params.providers ?? {};

    const mailingModule = MailingModule.selectMailingProvider_(mailingProvider);

    return {
      module: MailingModule,
      imports: [mailingModule],
      exports: [mailingModule],
    };
  }

  private static selectMailingProvider_(mailingProvider: MailingProvider) {
    switch (mailingProvider) {
      case MAILING_PROVIDERS.SENDGRID:
        return SendgridModule;
      case MAILING_PROVIDERS.SES:
        return AwsSesModule;
      default:
        throw new Error(`Mailing provider "${mailingProvider}" not supported`);
    }
  }
}
```

Notice how we choose between _modules_ and not single providers. This allows each adapter implementation to use custom services and providers if needed. But a plain adapter module implementation would look like this:

```ts
@Module({
  providers: [
    {
      provide: MailingProviderService,
      useClass: SendgridAdapter,
    },
  ],
  exports: [MailingProviderService],
})
export class SendgridModule {}
```

All we need to do is bind the abstract class to a custom implementation! The module structure allows for some extra customization if needed.

When instantiating `MailingModule`, we need to do it using `MailingModule.register`.

> 💡 TIP: The `register` method name is part of the [Community Guidelines](https://docs.nestjs.com/fundamentals/dynamic-modules#community-guidelines) when it comes to static method naming for Dynamic Modules. But you can come up with your own naming!

## Usefulness

In general, abstract interfaces will be useful when we have:

- external providers that we may want to switch in the future without compromising current implementation of provider consumers.
- libraries that we may want to switch, and may have breaking changes (such as ORMs).
- parts of our application that use code that may be changed / versioned in the future.

These points suggest that this pattern is useful for applications that _may evolve over time_. Not all projects belong to this category.

But, there's something else hidden in there: by defining these abstract interfaces, we essentially build a _standard_ for development. And this is especially useful for our style of work: developers moving between projects can expect these interfaces to be consistent, and not have to re-learn implementation details!
