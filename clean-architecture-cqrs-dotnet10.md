# Clean Architecture + CQRS in C# / .NET 10

## The Layers

Clean Architecture organizes code in concentric rings where **dependencies point inward only**. Inner layers know nothing about outer layers.

| Layer | Depends On | Responsibility |
|---|---|---|
| **Domain** | Nothing | Enterprise business rules: entities, value objects, domain events, invariants |
| **Application** | Domain | Use cases: commands, queries, handlers, validators, abstractions (interfaces) for the outside world |
| **Infrastructure** | Application, Domain | Implementations of those abstractions: EF Core, email, caching, identity, external APIs |
| **Presentation (API)** | Application (and Infrastructure only for DI wiring) | HTTP transport: endpoints/controllers, middleware, request/response contracts, composition root |

**CQRS** sits in the Application layer: **Commands** change state (return minimal data such as an ID or `Result`), and **Queries** read state (return DTOs, never mutate). Handlers are dispatched via a mediator (MediatR, or a source-generated alternative such as Mediator or Wolverine, since MediatR is now commercially licensed).

---

## ASCII Directory Tree

```
CleanArchitecture.sln
├── Directory.Build.props
├── Directory.Packages.props
├── global.json
├── .editorconfig
├── .gitignore
├── docker-compose.yml
│
├── src/
│   ├── CleanArchitecture.Domain/
│   │   ├── CleanArchitecture.Domain.csproj
│   │   ├── Abstractions/
│   │   │   ├── Entity.cs
│   │   │   ├── AggregateRoot.cs
│   │   │   ├── IAuditableEntity.cs
│   │   │   ├── ISoftDeletable.cs
│   │   │   ├── IDomainEvent.cs
│   │   │   ├── ValueObject.cs
│   │   │   ├── Result.cs
│   │   │   └── Error.cs
│   │   ├── Entities/
│   │   │   ├── Order.cs
│   │   │   ├── OrderItem.cs
│   │   │   ├── Customer.cs
│   │   │   └── Product.cs
│   │   ├── ValueObjects/
│   │   │   ├── Money.cs
│   │   │   ├── Address.cs
│   │   │   └── Email.cs
│   │   ├── Enums/
│   │   │   └── OrderStatus.cs
│   │   ├── Events/
│   │   │   ├── OrderCreatedDomainEvent.cs
│   │   │   └── OrderShippedDomainEvent.cs
│   │   ├── Exceptions/
│   │   │   ├── DomainException.cs
│   │   │   └── OrderNotFoundException.cs
│   │   ├── Errors/
│   │   │   ├── OrderErrors.cs
│   │   │   └── CustomerErrors.cs
│   │   ├── Repositories/
│   │   │   ├── IOrderRepository.cs
│   │   │   ├── ICustomerRepository.cs
│   │   │   └── IUnitOfWork.cs
│   │   ├── Specifications/
│   │   │   ├── Specification.cs
│   │   │   └── ActiveOrdersSpecification.cs
│   │   └── Services/
│   │       └── OrderPricingDomainService.cs
│   │
│   ├── CleanArchitecture.Application/
│   │   ├── CleanArchitecture.Application.csproj
│   │   ├── DependencyInjection.cs
│   │   ├── AssemblyReference.cs
│   │   ├── Abstractions/
│   │   │   ├── Messaging/
│   │   │   │   ├── ICommand.cs
│   │   │   │   ├── IQuery.cs
│   │   │   │   ├── ICommandHandler.cs
│   │   │   │   └── IQueryHandler.cs
│   │   │   ├── Behaviors/
│   │   │   │   ├── ValidationBehavior.cs
│   │   │   │   ├── LoggingBehavior.cs
│   │   │   │   ├── PerformanceBehavior.cs
│   │   │   │   ├── TransactionBehavior.cs
│   │   │   │   └── CachingBehavior.cs
│   │   │   ├── Data/
│   │   │   │   ├── IApplicationDbContext.cs
│   │   │   │   └── ISqlConnectionFactory.cs
│   │   │   ├── Services/
│   │   │   │   ├── IEmailService.cs
│   │   │   │   ├── IDateTimeProvider.cs
│   │   │   │   ├── ICurrentUserService.cs
│   │   │   │   ├── IJwtProvider.cs
│   │   │   │   └── ICacheService.cs
│   │   │   └── Caching/
│   │   │       └── ICachedQuery.cs
│   │   ├── Common/
│   │   │   ├── Models/
│   │   │   │   ├── PaginatedList.cs
│   │   │   │   └── PagedQuery.cs
│   │   │   ├── Mappings/
│   │   │   │   └── OrderMappingExtensions.cs
│   │   │   └── Exceptions/
│   │   │       ├── ValidationException.cs
│   │   │       └── NotFoundException.cs
│   │   ├── Orders/
│   │   │   ├── Commands/
│   │   │   │   ├── CreateOrder/
│   │   │   │   │   ├── CreateOrderCommand.cs
│   │   │   │   │   ├── CreateOrderCommandHandler.cs
│   │   │   │   │   └── CreateOrderCommandValidator.cs
│   │   │   │   ├── CancelOrder/
│   │   │   │   │   ├── CancelOrderCommand.cs
│   │   │   │   │   ├── CancelOrderCommandHandler.cs
│   │   │   │   │   └── CancelOrderCommandValidator.cs
│   │   │   │   └── ShipOrder/
│   │   │   │       ├── ShipOrderCommand.cs
│   │   │   │       └── ShipOrderCommandHandler.cs
│   │   │   ├── Queries/
│   │   │   │   ├── GetOrderById/
│   │   │   │   │   ├── GetOrderByIdQuery.cs
│   │   │   │   │   ├── GetOrderByIdQueryHandler.cs
│   │   │   │   │   └── OrderResponse.cs
│   │   │   │   └── GetOrders/
│   │   │   │       ├── GetOrdersQuery.cs
│   │   │   │       ├── GetOrdersQueryHandler.cs
│   │   │   │       └── OrderSummaryResponse.cs
│   │   │   └── EventHandlers/
│   │   │       └── OrderCreatedDomainEventHandler.cs
│   │   ├── Customers/
│   │   │   ├── Commands/
│   │   │   │   └── RegisterCustomer/
│   │   │   │       ├── RegisterCustomerCommand.cs
│   │   │   │       ├── RegisterCustomerCommandHandler.cs
│   │   │   │       └── RegisterCustomerCommandValidator.cs
│   │   │   └── Queries/
│   │   │       └── GetCustomerById/
│   │   │           ├── GetCustomerByIdQuery.cs
│   │   │           ├── GetCustomerByIdQueryHandler.cs
│   │   │           └── CustomerResponse.cs
│   │   └── Authentication/
│   │       ├── Commands/
│   │       │   └── Login/
│   │       │       ├── LoginCommand.cs
│   │       │       └── LoginCommandHandler.cs
│   │       └── Models/
│   │           └── TokenResponse.cs
│   │
│   ├── CleanArchitecture.Infrastructure/
│   │   ├── CleanArchitecture.Infrastructure.csproj
│   │   ├── DependencyInjection.cs
│   │   ├── Persistence/
│   │   │   ├── ApplicationDbContext.cs
│   │   │   ├── ApplicationDbContextInitialiser.cs
│   │   │   ├── UnitOfWork.cs
│   │   │   ├── Configurations/
│   │   │   │   ├── OrderConfiguration.cs
│   │   │   │   ├── OrderItemConfiguration.cs
│   │   │   │   ├── CustomerConfiguration.cs
│   │   │   │   └── ProductConfiguration.cs
│   │   │   ├── Interceptors/
│   │   │   │   ├── AuditableEntityInterceptor.cs
│   │   │   │   ├── SoftDeleteInterceptor.cs
│   │   │   │   └── DispatchDomainEventsInterceptor.cs
│   │   │   ├── Migrations/
│   │   │   │   ├── 20260101000000_InitialCreate.cs
│   │   │   │   ├── 20260101000000_InitialCreate.Designer.cs
│   │   │   │   └── ApplicationDbContextModelSnapshot.cs
│   │   │   ├── Repositories/
│   │   │   │   ├── Repository.cs
│   │   │   │   ├── OrderRepository.cs
│   │   │   │   └── CustomerRepository.cs
│   │   │   ├── Seeding/
│   │   │   │   └── DataSeeder.cs
│   │   │   └── Converters/
│   │   │       └── EmailValueConverter.cs
│   │   ├── Identity/
│   │   │   ├── ApplicationUser.cs
│   │   │   ├── IdentityService.cs
│   │   │   ├── JwtProvider.cs
│   │   │   ├── JwtOptions.cs
│   │   │   └── Authorization/
│   │   │       ├── PermissionRequirement.cs
│   │   │       ├── PermissionAuthorizationHandler.cs
│   │   │       └── PermissionAuthorizationPolicyProvider.cs
│   │   ├── Services/
│   │   │   ├── DateTimeProvider.cs
│   │   │   ├── EmailService.cs
│   │   │   ├── CurrentUserService.cs
│   │   │   └── CacheService.cs
│   │   ├── Options/
│   │   │   ├── EmailOptions.cs
│   │   │   └── DatabaseOptions.cs
│   │   ├── Outbox/
│   │   │   ├── OutboxMessage.cs
│   │   │   ├── OutboxMessageConfiguration.cs
│   │   │   └── ProcessOutboxMessagesJob.cs
│   │   ├── BackgroundJobs/
│   │   │   └── CleanupExpiredTokensJob.cs
│   │   ├── Caching/
│   │   │   └── RedisCacheService.cs
│   │   ├── Messaging/
│   │   │   ├── EventBus.cs
│   │   │   └── Consumers/
│   │   │       └── OrderPaidConsumer.cs
│   │   ├── ExternalServices/
│   │   │   ├── Payments/
│   │   │   │   ├── PaymentGatewayClient.cs
│   │   │   │   └── PaymentGatewayOptions.cs
│   │   │   └── Storage/
│   │   │       └── BlobStorageService.cs
│   │   └── Queries/
│   │       └── SqlConnectionFactory.cs
│   │
│   └── CleanArchitecture.Api/
│       ├── CleanArchitecture.Api.csproj
│       ├── Program.cs
│       ├── appsettings.json
│       ├── appsettings.Development.json
│       ├── Properties/
│       │   └── launchSettings.json
│       ├── Endpoints/
│       │   ├── IEndpoint.cs
│       │   ├── EndpointExtensions.cs
│       │   ├── Orders/
│       │   │   ├── CreateOrder.cs
│       │   │   ├── GetOrderById.cs
│       │   │   ├── GetOrders.cs
│       │   │   └── CancelOrder.cs
│       │   ├── Customers/
│       │   │   └── RegisterCustomer.cs
│       │   └── Authentication/
│       │       └── Login.cs
│       ├── Controllers/                    # alternative to Endpoints/ (pick one)
│       │   ├── ApiController.cs
│       │   ├── OrdersController.cs
│       │   └── CustomersController.cs
│       ├── Contracts/
│       │   ├── Orders/
│       │   │   ├── CreateOrderRequest.cs
│       │   │   └── UpdateOrderRequest.cs
│       │   └── Customers/
│       │       └── RegisterCustomerRequest.cs
│       ├── Middleware/
│       │   ├── GlobalExceptionHandler.cs
│       │   ├── CorrelationIdMiddleware.cs
│       │   └── RequestContextLoggingMiddleware.cs
│       ├── Filters/
│       │   └── ApiKeyAuthorizationFilter.cs
│       ├── Extensions/
│       │   ├── ResultExtensions.cs
│       │   ├── ApplicationBuilderExtensions.cs
│       │   └── ServiceCollectionExtensions.cs
│       ├── OpenApi/
│       │   ├── BearerSecuritySchemeTransformer.cs
│       │   └── ConfigureOpenApiOptions.cs
│       ├── Infrastructure/
│       │   └── CustomProblemDetailsFactory.cs
│       └── Health/
│           └── DatabaseHealthCheck.cs
│
└── tests/
    ├── CleanArchitecture.Domain.UnitTests/
    │   ├── CleanArchitecture.Domain.UnitTests.csproj
    │   ├── Orders/
    │   │   └── OrderTests.cs
    │   └── ValueObjects/
    │       └── MoneyTests.cs
    ├── CleanArchitecture.Application.UnitTests/
    │   ├── CleanArchitecture.Application.UnitTests.csproj
    │   ├── Orders/
    │   │   ├── CreateOrderCommandHandlerTests.cs
    │   │   └── CreateOrderCommandValidatorTests.cs
    │   └── Behaviors/
    │       └── ValidationBehaviorTests.cs
    ├── CleanArchitecture.Infrastructure.IntegrationTests/
    │   ├── CleanArchitecture.Infrastructure.IntegrationTests.csproj
    │   ├── BaseIntegrationTest.cs
    │   ├── IntegrationTestWebAppFactory.cs
    │   └── Orders/
    │       └── OrderRepositoryTests.cs
    ├── CleanArchitecture.Api.FunctionalTests/
    │   ├── CleanArchitecture.Api.FunctionalTests.csproj
    │   ├── CustomWebApplicationFactory.cs
    │   └── Orders/
    │       └── OrdersEndpointTests.cs
    └── CleanArchitecture.ArchitectureTests/
        ├── CleanArchitecture.ArchitectureTests.csproj
        └── LayerTests.cs
```

