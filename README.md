# StockMarket

Production-grade financial portfolio management system demonstrating **Domain-Driven Design (DDD)** and **Hexagonal Architecture** in a realistic, non-trivial domain.

[![Java](https://img.shields.io/badge/Java-21-orange)](https://openjdk.org/projects/jdk/21/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-brightgreen)](https://spring.io/projects/spring-boot)
[![Maven](https://img.shields.io/badge/Build-Maven-blue)](https://maven.apache.org/)

## Table of Contents

- [Overview](#overview)
- [Business Problem](#business-problem)
- [Features](#features)
- [Architecture](#architecture)
- [Technologies](#technologies)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running Locally](#running-locally)
- [API Overview](#api-overview)
- [Database](#database)
- [Testing](#testing)
- [Build](#build)
- [Development Workflow](#development-workflow)
- [Future Improvements](#future-improvements)

## Overview

StockMarket is a production-grade, instructional financial-portfolio management system built to demonstrate Domain-Driven Design (DDD) and Hexagonal Architecture in a realistic, non-trivial domain. Developed as a reference implementation for software engineering students, workshop participants, and professional architects, it provides a complete, testable system with sophisticated business rules, multiple interchangeable infrastructure adapters, and explicit architectural boundaries enforced at both the module level and through automated fitness tests.

The project simulates a stock portfolio application where users can create portfolios, deposit and withdraw cash, buy and sell shares, view transaction history, and analyze holding performance. What makes StockMarket architecturally significant is not the features themselves, but how they are built: the financial domain surfaces genuine modelling tensions — FIFO lot accounting, settlement mechanics, concurrent mutation of shared portfolio state under transactional guarantees, and aggregate boundary enforcement — that are difficult to contrive artificially. Every design decision is grounded in concrete business semantics rather than textbook abstractions.

The repository is accompanied by a comprehensive technical book that traces the architectural and domain reasoning behind StockMarket, from strategic design decisions and aggregate boundaries to persistence trade-offs and testing strategies.

## Business Problem

Financial portfolio systems must maintain strict integrity under concurrent access while enforcing complex business rules:

| Challenge | Description |
|---|---|
| **FIFO Lot Accounting** | When shares are sold, the system must consume shares from the oldest purchase lots first (First-In-First-Out), accurately tracking cost basis, proceeds, and realized profit/loss for tax and reporting purposes. |
| **Concurrent Portfolio Mutation** | Multiple users or processes may attempt to trade from the same portfolio simultaneously. The system must prevent race conditions that could lead to incorrect balances, overselling, or double-spending. |
| **Aggregate Integrity** | A `Portfolio` aggregate root must enforce invariants such as "cannot sell more shares than owned," "cannot withdraw more cash than available," and "cannot purchase with insufficient funds." |
| **Technology Flexibility** | The core domain and application logic must remain completely agnostic to persistence technology (relational vs. document) and external market data providers, allowing infrastructure to be swapped without touching business logic. |
| **Auditability** | Every financial operation must leave an immutable transaction record for compliance, reporting, and performance analysis. |

StockMarket solves these problems by combining a rich domain model with hexagonal architecture, ensuring that business rules live in the domain layer while infrastructure concerns are pushed to the edges through explicit ports and adapters.

## Features

| Feature | Description |
|---|---|
| **Portfolio Lifecycle** | Create portfolios, retrieve by ID, and list all portfolios with owner names and cash balances. |
| **Cash Management** | Deposit and withdraw funds with strict validation (positive amounts only, no overdrafts). |
| **Stock Purchases** | Buy shares at live market prices. The system verifies sufficient funds, creates purchase lots, and records transactions. |
| **Stock Sales (FIFO)** | Sell shares with automatic FIFO lot consumption. Calculates proceeds, cost basis, and realized profit/loss. |
| **Transaction History** | Retrieve complete audit trails of deposits, withdrawals, purchases, and sales, optionally filtered by type. |
| **Holdings Performance** | Real-time reporting of unrealized gains, realized gains, average purchase price, and current market price per ticker. |
| **Live Market Prices** | Integration with Finnhub and Alpha Vantage APIs, plus a mock provider for testing without API keys. |
| **Dual Persistence** | Choose between JPA/MySQL (pessimistic locking) or MongoDB (optimistic locking with automatic retry) at runtime via Spring profiles. |
| **Concurrent Safety** | Pessimistic write locks (JPA) or optimistic versioning with AOP retry (MongoDB) protect against concurrent portfolio updates. |
| **Comprehensive Testing** | Domain unit tests, application service unit tests, adapter contract tests, REST integration tests, architecture fitness tests, and concurrent-write stress tests. |

## Architecture

StockMarket follows **Hexagonal Architecture (Ports & Adapters)** combined with **Domain-Driven Design** principles. The codebase is organized as a Maven multi-module project where each module corresponds to an architectural layer, and the dependency rule — adapters depend on the application core, never the reverse — is enforced at build time.

### Architectural Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    Driving Adapters                          │
│  ┌─────────────────┐  ┌─────────────────────────────────┐  │
│  │  REST API       │  │  Future: CLI, Messaging, etc.   │  │
│  │  (Spring MVC)   │  │                                 │  │
│  └────────┬────────┘  └─────────────────────────────────┘  │
│           │                                                  │
│           ▼  Primary Ports (Use Cases)                       │
├─────────────────────────────────────────────────────────────┤
│                    Application Layer                         │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Application Services (orchestration, no business rules)│ │
│  │  • CashManagementService                                │ │
│  │  • PortfolioLifecycleService                            │ │
│  │  • PortfolioStockOperationsService                      │ │
│  │  • ReportingService                                     │ │
│  │  • TransactionService                                   │ │
│  │  • GetStockPriceService                                 │ │
│  └─────────────────────────────────────────────────────────┘ │
│           │                                                  │
│           ▼  Secondary Ports (Repositories, Providers)       │
├─────────────────────────────────────────────────────────────┤
│                    Domain Layer                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Rich Domain Model (entities, value objects, invariants)│ │
│  │  • Portfolio (Aggregate Root)                           │ │
│  │  • Holding (Entity)                                     │ │
│  │  • Lot (Entity)                                         │ │
│  │  • Money, Price, ShareQuantity, Ticker (Value Objects)  │ │
│  │  • Transaction hierarchy (Deposit, Withdrawal, etc.)    │ │
│  │  • HoldingPerformanceCalculator (Domain Service)        │ │
│  └─────────────────────────────────────────────────────────┘ │
│           ▲                                                  │
│           │  Secondary Ports                                 │
├─────────────────────────────────────────────────────────────┤
│                    Driven Adapters                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  JPA/MySQL   │  │  MongoDB     │  │  Market Data     │  │
│  │  Persistence │  │  Persistence │  │  (Finnhub, etc.) │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Key Architectural Decisions

| Decision | Rationale |
|---|---|
| **Rich Domain Model** | Business rules (e.g., "cannot sell more than owned," FIFO accounting) are enforced inside the `Portfolio` aggregate, not in application services or controllers. This prevents "anemic domain model" anti-patterns and ensures invariants are maintained regardless of which adapter drives the system. |
| **Interface Segregation** | Use cases are split into fine-grained ports (`CashManagementUseCase`, `PortfolioLifecycleUseCase`, `PortfolioStockOperationsUseCase`, etc.) so that clients depend only on the methods they need. |
| **Anti-Corruption Layers** | Mappers in each persistence adapter translate between domain objects and infrastructure-specific entities/documents, preventing persistence concerns from leaking into the domain. |
| **CQRS-Ready Structure** | Although not fully separated into distinct read/write models, the reporting use case demonstrates how read-side concerns (holdings performance) can be optimized independently from write-side commands. |
| **Framework Isolation** | The domain module has zero framework dependencies. The application module depends only on `jakarta.transaction-api`. Spring appears only in adapter and bootstrap modules. |

## Technologies

| Category | Technology | Purpose |
|---|---|---|
| Language | Java 21 | Modern language features (records, pattern matching, virtual threads ready) |
| Framework | Spring Boot 3.x | Application bootstrap, dependency injection, web layer, caching |
| Build Tool | Apache Maven | Multi-module build, dependency management, test-jar sharing |
| Persistence (Option A) | Spring Data JPA + Hibernate + MySQL 8 | Relational storage with pessimistic locking (`SELECT ... FOR UPDATE`) |
| Persistence (Option B) | Spring Data MongoDB 7.x | Document storage with optimistic locking (`@Version`) and multi-document transactions |
| Testing | JUnit 5, AssertJ, Mockito, RestAssured, WireMock, Testcontainers, ArchUnit | Unit, integration, contract, and architecture fitness testing |
| Caching | Caffeine + Spring Cache | Stock price caching to respect API rate limits |
| Retry | Spring Retry + AOP AspectJ | Automatic retry of concurrent write conflicts on MongoDB |
| API Documentation | SpringDoc OpenAPI | Auto-generated Swagger UI at `/swagger-ui.html` |
| CI/CD | GitHub Actions | Automated build, test, and SonarCloud analysis |
| Quality | SonarCloud, JaCoCo | Code coverage and quality gate enforcement |
| Containers | Docker + Docker Compose | Local MySQL and MongoDB instances |
| External APIs | Finnhub, Alpha Vantage | Real-time stock price quotes |

## Project Structure

StockMarket is a Maven multi-module project. The module boundaries physically enforce the hexagonal dependency rule.

```
StockMarket/
├── pom.xml                                    # Parent POM: dependency management, versions
├── domain/                                    # Pure domain model — NO framework dependencies
│   └── src/main/java/.../model/
│       ├── market/                            # Ticker, StockPrice
│       ├── money/                             # Money, Price, ShareQuantity
│       ├── portfolio/                         # Portfolio, Holding, Lot, aggregates, invariants
│       └── transaction/                       # Transaction hierarchy (sealed types)
├── application/                               # Application services and ports
│   └── src/main/java/.../application/
│       ├── port/in/                           # Primary ports (use case interfaces)
│       ├── port/out/                          # Secondary ports (repository interfaces)
│       ├── service/                           # Application service implementations
│       └── exception/                         # Application-layer exceptions
├── adapters-inbound-rest/                     # Driving adapter: REST API
│   └── src/main/java/.../adapter/in/
│       ├── PortfolioRestController.java
│       ├── StockRestController.java
│       ├── ExceptionHandlingAdvice.java
│       └── webmodel/                          # DTOs (records)
├── adapters-outbound-persistence-jpa/         # Driven adapter: JPA/MySQL
│   └── src/main/java/.../adapter/out/persistence/jpa/
│       ├── entity/                            # JPA entities
│       ├── mapper/                            # Domain ↔ JPA mappers
│       ├── repository/                        # Port implementations
│       └── springdatarepository/              # Spring Data interfaces
├── adapters-outbound-persistence-mongodb/     # Driven adapter: MongoDB
│   └── src/main/java/.../adapter/out/persistence/mongodb/
│       ├── document/                          # MongoDB documents
│       ├── mapper/                            # Domain ↔ Document mappers
│       ├── repository/                        # Port implementations
│       └── springdatarepository/              # Spring Data interfaces
├── adapters-outbound-market/                  # Driven adapter: Market data APIs
│   └── src/main/java/.../adapter/out/rest/
│       ├── FinhubStockPriceAdapter.java
│       ├── AlphaVantageStockPriceAdapter.java
│       └── MockFinhubStockPriceAdapter.java
└── bootstrap/                                 # Composition root
    └── src/main/java/.../stockmarket/
        ├── StockMarketApplication.java         # @SpringBootApplication
        ├── config/SpringAppConfig.java         # Bean wiring (manual, no component scan for services)
        └── config/RetryOnWriteConflictAspect.java  # AOP retry infrastructure
```

### Dependency Rules (Enforced by Maven)

| Module | Dependencies |
|---|---|
| `domain` | Depends on nothing (pure Java) |
| `application` | Depends on `domain` |
| `adapters-inbound-rest` | Depends on `application` |
| `adapters-outbound-persistence-jpa` | Depends on `application` |
| `adapters-outbound-persistence-mongodb` | Depends on `application` |
| `adapters-outbound-market` | Depends on `application` |
| `bootstrap` | Depends on all modules; contains the `main()` method and Spring configuration |

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| JDK | 21 or higher | Temurin distribution recommended |
| Apache Maven | 3.6+ | Or use the included `./mvnw` wrapper |
| Docker | 20.10+ | Required for Testcontainers and local databases |
| Docker Compose | 2.x+ | For running local MySQL + MongoDB |
| Git | Any | For cloning the repository |
| IntelliJ IDEA | Ultimate (recommended) | For HTTP client testing with `doc/calls.http` |

> **Note on Docker:** Testcontainers 1.21.4 handles Docker Engine 27.x and 29.x out of the box. If you encounter "Could not find a valid Docker environment," verify `docker version` reports API version ≥ 1.44.

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/AhmedDhibi1/StockMarket.git
cd StockMarket
```

### 2. Verify the Build (No External Setup Required)

The entire test suite runs out of the box without API keys or manual database setup, thanks to Testcontainers:

```bash
./mvnw clean test
```

This command will:

- Compile all modules
- Run domain unit tests
- Run application service unit tests
- Start temporary MySQL and MongoDB containers automatically
- Run adapter contract tests against real databases
- Run REST integration tests
- Execute architecture fitness tests (ArchUnit)

## Configuration

StockMarket uses Spring profiles to select persistence technology and market data provider. You must always activate one persistence profile and one market profile.

### Available Profiles

| Category | Profile | Description |
|---|---|---|
| Persistence | `jpa` | Spring Data JPA over MySQL with pessimistic row-level locking |
| Persistence | `mongodb` | Spring Data MongoDB with optimistic locking and AOP retry |
| Market Data | `finhub` | Real-time quotes from Finnhub |
| Market Data | `alphaVantage` | Real-time quotes from Alpha Vantage |
| Market Data | `mockfinhub` | Random-but-reasonable prices — no API key needed (used by tests) |

### Valid Combinations

```
jpa,finhub
jpa,alphaVantage
jpa,mockfinhub
mongodb,finhub
mongodb,alphaVantage
mongodb,mockfinhub
```

### Profile-Specific Configuration

- `application-jpa.properties`: Disables MongoDB auto-configuration to prevent connection-refused logs
- `application-mongodb.properties`: Configures MongoDB URI and disables JPA auto-configuration
- `application-test.properties`: Uses Testcontainers JDBC URL `jdbc:tc:mysql:8.0.32:///testdb` for integration tests

### Environment Variables

API keys and database credentials are read from environment variables so that real secrets are never committed to version control.

| Variable | Purpose | Required For |
|---|---|---|
| `FINNHUB_API_KEY` | Finnhub API authentication | `finhub` profile |
| `ALPHA_VANTAGE_API_KEY` | Alpha Vantage API authentication | `alphaVantage` profile |
| `SPRING_DATA_MONGODB_URI` | MongoDB connection string | `mongodb` profile |
| `SPRING_DATASOURCE_USERNAME` | MySQL username | `jpa` profile (defaults to `root`) |
| `SPRING_DATASOURCE_PASSWORD` | MySQL password | `jpa` profile (defaults to `rootpwd`) |

### Quick Setup with `.env`

Copy the example file and fill in your keys:

```bash
cp .env.example .env
# Edit .env with your real API keys
source .env
```

The `.env` file is listed in `.gitignore` and will never be committed.

Free-tier API keys are sufficient for local development:

- Finnhub: https://finnhub.io/
- Alpha Vantage: https://www.alphavantage.co/support/#api-key

## Running Locally

### Step 1: Start Local Databases

```bash
docker-compose up -d
```

This starts:

- MySQL 8.0.32 on port `3307` (mapped to container port `3306`)
- MongoDB 7.0 on port `27017` with a single-node replica set (`rs0`) for transaction support

Wait for the health checks to pass:

```bash
docker-compose ps
```

### Step 2: Install Modules

Because this is a multi-module project, sibling modules must be installed into your local Maven repository before running the bootstrap module:

```bash
./mvnw install -DskipTests -q
```

### Step 3: Run the Application

**Option A: JPA + Mock Provider** (no API keys needed)

```bash
./mvnw spring-boot:run -pl bootstrap -Dspring-boot.run.profiles=jpa,mockfinhub
```

**Option B: JPA + Finnhub** (requires `FINNHUB_API_KEY`)

```bash
source .env
./mvnw spring-boot:run -pl bootstrap -Dspring-boot.run.profiles=jpa,finhub
```

**Option C: MongoDB + Alpha Vantage**

```bash
source .env
./mvnw spring-boot:run -pl bootstrap -Dspring-boot.run.profiles=mongodb,alphaVantage
```

The application starts on `http://localhost:8081`.

### Step 4: Explore the API

| Resource | URL |
|---|---|
| Swagger UI | http://localhost:8081/swagger-ui.html |
| OpenAPI Docs | http://localhost:8081/api-docs |

Pre-configured HTTP requests are available in `doc/calls.http` for IntelliJ IDEA Ultimate users.

### Example Workflow

1. `POST /api/portfolios` — Create a portfolio for "Alice"
2. `POST /api/portfolios/{id}/deposits` — Deposit $50,000
3. `POST /api/portfolios/{id}/purchases` — Buy 10 shares of AAPL
4. `GET /api/portfolios/{id}/holdings` — Check unrealized gains
5. `POST /api/portfolios/{id}/sales` — Sell 5 shares of AAPL (FIFO)
6. `GET /api/portfolios/{id}/transactions` — View transaction history

### Running with Docker

While the application itself is not containerized in this repository, the infrastructure dependencies are fully Dockerized:

```bash
# Start infrastructure
docker-compose up -d

# Stop infrastructure
docker-compose down

# Reset data volumes
docker-compose down -v
```

The `docker-compose.yml` configures:

- MySQL with healthchecks and a dedicated database (`review-db` / `stockMarket`)
- MongoDB as a replica set (`rs0`) with automatic initialization, required for multi-document transactions

## API Overview

### Portfolio Endpoints

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/portfolios` | Create a new portfolio |
| GET | `/api/portfolios` | List all portfolios |
| GET | `/api/portfolios/{id}` | Get portfolio by ID |
| POST | `/api/portfolios/{id}/deposits` | Deposit cash |
| POST | `/api/portfolios/{id}/withdrawals` | Withdraw cash |
| POST | `/api/portfolios/{id}/purchases` | Buy shares |
| POST | `/api/portfolios/{id}/sales` | Sell shares (FIFO) |
| GET | `/api/portfolios/{id}/transactions` | Get transaction history |
| GET | `/api/portfolios/{id}/transactions?type=TYPE` | Filtered transaction history |
| GET | `/api/portfolios/{id}/holdings` | Get holdings performance |

### Stock Price Endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/stocks/{symbol}` | Get current price for a ticker |

### Error Handling

The API uses [RFC 7807 Problem Details](https://www.rfc-editor.org/rfc/rfc7807) for error responses. Global exception handling (`ExceptionHandlingAdvice`) maps domain and application exceptions to appropriate HTTP status codes:

| Exception | HTTP Status | Meaning |
|---|---|---|
| `PortfolioNotFoundException` | 404 Not Found | Portfolio ID does not exist |
| `HoldingNotFoundException` | 404 Not Found | Ticker not found in portfolio holdings |
| `InvalidAmountException` | 400 Bad Request | Zero or negative monetary amount |
| `InvalidQuantityException` | 400 Bad Request | Zero or negative share quantity |
| `InvalidTickerException` | 400 Bad Request | Ticker format invalid (must be 1-5 uppercase letters) |
| `InsufficientFundsException` | 409 Conflict | Not enough cash for purchase/withdrawal |
| `ConflictQuantityException` | 409 Conflict | Attempting to sell more shares than owned |
| `ExternalApiException` | 503 Service Unavailable | Market data provider failure |

## Database

### JPA / MySQL (Pessimistic Locking)

When the `jpa` profile is active:

- Hibernate automatically creates tables (`spring.jpa.hibernate.ddl-auto=create-drop` in local mode)
- Portfolio reads for updates use `SELECT ... FOR UPDATE` (pessimistic write lock) via `@Lock(LockModeType.PESSIMISTIC_WRITE)`
- This serializes concurrent modifications at the database row level, preventing lost updates without application-level retry logic
- `@BatchSize(size = 30)` is applied to Holdings and Lots collections to mitigate N+1 query problems

### MongoDB (Optimistic Locking + Retry)

When the `mongodb` profile is active:

- Documents use an `@Version` field for optimistic concurrency control
- The `MongoPortfolioRepository` maintains a transaction-scoped version context to handle stale reads
- If a concurrent write conflict occurs, the `RetryOnWriteConflictAspect` (Spring AOP + Spring Retry) automatically re-executes the entire use case (read → domain logic → write) up to 3 times with a 50ms fixed backoff
- The retry aspect runs outside the transactional boundary (higher precedence than `@Transactional`) so each retry gets a fresh transaction

### Transaction History

All financial operations produce immutable `Transaction` records:

| Type | Description |
|---|---|
| `DEPOSIT` | Cash added to portfolio |
| `WITHDRAWAL` | Cash removed from portfolio |
| `PURCHASE` | Shares bought (records ticker, quantity, unit price, total amount) |
| `SALE` | Shares sold (records ticker, quantity, unit price, total amount, profit) |

## Testing

StockMarket employs a comprehensive, multi-layered testing strategy aligned with the Test Pyramid:

### 1. Domain Unit Tests (`domain` module)

- Pure Java, no Spring, no mocks (except the system under test is real)
- Test aggregate invariants: `PortfolioTest`, `HoldingTest`, `LotTest`
- Test value object behavior: `MoneyTest`, `TickerTest`
- Test domain services: `HoldingPerformanceCalculatorTest`, `TransactionTest`

### 2. Application Service Unit Tests (`application` module)

- Mocked outgoing ports (no I/O)
- Verify orchestration sequences: `CashManagementServiceTest`, `PortfolioStockOperationsServiceTest`, `ReportingServiceTest`
- Verify exception propagation and edge cases
- Uses `@SpecificationRef` annotations to link tests to acceptance criteria (e.g., `US-07.FIFO-1`)

### 3. Adapter Contract Tests

- Abstract contract tests in `application` module (`AbstractPortfolioPortContractTest`, `AbstractTransactionPortContractTest`)
- Concrete implementations in JPA and MongoDB modules run the same assertions against real databases via Testcontainers
- Guarantees that JPA and MongoDB adapters behave identically from the application's perspective

### 4. Adapter-Specific Integration Tests

- `JpaPessimisticLockingTest`: Verifies `SELECT ... FOR UPDATE` lock acquisition against real MySQL
- `MongoOptimisticLockingTest`: Spawns concurrent threads to prove stale writes trigger `OptimisticLockingFailureException`

### 5. REST Integration Tests (`bootstrap` module)

- Black-box tests using RestAssured against a running Spring Boot instance on a random port
- `PortfolioLifecycleRestIntegrationTest`: End-to-end CRUD, deposits, withdrawals
- `PortfolioTradingRestIntegrationTest`: Buy/sell flows, including deterministic FIFO scenarios with a `FixedPriceStockPriceAdapter`
- `PortfolioErrorHandlingRestIntegrationTest`: 404 and validation edge cases
- `RetryOnWriteConflictJpaIntegrationTest` / `RetryOnWriteConflictMongoIntegrationTest`: Concurrent deposit stress tests

### 6. Architecture Fitness Tests

`HexagonalArchitectureTest` (ArchUnit) scans compiled bytecode to enforce:

- Domain does not depend on application, adapters, or Spring
- Application does not depend on adapters or Spring
- Inbound adapters do not depend on outbound adapters

### Running Tests

```bash
# All tests (unit + integration)
./mvnw clean verify

# Unit tests only
./mvnw test

# Integration tests only
./mvnw verify -pl bootstrap

# Specific module
./mvnw test -pl domain
```

## Build

### Standard Build

```bash
./mvnw clean install
```

### Build with Coverage

JaCoCo aggregate reports are generated in the `bootstrap` module during the verify phase:

```bash
./mvnw clean verify
# Report location: bootstrap/target/site/jacoco-merged/
```

### Build for Production

```bash
./mvnw clean package -DskipTests
# WAR artifact: bootstrap/target/stockmarket-bootstrap-0.0.1-SNAPSHOT.war
```

## Development Workflow

### Branching

- `main` is the primary branch protected by status checks
- All changes should go through Pull Requests

### CI/CD Pipeline (GitHub Actions)

The `.github/workflows/build.yml` runs on every push and PR to `main`:

1. Checkout with full fetch depth (for SonarCloud analysis)
2. Setup JDK 21 (Temurin) with Maven caching
3. Docker diagnostics (version info for debugging Testcontainers issues)
4. Build & Test: `./mvnw -B -U -ntp clean verify`
5. Failure diagnostics: Prints Surefire and Failsafe reports on failure
6. Artifacts: Uploads test results and JaCoCo coverage reports
7. SonarCloud: Quality gate analysis (requires manual setup per `CI_SETUP.md`)

### Quality Gates

- All tests must pass
- SonarCloud quality gate must pass
- ArchUnit architecture rules must pass

### Local Development Tips

- Two Maven commands to run: `install` then `spring-boot:run -pl bootstrap`. The `-pl bootstrap` restriction means sibling modules must be pre-installed.
- **IntelliJ Run Configuration**: Set Active profiles to `jpa,finhub` (or your preferred combination) and add environment variables from your `.env` file.
- **HTTP Client**: Open `doc/calls.http` in IntelliJ Ultimate to execute requests directly from the IDE.

## Future Improvements

While StockMarket is feature-complete as a pedagogical reference, the following enhancements would be natural next steps for a production system:

| Improvement | Description |
|---|---|
| **Batch Price Fetching** | The `StockPriceProviderPort` currently fetches prices sequentially. A multi-symbol endpoint override would eliminate the N+1 API call problem for large portfolios. |
| **Parallel Price Fetching** | Replace the sequential loop with a bounded `ExecutorService` (virtual threads or fixed pool) to reduce latency while respecting rate limits. |
| **Short-Lived Caching** | Extend the existing Caffeine cache with time-based eviction (e.g., 30-second TTL) to deduplicate calls for the same ticker within a request window. |
| **Event Sourcing** | Replace direct state persistence with domain events (`PortfolioCreated`, `SharesPurchased`, `SharesSold`) to enable audit trails, reactive read models, and cross-aggregate consistency. |
| **Dedicated Read Model** | Separate the holdings performance report into a CQRS read model (e.g., a materialized view or MongoDB aggregation pipeline) to avoid computing aggregates from scratch on every request. |
| **Authentication & Authorization** | Add Spring Security with JWT or OAuth2 to protect portfolio endpoints and enforce ownership rules. |
| **Rate Limiting & Resilience** | Implement Resilience4j circuit breakers and rate limiters for external market data APIs. |
| **Containerized Application** | Provide a Dockerfile and Kubernetes manifests for the Spring Boot application itself, not just the databases. |
| **Migration Tooling** | Replace `ddl-auto=create-drop` with Flyway or Liquibase for production schema management. |
| **Monitoring** | Integrate Micrometer, Prometheus, and Grafana for observability of business metrics (trade volume, error rates) and system health. |

---

## License

See [LICENSE](LICENSE) for details.
