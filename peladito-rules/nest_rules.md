# Nest.js Development Standards & Best Practices

## Code Quality & Maintainability

- **Type Safety**

  - Use TypeScript across your entire application.
  - Leverage Nest's decorators and TypeScript features for compile-time checks.
  - Avoid `any` type (enable `tsconfig.json` `noImplicitAny`)
  - If an edge case absolutely requires using `any`, you must add the following comment on the line immediately above the usage:

    ```typescript
    // ai-disable-next-line no-use-any
    ```

    This signals that the use of `any` is intentional and should be treated as an exception.

    **Example**:

    ```typescript
    // ai-disable-next-line no-use-any
    function riskyParse(input: any) {
      // ...
    }
    ```

- **Security**

  - Never hardcode sensitive keys/credentials
  - Use environment variables (`.env` files) with proper `.gitignore`

- **Descriptive Naming & File Organization**

  - Use clear, descriptive variable, method, and class names (e.g., `isActive`, `userEmail`, `handleUserLogin`).
  - DTO classes must end with Dto (e.g., `CreateUserDto`) and use camelCase for their properties.

- **Modular Structure**

  - Each module must have its own folder, containing `controller`, `service`, and relevant DTOs.
  - Alternatively, shared DTOs can live in a global `common/dto folder`.

- **DTO Rules**

  - DTO Suffix: Each Data Transfer Object ends with Dto (e.g., `CreateUserDto`).
  - Validation: Implement `class-validator` and/or `class-transformer` for every DTO.
  - Example:

  ```typescript
  import { IsString, IsEmail } from "class-validator";

  export class CreateUserDto {
    @IsString()
    username: string;

    @IsEmail()
    email: string;

    @IsString()
    @Exclude() 
    password: string;
  }
  ```

## Project Structure & SOLID Principles

- **Single Responsibility Principle (SRP)**

  - A module encapsulates one domain or feature.
  - A service handles business logic and database interactions for that feature.
  - A controller should focus on request/response handling and simple validations, delegating complex logic to services.

- **Open/Closed Principle (OCP)**

  - Components (modules, services) should be open for extension but closed for modification.
  - Example: Instead of hardcoding logic, inject dependencies or use configuration to extend features without modifying existing code extensively.

