---
title: Utilities
parent: Architecture
layout: default
nav_order: 3
---

# Utilities

While building applications, we often need to implement common patterns and utilities that help maintain code consistency and improve developer experience. This document outlines our recommended approaches for various utility patterns.

## The `common` folder

The common folder is a special location in our project structure where we place utilities, types, and other elements that are used across multiple modules. This helps maintain a single source of truth for shared functionality and reduces code duplication.

A typical common folder structure might look like this:

```
src
└───common
    └───constants
    └───enums
    └───types
    └───utils
    └───decorators
    └───filters
    └───guards
    └───pipes
    └───interfaces
```

Each subfolder serves a specific purpose:

- constants: Global constant values and configuration
- enums: Global enumeration types (as described below)
- types: Shared TypeScript types and type utilities
- utils: Helper functions and utility classes
- decorators: Custom decorators for controllers and methods
- filters: Exception filters and error handling
- guards: Authentication and authorization guards
- pipes: Global transformation and validation pipes
- interfaces: Shared interfaces used across modules

When deciding whether to place something in the common folder, consider:

- Is it used by multiple modules?
- Does it represent truly shared functionality?
- Would duplicating it across modules create maintenance issues?

If the answer to any of these questions is "yes", it probably belongs in common.

## Enums

Enums are an important part of any application. They are the natural (but labelled) extension of boolean values. They are typically used to describe possible _states_ of variables.

There are, in general, two types of enums to consider: module-specific enums, which should go in the respective module (as outlined [here](./2_project-structure.md)), and global enums, used throughout the project. These should live under `./common/enums`.

Enum files should use the `.enum.ts` extension.

### Enums as Plain Objects

Instead of using TypeScript enums, we recommend using Plain Old JavaScript Objects (POJOs) for enum-like structures. This approach provides several benefits:

- Allows for direct string literal usage without enum reference
- Provides better type inference and autocomplete
- Results in simpler JavaScript output
- Enables easier testing and serialization

> Typescript enums do weird things under the hood, while, POJOs are much easier to control.

```typescript
// ❌ Don't use TypeScript enums
enum PaymentStatus {
  PENDING = "PENDING",
  PROCESSING = "PROCESSING",
  COMPLETED = "COMPLETED",
  FAILED = "FAILED",
}

// ✅ Use const objects instead
export const PAYMENT_STATUSES = {
  PENDING: "PENDING",
  PROCESSING: "PROCESSING",
  COMPLETED: "COMPLETED",
  FAILED: "FAILED",
} as const;
```

> 💡 Our convention is to name enum POJOs with _uppercase snakecase_ (UPPERCASE*SNAKECASE), and use \_plurals*.

### Enum Types

By defining enums as POJOs with the `const` keyword, it's possible to draw literal union types as expected enum types. This is easy to do, through the usage of either of these two helper types:

```ts
/**
 * Get the value of a type that is an object
 */
export type ValueOf<T> = T[keyof T];

/**
 * Get the value of a type that is an object with nesting.
 */
export type NestedValueOf<T> = T extends object
  ? {
      [K in keyof T]: T[K] extends object ? NestedValueOf<T[K]> : T[K];
    }[keyof T]
  : never;
```

`ValueOf` simply takes all the values in a plain object, while `NestedValueOf` recursively goes through an object with nested keys, drawing out the literal types of values.

Typically, an enum definition will come coupled with its type, like:

```ts
export const PAYMENT_STATUSES = {
  PENDING: "PENDING",
  PROCESSING: "PROCESSING",
  COMPLETED: "COMPLETED",
  FAILED: "FAILED",
} as const;

export type PaymentStatus = ValueOf<typeof PAYMENT_STATUSES>;
```

> 🚨 Note that we use singlular for the type name, and that the `typeof` keyword is necessary.

## Types

> 🚧 Under construction!