---

## Directory Explanations

### Solution Root
- **`Directory.Build.props`**: shared MSBuild settings (`net10.0`, nullable, warnings-as-errors, analyzers) applied to every project.
- **`Directory.Packages.props`**: Central Package Management so all projects use the same NuGet versions.
- **`global.json`**: pins the .NET 10 SDK version for consistent builds.
- **`src/` and `tests/`**: separates shippable code from test code.

### Domain (`Domain` project: zero dependencies)
- **`Abstractions/`**: base building blocks (`Entity`, `AggregateRoot`, `ValueObject`, `Result`, `Error`). They are domain concepts and must not depend on anything outside.
- **`Entities/`**: objects with identity and behavior. Business rules live here, not in services.
- **`ValueObjects/`**: immutable, equality-by-value types (`Money`, `Address`). They live in the Domain because they encode business invariants.
- **`Enums/`**: domain vocabulary such as statuses and types.
- **`Events/`**: domain events describing something that happened. Raised by aggregates, handled in the Application layer.
- **`Exceptions/`**: exceptions for broken invariants that cannot be expressed as a `Result`.
- **`Errors/`**: static `Error` definitions per aggregate, used with the `Result` pattern.
- **`Repositories/`**: repository *interfaces* only. The Domain defines what persistence it needs (dependency inversion), and Infrastructure implements it.
- **`Specifications/`**: reusable, composable query criteria that express business rules about selection.
- **`Services/`**: domain services for logic spanning multiple aggregates that doesn't naturally fit on one entity.

