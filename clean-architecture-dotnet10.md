# Clean Architecture in C# / .NET 10

## The Layers

Clean Architecture organizes code in concentric rings where **dependencies point inward only**. Inner layers know nothing about outer layers.

| Layer | Depends On | Responsibility |
|---|---|---|
| **Domain** | Nothing | Enterprise business rules: entities, value objects, domain events, invariants |
| **Application** | Domain | Use cases: application services, DTOs, validators, abstractions (interfaces) for the outside world |
| **Infrastructure** | Application, Domain | Implementations of those abstractions: EF Core, email, caching, identity, external APIs |
| **Presentation (API)** | Application (and Infrastructure only for DI wiring) | HTTP transport: endpoints/controllers, middleware, request/response contracts, composition root |

**Use cases** live in the Application layer as **application services**. Each feature exposes an interface (for example `IOrderService`) that describes what the system can do, and a concrete implementation that orchestrates the work: load aggregates through repositories, invoke domain behavior, persist through the unit of work, and return a `Result` or a DTO. The Presentation layer depends on the interface and calls it directly. There is no mediator and no command/query split: a single service and a single repository model handle both reads and writes.

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
│   │   │   ├── Services/
│   │   │   │   ├── IEmailService.cs
│   │   │   │   ├── IDateTimeProvider.cs
│   │   │   │   ├── ICurrentUserService.cs
│   │   │   │   ├── IJwtProvider.cs
│   │   │   │   └── ICacheService.cs
│   │   │   └── Validation/
│   │   │       └── IValidationService.cs
│   │   ├── Common/
│   │   │   ├── Models/
│   │   │   │   ├── PaginatedList.cs
│   │   │   │   └── PaginationRequest.cs
│   │   │   ├── Mappings/
│   │   │   │   └── OrderMappingExtensions.cs
│   │   │   └── Exceptions/
│   │   │       ├── ValidationException.cs
│   │   │       └── NotFoundException.cs
│   │   ├── Orders/
│   │   │   ├── IOrderService.cs
│   │   │   ├── OrderService.cs
│   │   │   ├── Dtos/
│   │   │   │   ├── CreateOrderRequest.cs
│   │   │   │   ├── OrderResponse.cs
│   │   │   │   └── OrderSummaryResponse.cs
│   │   │   ├── Validators/
│   │   │   │   ├── CreateOrderRequestValidator.cs
│   │   │   │   └── CancelOrderRequestValidator.cs
│   │   │   └── EventHandlers/
│   │   │       └── OrderCreatedDomainEventHandler.cs
│   │   ├── Customers/
│   │   │   ├── ICustomerService.cs
│   │   │   ├── CustomerService.cs
│   │   │   ├── Dtos/
│   │   │   │   ├── RegisterCustomerRequest.cs
│   │   │   │   └── CustomerResponse.cs
│   │   │   └── Validators/
│   │   │       └── RegisterCustomerRequestValidator.cs
│   │   └── Authentication/
│   │       ├── IAuthenticationService.cs
│   │       ├── AuthenticationService.cs
│   │       └── Dtos/
│   │           ├── LoginRequest.cs
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
│   │   └── ExternalServices/
│   │       ├── Payments/
│   │       │   ├── PaymentGatewayClient.cs
│   │       │   └── PaymentGatewayOptions.cs
│   │       └── Storage/
│   │           └── BlobStorageService.cs
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
    │   └── Orders/
    │       ├── OrderServiceTests.cs
    │       └── CreateOrderRequestValidatorTests.cs
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
- **`Repositories/`**: repository *interfaces* only. The Domain defines what persistence it needs (dependency inversion), and Infrastructure implements it. Because there is no separate read model, these interfaces cover both loading aggregates for changes and answering the queries the use cases need (for example `GetByIdAsync`, `ListAsync(Specification<Order>)`).
- **`Specifications/`**: reusable, composable query criteria that express business rules about selection. They are the main tool for keeping read logic in the Domain without leaking EF Core.
- **`Services/`**: domain services for logic spanning multiple aggregates that doesn't naturally fit on one entity.

### Application (`Application` project: depends on Domain)
- **`DependencyInjection.cs` / `AssemblyReference.cs`**: registers application services and validators; the assembly marker is used for scanning and architecture tests.
- **`Abstractions/Services/`**: interfaces for anything external (email, clock, current user, JWT, cache). Application declares the need, and Infrastructure fulfills it.
- **`Abstractions/Validation/`**: a small abstraction for running validators, so services don't depend on FluentValidation types directly if you prefer to keep that library swappable.
- **`Common/`**: shared helpers (paging models, mapping extensions, application exceptions) that don't belong to one feature.
- **`Orders/`, `Customers/`, `Authentication/`** (feature folders): organized by **feature/aggregate first**. Everything for one area sits together, so a change to "orders" touches one folder.
  - **`I<Feature>Service.cs`**: the **use case boundary**. This interface is what the Presentation layer depends on. Each method is one use case (`CreateAsync`, `CancelAsync`, `ShipAsync`, `GetByIdAsync`, `GetPagedAsync`).
  - **`<Feature>Service.cs`**: the implementation. It validates input, loads aggregates via repositories, calls domain behavior, commits through `IUnitOfWork`, and maps to DTOs. It holds *orchestration only*; business rules stay in the Domain.
  - **`Dtos/`**: request and response models that cross the Application boundary. Entities are never returned to the outside.
  - **`Validators/`**: FluentValidation validators for the request DTOs.
  - **`EventHandlers/`**: reactions to domain events (send email, publish integration event), kept in Application because they orchestrate use cases.

