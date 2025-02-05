---
title: Types of testing
parent: Testing
layout: default
nav_order: 1
---

# Unit testing

Unit testing is a type of software testing where individual units or components of a software are tested in isolation. The purpose of unit testing is to verify that each unit of the software performs as expected.

## When to use unit testing

Unit testing is a good way to test complicated business logic and edge cases. For example, if you have a function that calculates the total of an invoice and applies different taxes and discounts, it's a good idea to test it with different inputs to ensure it works as expected.

This way we can refactor with confidence, knowing that the core logic is working as expected.

## Good practices

### Test edge cases

It's important to not only unit test the happy path, but also the edge cases. For example if you have the same function that calculates the total of an invoice, you should test it with an empty invoice, an invoice with only one item, an invoice with only one item and one discount, etc.

```ts
// ❌ Test without edge cases
it('should calculate the total of an invoice', () => {
  const invoice = new Invoice();
  invoice.addItem({ name: 'Item 1', price: 100 });
  invoice.addItem({ name: 'Item 2', price: 200 });
  invoice.addDiscount(10);

  expect(invoice.total).toBe(270);
});

// ✅ Test with edge cases
it('should calculate the total of an invoice', () => {
  const invoice = new Invoice();
  invoice.addItem({ name: 'Item 1', price: 100 });
  invoice.addItem({ name: 'Item 2', price: 200 });
  invoice.addDiscount(10);

  expect(invoice.total).toBe(270);
});

it('should calculate the total of an invoice with an empty invoice', () => {
  const invoice = new Invoice();
  expect(invoice.total).toBe(0);
});

it('should throw if discount is greater than 100', () => {
  const invoice = new Invoice();
  invoice.addItem({ name: 'Item 1', price: 100 });
  invoice.addDiscount(100);

  expect(() => invoice.total).toThrow();
});
```

### Keep tests focused and isolated

Each test should focus on testing one specific behavior or scenario. Avoid testing multiple things in a single test case. This makes tests easier to understand and maintain, and when they fail, it's clearer what went wrong.

```ts
// ❌ Testing multiple behaviors in one test
it('should handle invoice operations', () => {
  const invoice = new Invoice();
  invoice.addItem({ name: 'Item', price: 100 });
  expect(invoice.total).toBe(100);

  invoice.addDiscount(10);
  expect(invoice.total).toBe(90);

  invoice.clear();
  expect(invoice.total).toBe(0);
});

// ✅ Separate tests for each behavior
it('should calculate total after adding an item', () => {
  const invoice = new Invoice();
  invoice.addItem({ name: 'Item', price: 100 });
  expect(invoice.total).toBe(100);
});

it('should apply discount correctly', () => {
  const invoice = new Invoice();
  invoice.addItem({ name: 'Item', price: 100 });
  invoice.addDiscount(10);
  expect(invoice.total).toBe(90);
});

it('should clear invoice correctly', () => {
  const invoice = new Invoice();
  invoice.addItem({ name: 'Item', price: 100 });
  invoice.clear();
  expect(invoice.total).toBe(0);
});
```

### Keep mocks to a minimum

Mocks are useful tools for isolating the code under test, but excessive mocking can lead to brittle tests and missed bugs. Here are some guidelines for using mocks effectively:

- **Mock at boundaries**: Only mock external dependencies (HTTP calls, databases, third-party services)
- **Use real implementations** for business logic and domain objects when possible
- **Prefer stubs over mocks** when you only need to return specific values
- **Watch out for mock complexity** - if you need many mocks, consider it a code smell

```ts
// ❌ Over-mocking
it('should process order', () => {
  // Mocking everything makes tests fragile
  const product = mock(Product);
  const cart = mock(Cart);
  const price = mock(PriceCalculator);
  const order = new Order(product, cart, price);

  when(price.calculate()).thenReturn(100);
  // ... many more mock setups
});

// ✅ Better approach - mock only external dependencies
it('should process order', async () => {
  // Use real implementations for business logic
  const product = new Product('book', 50);
  const cart = new Cart([product]);

  // Mock only the payment gateway
  const paymentGateway = mock(PaymentGateway);
  when(paymentGateway.charge()).thenResolve({ success: true });

  const order = new Order(cart, paymentGateway);
  const result = await order.process();

  expect(result.success).toBe(true);
});
```

#### Signs you might be over-mocking:

- Your test setup is longer than the test itself
- You're mocking internal implementation details
- Tests break when refactoring, even though the behavior hasn't changed
- You need to mock chains of method calls

# End-to-end testing

End-to-end testing is a type of software testing where the software is tested as a whole, but in a controlled manner. The purpose of end-to-end testing is to verify that the software is working as expected when it is integrated with other software.

## When to use end-to-end testing

Every endpoint of the application should have at least one end-to-end test for the happy path. They are useful to know if all the endpoints are working as expected at least for the happy path.

## Good practices

### Focus on critical API endpoints

End-to-end tests should focus on testing complete API flows and critical endpoints. Use NestJS's built-in testing utilities to test your controllers and services together.

```ts
// ✅ Testing a critical API flow
describe('Order Flow (e2e)', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  it('should complete full order process', async () => {
    // Login
    const loginResponse = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'test@example.com', password: 'password' })
      .expect(200);

    authToken = loginResponse.body.token;

    // Create order
    const orderResponse = await request(app.getHttpServer())
      .post('/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        items: [{ productId: 1, quantity: 2 }],
      })
      .expect(201);

    // Verify order status
    await request(app.getHttpServer())
      .get(`/orders/${orderResponse.body.id}`)
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200)
      .expect((res) => {
        expect(res.body.status).toBe('created');
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

### Set up test environment properly

Each test suite should have proper setup and teardown logic using NestJS testing utilities. Use separate test databases and environment variables for testing.

```ts
// ✅ Proper test setup with database
describe('Users API (e2e)', () => {
  let app: INestApplication;
  let prisma: PrismaService;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    prisma = app.get(PrismaService);

    // Apply global pipes, interceptors, etc.
    app.useGlobalPipes(new ValidationPipe());
    await app.init();
  });

  beforeEach(async () => {
    // Custom logic to clean the database
    await cleanDatabase();
  });

  afterAll(async () => {
    await prisma.$disconnect();
    await app.close();
  });
});
```

### Test error scenarios

Test API error responses and edge cases thoroughly.

```ts
// ✅ Testing error scenarios
describe('Products API (e2e)', () => {
  it('should handle validation errors', () => {
    return request(app.getHttpServer())
      .post('/products')
      .set('Authorization', `Bearer ${adminToken}`)
      .send({
        // Missing required fields
        price: -100,
      })
      .expect(400)
      .expect((res) => {
        expect(res.body.message).toContain('name is required');
        expect(res.body.message).toContain('price must be positive');
      });
  });

  it('should handle not found errors', () => {
    return request(app.getHttpServer())
      .get('/products/999999')
      .expect(404)
      .expect((res) => {
        expect(res.body.message).toBe('Product not found');
      });
  });
});
```