### Application (`Application` project: depends on Domain)
- **`DependencyInjection.cs` / `AssemblyReference.cs`**: registers handlers, validators, and behaviors; the assembly marker is used for scanning and architecture tests.
- **`Abstractions/Messaging/`**: CQRS marker interfaces (`ICommand`, `IQuery<T>`) and handler contracts. Owning them here keeps you from coupling directly to a mediator library.
- **`Abstractions/Behaviors/`**: pipeline behaviors, i.e. cross-cutting concerns (validation, logging, transactions, caching) applied around every handler.
- **`Abstractions/Data/`**: `IApplicationDbContext` and connection factories, so queries can read data without referencing EF Core or Npgsql concretely.
- **`Abstractions/Services/`**: interfaces for anything external (email, clock, current user, JWT). Application declares the need, and Infrastructure fulfills it.
- **`Common/`**: shared helpers (paging models, mapping extensions, application exceptions) that don't belong to one feature.
- **`Orders/`, `Customers/`, `Authentication/`** (feature folders): organized by **feature/aggregate first**, then by Commands/Queries. This is "vertical slice" organization inside the layer, so everything for one use case sits together.
  - **`Commands/<UseCase>/`**: the command record, its handler (state change), and its FluentValidation validator.
  - **`Queries/<UseCase>/`**: the query, its handler (read-only, often Dapper or `AsNoTracking` projections), and its response DTO.
  - **`EventHandlers/`**: reactions to domain events (send email, publish integration event), kept in Application because they orchestrate use cases.

