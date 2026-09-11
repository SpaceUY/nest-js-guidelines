---
title: General good practices
parent: Testing
layout: default
nav_order: 2
---

1. [Unit Testing](#unit-testing)
   - [When to Use Unit Testing](#when-to-use-unit-testing)
   - [Good Practices](#good-practices)
2. [End-to-End Testing](#end-to-end-testing)
   - [When to Use End-to-End Testing](#when-to-use-end-to-end-testing)
   - [Good Practices](#good-practices-1)

# General good practices

## Use test factories

Create factories to generate test data and reduce code duplication.

```ts
// users/user.factory.ts
export class UserFactory {
  static create(override: Partial<CreateUserDto> = {}) {
    return {
      email: `test-${Date.now()}@example.com`,
      password: 'password123',
      ...override,
    };
  }
}
```

## Use test helpers

Create helpers to reduce code duplication and make tests more readable.

```ts
// users/user.helper.ts
export class UserTestHelper {
  static async createTestingModule() {
    const module: TestingModule = await Test.createTestingModule({
      imports: [UsersModule],
      providers: [PrismaService],
    }).compile();

    return {
      module,
      usersService: module.get<UsersService>(UsersService),
      prisma: module.get<PrismaService>(PrismaService),
    };
  }

  static async createTestUser(
    usersService: UsersService,
    override = {}
  ): Promise<User> {
    const userData = UserFactory.create(override);
    return await usersService.create(userData);
  }

  static async cleanupTestUsers(prisma: PrismaService) {
    await prisma.user.deleteMany({
      where: {
        email: { contains: 'test-' },
      },
    });
  }
}
```

## Use regression tests

When fixing a bug, always write a regression test to prevent it from reoccurring. Follow these steps:

1. Create a test that reproduces the bug

   - Start with the exact conditions that caused the bug
   - Make the test fail to verify it correctly detects the issue

2. Fix the bug and verify the test passes

3. Document the test clearly:

```ts
describe('UserService', () => {
  it('should not allow duplicate email addresses (bug #123)', () => {
    // Arrange: Setup the condition that caused the bug
    const existingUser = UserFactory.create({ email: 'test@example.com' });
    const duplicateUser = UserFactory.create({ email: 'test@example.com' });

    // Act & Assert: Verify the fix prevents the bug
    expect(async () => {
      await usersService.create(duplicateUser);
    }).rejects.toThrow('Email already exists');
  });
});
```

## Use meaningful test descriptions

Write clear and descriptive test names that explain the expected behavior. Follow the pattern "it should [expected behavior] when [condition]":

```ts
// ❌ Unclear test names
it('test1', () => {
  // ...
});

it('invoice calculation', () => {
  // ...
});

// ✅ Descriptive test names
describe('UserService', () => {
  it('should return null when user is not found', async () => {
    // test code
  });

  it('should throw UnauthorizedException when password is incorrect', async () => {
    // test code
  });
});
```

## Group related tests

Use describe blocks to group related tests and create a clear hierarchy:

```ts
describe('UserService', () => {
  describe('authentication', () => {
    describe('login', () => {
      it('should return JWT token when credentials are valid', () => {
        // test code
      });

      it('should throw UnauthorizedException when credentials are invalid', () => {
        // test code
      });
    });
  });
});
```

## Follow AAA pattern

Structure your tests using the Arrange-Act-Assert pattern to improve readability:

```ts
describe('UserService', () => {
  it('should update user profile successfully', async () => {
    // Arrange
    const user = await UserTestHelper.createTestUser(usersService);
    const updateData = { name: 'New Name' };

    // Act
    const result = await usersService.update(user.id, updateData);

    // Assert
    expect(result.name).toBe(updateData.name);
    expect(result.id).toBe(user.id);
  });
});
```

## Use beforeEach and afterEach hooks

Set up and clean up test data properly to ensure test isolation:

```ts
describe('UserService', () => {
  let usersService: UsersService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const { usersService: service, prisma: prismaService } =
      await UserTestHelper.createTestingModule();
    usersService = service;
    prisma = prismaService;
  });

  afterEach(async () => {
    await UserTestHelper.cleanupTestUsers(prisma);
  });

  // test cases
});
```