- **DRY (Don't Repeat Yourself)**

  - Reusable logic, such as database interactions or utility functions, should live in dedicated services, helpers, or pipes.
  - Extract repeated code into custom decorators or shared utility functions when appropriate.

- **Avoid Common Anti-Patterns**

  - Fat Controllers: Controllers with too much logic. Offload to services or dedicated helper classes.
  - God Services: Services that do everything for multiple modules. Instead, split logic by domain.
  - Direct DB Calls in Controller: Always use a service for database interactions.
  - Magic Numbers/Strings: Avoid hardcoding values directly in your code (e.g., `statusCode = 200`, `'my-secret-key'`, `TIMEOUT = 3000`).
    - Correct Approach: Place such constants in a config file, use environment variables, or define them in an enum or constants module. For instance:
    ```typescript
    // constants.ts
    export const DEFAULT_TIMEOUT = 3000;
    export const JWT_SECRET = process.env.JWT_SECRET || "fallback-secret";
    import {  
      HttpStatus,
    } from '@nestjs/common';
    ```

## Security & Environment Management

- **Sensitive Data**

  - Never hardcode secrets (API keys, DB credentials, tokens) directly in code.
  - Store secrets in environment variables, using `.env` files, and ensure they're `.gitignored`.
  - A good approach for using this technique in Nest is to create a ConfigModule that exposes a ConfigService which loads the appropriate `.env` file. While you may choose to write such a module yourself, for convenience Nest provides the @nestjs/config package out-of-the box
  - Encrypt Sensitive DB Data: Any sensitive information (e.g., passwords, tokens) stored in the database must be hashed or encrypted. For instance, always store passwords using a secure hashing algorithm (e.g., bcrypt, Argon2), never as plain text.

- **Validation**

  - Use a global validation pipe (`ValidationPipe`) or method-level pipes to automatically validate incoming requests against DTOs.
  - Additional business logic checks can be placed either in the controller or in services that the controller calls.

- **Dependency Injection**

  - Always inject services or repositories through Nest's DI system. Avoid manual instantiation (e.g., `new SomeService()`).

## Logging & Monitoring

- **Logging**

  - Use nestjs-pino for structured, performant logs.
  - Configure log levels (e.g., debug, info, error) depending on environment (`development` vs. `production`).
  - Log critical information, errors, and relevant request data. Avoid logging sensitive user data or tokens.

- **Monitoring & Metrics**

  - Integrate @willsoto/nestjs-prometheus for Prometheus metrics.
  - Expose application metrics (e.g., request count, error count, response time) to ensure observability and easy monitoring.

## Database Access (Prisma)

- **Service-Driven Queries**

  - Interact with Prisma only from services, never directly from controllers.
  - If a controller needs data validation or preprocessing, perform it using dedicated services or other modules before/after hitting the database.

- **Repository/Service Pattern**

  - For more complex logic, consider a repository pattern or a dedicated Prisma service to abstract DB calls.
  - Keep queries and domain logic well-separated to enhance maintainability.

## Controller & Swagger Documentation

- **Controller Methods**
  - Each route handler must have clear documentation using either individual decorators or custom combined decorators:
    
    Option 1 - Individual decorators:
    ```typescript
    @Post()
    @ApiOperation({ summary: 'Create a user' })
    @ApiResponse({
      status: HttpStatus.CREATED,
      description: 'User created successfully'
    })
    create(@Body() createUserDto: CreateUserDto) {
      return this.usersService.create(createUserDto);
    }
    ```

    Option 2 - Custom decorator (Recommended for better DX):
    ```typescript
    @Post()
    @DocumentedEndpoint({
      summary: 'Create a user',
      status: HttpStatus.CREATED,
      description: 'User created successfully'
    })
    create(@Body() createUserDto: CreateUserDto) {
      return this.usersService.create(createUserDto);
    }
    ```

    Example implementation of the custom decorator:
    ```typescript
    export interface EndpointDoc {
      summary: string;
      status: HttpStatus;
      description: string;
    }

    export const DocumentedEndpoint = (options: EndpointDoc): MethodDecorator => {
      return (
        target: any,
        propertyKey: string | symbol,
        descriptor: PropertyDescriptor,
      ) => {
        ApiOperation({ summary: options.summary })(target, propertyKey, descriptor);
        ApiResponse({
          status: options.status,
          description: options.description,
        })(target, propertyKey, descriptor);
        
        return descriptor;
      };
    };
    ```

- **Swagger Decorators**
  - Use @ApiTags at the controller level to group endpoints in Swagger
  - Utilize @ApiParam, @ApiQuery, and @ApiBody where applicable to document request parameters, queries, and bodies
  - Provide @ApiResponse annotations to describe possible response status codes and outcomes

## Testing & Quality Assurance

- **Unit Tests**

  - Use Jest for unit tests.
  - Target utilities. Ensure critical business logic is covered.

- **E2E Tests**
  - Leverage SuperTest (via Nest's E2E testing setup) to test the actual HTTP endpoints.
  - Include common scenarios, edge cases, and error states.

## Additional Considerations

- **Error Handling**

  - Use custom exceptions or Nest's built-in HTTP exceptions (`HttpException`, `NotFoundException`, etc.) to provide meaningful error messages and correct status codes.
  - Consider implementing a global or scoped `HttpExceptionFilter` (or other Exception Filters) to centralize error handling and return consistent responses.

- **Default Healthcheck Controller**
  - Provide a default endpoint (e.g., GET /) that returns a simple health/status response.
  - This allows quick verification from browsers or load balancers that the service is up and running.
    ```typescript
    @Controller()
    export class AppController {
      @Get()
      healthCheck() {
        return { status: "ok" };
      }
    }
    ```