### Infrastructure (`Infrastructure` project: depends on Application + Domain)
- **`DependencyInjection.cs`**: `AddInfrastructure(...)`, registering all implementations.
- **`Persistence/`**: everything EF Core.
  - **`ApplicationDbContext.cs`**: the DbContext and `IUnitOfWork` implementation.
  - **`Configurations/`**: `IEntityTypeConfiguration<T>` mappings. They are kept out of the entities so Domain stays persistence-ignorant.
  - **`Interceptors/`**: save-time hooks for auditing, soft-delete, and dispatching domain events.
  - **`Migrations/`**: generated schema history.
  - **`Repositories/`**: concrete implementations of Domain repository interfaces.
  - **`Seeding/`**: initial and reference data.
  - **`Converters/`**: EF value converters for value objects.
- **`Identity/`**: ASP.NET Identity, JWT generation, permission-based authorization handlers. These are framework-specific concerns.
- **`Services/`**: implementations of the Application service interfaces (clock, email, current user).
- **`Options/`**: strongly-typed configuration classes (`IOptions<T>`) for infrastructure settings.
- **`Outbox/`**: Transactional Outbox pattern for reliable event publishing, with a background job that processes stored messages.
- **`BackgroundJobs/`**: scheduled and recurring work (Quartz, Hangfire, or hosted services).
- **`Caching/`**: Redis or in-memory cache implementations.
- **`Messaging/`**: message-broker integration (MassTransit, Azure Service Bus) and consumers.
- **`ExternalServices/`**: typed HTTP clients and SDK wrappers for third parties (payments, storage).
- **`Queries/`**: the Dapper/raw-SQL connection factory used by query handlers for fast reads.

