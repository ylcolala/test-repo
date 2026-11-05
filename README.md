# Java Spring Boot Development Standards

## Technology

- Java version follows the existing project.
- Spring Boot follows the existing project version.
- Use Maven or Gradle according to the project.
- JUnit 5 and Mockito for testing.
- Testcontainers for integration testing where appropriate.

## Architecture

Use layered architecture:

- controller
- application
- domain
- infrastructure
- configuration

Keep business logic independent from infrastructure where practical.

## Java Standards

- Constructor injection.
- SOLID principles.
- Immutable objects where appropriate.
- Meaningful names.
- Avoid duplicated code.
- Avoid unnecessary abstractions.

## Spring Standards

- Use @Service for application services.
- Use @Repository appropriately.
- Use @RestController for REST APIs.
- Use @ConfigurationProperties for configuration.
- Use centralized exception handling.
- Use Bean Validation.

## Database

- Never expose JPA entities directly through REST APIs.
- Avoid N+1 queries.
- Define transaction boundaries explicitly.
- Use database migrations where appropriate.

## Financial Business Logic

- Use BigDecimal for financial calculations.
- Define currency and scale explicitly.
- Define rounding rules explicitly.
- Maintain calculation auditability.

## Testing

- Test normal and exceptional scenarios.
- Test boundary conditions.
- Test transaction behavior.
- Test duplicate message processing.
- Test idempotency.
- Prefer realistic integration tests.

## Delivery

Before completing a task:

- Compile the project.
- Run relevant unit tests.
- Run integration tests where applicable.
- Review Git diff.
- Report known limitations.
