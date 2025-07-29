# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Build and Run
- **Build JAR**: `./gradlew clean bootJar`
- **Run for development**: `./gradlew devRun`
- **Build WAR**: `./gradlew :fineract-war:clean :fineract-war:war`
- **Build documentation**: `./gradlew doc`

### Database Setup
Before running the application, initialize the databases:
```bash
./gradlew createDB -PdbName=fineract_tenants
./gradlew createDB -PdbName=fineract_default
```

### Code Quality and Linting
- **Format code**: `./gradlew spotlessApply`
- **Check code style**: `./gradlew spotlessCheck`
- **Run Checkstyle**: `./gradlew checkstyleMain checkstyleTest`
- **Run SpotBugs**: `./gradlew spotbugsMain spotbugsTest`
- **Check licenses**: `./gradlew licenseMain licenseTest`
- **Run RAT (Release Audit Tool)**: `./gradlew rat`
- **Full check**: `./gradlew check`

### Testing
- **Run unit tests (fast)**: `./gradlew test -x :twofactor-tests:test -x :oauth2-tests:test -x :integration-tests:test`
- **Run all tests**: `./gradlew test`
- **Run integration tests**: `./gradlew :integration-tests:test`
- **Run E2E tests**: `./gradlew :fineract-e2e-tests-runner:test`
- **Run a single test**: `./gradlew :module-name:test --tests "TestClassName"`
- **Generate test coverage**: `./gradlew clean build jacocoTestReport`

### Docker
- **Build Docker image**: `./gradlew :fineract-provider:jibDockerBuild -x test`
- **Run with Docker Compose**: `docker compose -f docker-compose-development.yml up -d`

## High-Level Architecture

Apache Fineract is a modular microfinance platform built with Spring Boot. The architecture follows these key principles:

### Module Structure
The codebase is organized into several key modules:

- **fineract-provider**: Main application module containing the Spring Boot application and REST API endpoints
- **fineract-core**: Core domain models and business logic shared across modules
- **fineract-client**: Auto-generated client library for API consumers
- **fineract-loan**: Loan-specific functionality and domain logic
- **fineract-savings**: Savings account functionality
- **fineract-accounting**: Accounting and GL functionality
- **fineract-charge**: Fee and charge management
- **fineract-command**: Command pattern implementation for API operations
- **fineract-investor**: External investor integration features
- **fineract-progressive-loan**: Advanced loan product features
- **integration-tests**: Integration test suite
- **custom/**: Directory for custom implementations and extensions

### Key Architectural Patterns

1. **Multi-tenancy**: Fineract supports multiple tenants with separate databases. Tenant resolution happens through HTTP headers.

2. **Command Pattern**: All write operations use a command pattern implementation. Commands are validated, processed, and audited through a centralized pipeline.

3. **Event-Driven**: Business events can be published to external systems via Kafka or ActiveMQ for integration purposes.

4. **Batch Processing**: Uses Spring Batch for COB (Close of Business) and other scheduled jobs. Supports remote partitioning for scalability.

5. **API-First**: RESTful API with OpenAPI/Swagger documentation auto-generated from code annotations.

6. **Static Weaving**: JPA entities use EclipseLink with static weaving for performance optimization.

### Database Architecture

- Supports MariaDB, MySQL, and PostgreSQL
- Each tenant has its own database
- Uses Liquibase for database migrations
- Connection pooling via HikariCP
- Timezone handling: All date/time stored in UTC

### Security
- Basic Auth and OAuth2 support
- Optional 2FA authentication
- Role-based access control (RBAC)
- API permissions are granular and configurable

### Important Development Notes

1. **Java Version**: Requires Java 21 (Azul Zulu JVM tested in CI)
2. **Build System**: Gradle with custom plugins for various tasks
3. **Code Style**: Enforced via Spotless and Checkstyle - always run `./gradlew spotlessApply` before committing
4. **Error Handling**: Uses Spring's exception handling with custom error responses
5. **Logging**: SLF4J with Logback, structured logging supported
6. **Testing**: JUnit 5 with Mockito, Cucumber for BDD tests