### Presentation / API (`Api` project: composition root)
- **`Program.cs`**: builds the host, wires up DI from all layers, and configures the pipeline. This is the only place that references Infrastructure.
- **`Endpoints/`**: Minimal API endpoints grouped by feature. Each maps HTTP to a command or query via the sender, with no business logic.
- **`Controllers/`**: the MVC-controller alternative. Pick either Endpoints or Controllers, not both.
- **`Contracts/`**: HTTP request/response models. They are separate from Application commands so the public API can evolve independently of internal use cases.
- **`Middleware/`**: global exception handling (`IExceptionHandler`), correlation ID propagation, and request logging enrichment. Order in `Program.cs` matters: correlation ID runs first so every later component — logging, exception handling — can see it.
  - **`CorrelationIdMiddleware.cs`**: a hand-rolled middleware, not a package. It reads an inbound `X-Correlation-Id` header if present, otherwise generates a new `Guid`, and:
    1. Stashes the ID on `HttpContext.Items` (or a small `ICorrelationIdProvider` registered as scoped) so anything later in the request — the sender, handlers, event handlers — can read it without touching `HttpContext` directly. A `LoggingBehavior` pipeline behavior is the natural place to pull it in for command/query logging.
    2. Pushes it into an `ILogger` scope (`logger.BeginScope(...)`) so every log line for the request, including those from `LoggingBehavior`, carries it automatically.
    3. Writes it back onto the response headers before the response starts, so callers can correlate their request with server-side logs.
    4. Flows it onto any outbound `HttpClient` calls (via a `DelegatingHandler` in Infrastructure/ExternalServices) and onto outbox/queue messages, so the ID survives across service boundaries.
  - **`RequestContextLoggingMiddleware.cs`**: enriches the log scope further (user ID, tenant, route), typically registered immediately after `CorrelationIdMiddleware` so it can include the correlation ID's scope.
- **`Filters/`**: endpoint or action filters for cross-cutting HTTP concerns.
- **`Extensions/`**: helpers mapping `Result` → `IResult`/`ProblemDetails`, plus pipeline and service setup.
- **`OpenApi/`**: .NET 10 built-in OpenAPI document transformers (JWT security scheme, metadata).
- **`Health/`**: health check implementations exposed via `/health`.

### Tests
- **`Domain.UnitTests`**: fast, pure tests of entities and value objects with no mocks needed.
- **`Application.UnitTests`**: handlers, validators, and behaviors with mocked abstractions.
- **`Infrastructure.IntegrationTests`**: real database (Testcontainers) to verify repositories and mappings.
- **`Api.FunctionalTests`**: end-to-end HTTP tests via `WebApplicationFactory`.
- **`ArchitectureTests`**: NetArchTest or ArchUnitNET rules that fail the build if a dependency rule is violated (e.g., Domain referencing Infrastructure).

---

## Notes on Variations

- **Dependency direction**: `Api → Application → Domain` and `Infrastructure → Application → Domain`. The Api references Infrastructure only to call `AddInfrastructure()` in `Program.cs`.
- **Repositories location**: some teams put repository interfaces in Application instead of Domain. Both are accepted; keeping them in Domain fits a DDD-leaning approach.
- **Shared Contracts**: large systems sometimes add a separate `Contracts` or `SharedKernel` project.
- **Feature-folder vs. technical-folder**: the tree above organizes Application by feature, which is the prevailing modern practice over `Commands/`, `Queries/` at the top level.
- **Mediator library**: with MediatR now commercial, alternatives include the `Mediator` source generator, Wolverine, or a hand-rolled dispatcher using the `ICommandHandler`/`IQueryHandler` abstractions above.
- **Correlation IDs**: `CorrelationIdMiddleware` is intentionally hand-rolled rather than pulled from a package — it's ~30 lines, and owning it means you control the header name, the ID format, and exactly where it gets attached (log scope, response header, outbound `HttpClient`, outbox messages). It must be one of the first entries in the middleware chain in `Program.cs`, before exception handling and logging, so those components — and `LoggingBehavior` further down the pipeline — can pick up the ID. If you also use OpenTelemetry, treat the correlation ID as a business-facing concern distinct from the OTel trace ID: keep both, since callers and support tooling look for the simple header while tracing systems use the W3C `traceparent`.
- **Small projects**: this is the "all possible" version. You don't need everything, so start with Domain, Application, Infrastructure, and Api, and add the rest when a real need appears.