### Infrastructure (`Infrastructure` project: depends on Application + Domain)
- **`DependencyInjection.cs`**: `AddInfrastructure(...)`, registering all implementations.
- **`Persistence/`**: everything EF Core.
  - **`ApplicationDbContext.cs`**: the DbContext and `IUnitOfWork` implementation.
  - **`Configurations/`**: `IEntityTypeConfiguration<T>` mappings. They are kept out of the entities so Domain stays persistence-ignorant.
  - **`Interceptors/`**: save-time hooks for auditing, soft-delete, and dispatching domain events.
  - **`Migrations/`**: generated schema history.
  - **`Repositories/`**: concrete implementations of Domain repository interfaces. Read methods use `AsNoTracking` and projections where returning data only; write paths return tracked aggregates.
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

### Presentation / API (`Api` project: composition root)
- **`Program.cs`**: builds the host, wires up DI from all layers, and configures the pipeline. This is the only place that references Infrastructure.
- **`Endpoints/`**: Minimal API endpoints grouped by feature. Each maps HTTP to a call on an application service (`IOrderService`, `ICustomerService`), with no business logic.
- **`Controllers/`**: the MVC-controller alternative. Pick either Endpoints or Controllers, not both.
- **`Contracts/`**: HTTP request/response models. They are separate from Application DTOs so the public API can evolve independently of internal use cases; endpoints map between the two.
- **`Middleware/`**: global exception handling (`IExceptionHandler`) and request logging enrichment.
- **`Filters/`**: endpoint or action filters for cross-cutting HTTP concerns.
- **`Extensions/`**: helpers mapping `Result` → `IResult`/`ProblemDetails`, plus pipeline and service setup.
- **`OpenApi/`**: .NET 10 built-in OpenAPI document transformers (JWT security scheme, metadata).
- **`Health/`**: health check implementations exposed via `/health`.

### Tests
- **`Domain.UnitTests`**: fast, pure tests of entities and value objects with no mocks needed.
- **`Application.UnitTests`**: application services and validators with mocked repositories and abstractions.
- **`Infrastructure.IntegrationTests`**: real database (Testcontainers) to verify repositories and mappings.
- **`Api.FunctionalTests`**: end-to-end HTTP tests via `WebApplicationFactory`.
- **`ArchitectureTests`**: NetArchTest or ArchUnitNET rules that fail the build if a dependency rule is violated (e.g., Domain referencing Infrastructure).

---

## Request Flow

A typical "create order" request travels through the layers like this:

1. **Api**: the endpoint receives `CreateOrderRequest` (contract), maps it to the Application DTO, and calls `IOrderService.CreateAsync(...)`.
2. **Application**: `OrderService` validates the DTO, loads the `Customer` via `ICustomerRepository`, calls `Order.Create(...)` on the aggregate, adds it through `IOrderRepository`, and commits with `IUnitOfWork.SaveChangesAsync()`.
3. **Domain**: `Order.Create(...)` enforces invariants and raises `OrderCreatedDomainEvent`.
4. **Infrastructure**: `DispatchDomainEventsInterceptor` publishes the event on save; `OrderCreatedDomainEventHandler` reacts (for example, sends a confirmation email through `IEmailService`).
5. **Application → Api**: the service returns `Result<OrderResponse>`, and the endpoint converts it to an HTTP response via `ResultExtensions`.

Dependencies stay inward throughout: the endpoint knows the service *interface*, the service knows repository *interfaces*, and only `Program.cs` knows the concrete implementations.

---

## Notes on Variations

- **Dependency direction**: `Api → Application → Domain` and `Infrastructure → Application → Domain`. The Api references Infrastructure only to call `AddInfrastructure()` in `Program.cs`.
- **Repositories location**: some teams put repository interfaces in Application instead of Domain. Both are accepted; keeping them in Domain fits a DDD-leaning approach.
- **Application services vs. use case classes**: this layout groups related use cases in one service per feature (`OrderService`). If services grow large, split them into one class per use case (`CreateOrderUseCase`, `CancelOrderUseCase`) with a shared `IUseCase<TRequest, TResponse>` interface. This is the same architecture with finer granularity.
- **Read-heavy screens**: without a separate read model, complex list or report views are served by repository methods or specifications that project straight to DTOs. If reads become a bottleneck, add read-optimized repository methods before considering a heavier pattern.
- **Cross-cutting concerns**: without pipeline behaviors, apply validation and transactions explicitly in services, or move them to the edges (endpoint filters for validation, `IUnitOfWork` committed once at the end of each service method). Logging and exception handling stay in Api middleware.
- **Shared Contracts**: large systems sometimes add a separate `Contracts` or `SharedKernel` project.
- **Feature-folder vs. technical-folder**: the tree above organizes Application by feature, which is the prevailing modern practice over top-level `Services/`, `Dtos/`, `Validators/` folders.
- **Small projects**: this is the "all possible" version. You don't need everything, so start with Domain, Application, Infrastructure, and Api, and add the rest when a real need appears.
