# Clean Architecture Application Layer
## Modular Monolith + CQRS + MediatR with C# / .NET

> A practical guide for organizing the **Application Layer** in a modular monolith using **Clean Architecture**, **CQRS**, and **MediatR**.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Goals](#architecture-goals)
3. [Recommended Solution Structure](#recommended-solution-structure)
4. [Module Structure](#module-structure)
5. [Application Layer Structure](#application-layer-structure)
6. [Feature-Based Organization](#feature-based-organization)
7. [Commands](#commands)
8. [Queries](#queries)
9. [Handlers](#handlers)
10. [DTOs and Result Models](#dtos-and-result-models)
11. [Validators](#validators)
12. [Application Interfaces](#application-interfaces)
13. [Repository Abstractions](#repository-abstractions)
14. [External Service Abstractions](#external-service-abstractions)
15. [MediatR Pipeline Behaviors](#mediatr-pipeline-behaviors)
16. [Domain vs Application Responsibilities](#domain-vs-application-responsibilities)
17. [Infrastructure Responsibilities](#infrastructure-responsibilities)
18. [Complete Order Example](#complete-order-example)
19. [Dependency Injection](#dependency-injection)
20. [Caching Pattern](#caching-pattern)
21. [Transactions](#transactions)
22. [Domain Events](#domain-events)
23. [Cross-Module Communication](#cross-module-communication)
24. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
25. [Testing](#testing)
26. [Recommended Final Structure](#recommended-final-structure)
27. [Practical Rules](#practical-rules)

---

# Overview

A **Modular Monolith** is a single deployable application composed of strongly separated business modules.

For example:

- Sales
- Orders
- Inventory
- Customers
- Payments

Each module owns its business functionality and follows Clean Architecture internally.

A typical module can be organized as:

```text
Sales
├── Domain
├── Application
├── Infrastructure
└── Presentation
```

The Application Layer is responsible for implementing **use cases**.

With CQRS:

```text
Command  -> changes state
Query    -> reads state
```

With MediatR:

```text
API/UI
   |
   v
IMediator.Send(...)
   |
   v
Command / Query
   |
   v
Handler
   |
   v
Domain + Application Abstractions
   |
   v
Infrastructure
```

The important idea is:

> Organize the Application Layer around business capabilities/use cases rather than creating large technical folders.

---

# Architecture Goals

The Application Layer should:

- represent application use cases
- coordinate domain operations
- define abstractions for infrastructure dependencies
- contain CQRS commands and queries
- contain MediatR handlers
- perform application-level validation
- map application data to DTOs
- coordinate transactions when appropriate
- remain independent of infrastructure implementations

The Application Layer should **not**:

- contain EF Core `DbContext` implementations
- directly call SQL
- contain HTTP implementation details
- contain ASP.NET controllers
- contain UI code
- contain infrastructure-specific implementations
- contain large amounts of domain business logic

---

# Recommended Solution Structure

A modular monolith might look like this:

```text
src/
│
├── Modules/
│   │
│   ├── Sales/
│   │   ├── Sales.Domain/
│   │   ├── Sales.Application/
│   │   ├── Sales.Infrastructure/
│   │   └── Sales.Presentation/
│   │
│   ├── Inventory/
│   │   ├── Inventory.Domain/
│   │   ├── Inventory.Application/
│   │   ├── Inventory.Infrastructure/
│   │   └── Inventory.Presentation/
│   │
│   └── Customers/
│       ├── Customers.Domain/
│       ├── Customers.Application/
│       ├── Customers.Infrastructure/
│       └── Customers.Presentation/
│
├── BuildingBlocks/
│   ├── Domain/
│   ├── Application/
│   └── Infrastructure/
│
└── Host/
    └── Web/
```

For a larger system, each module can have its own Clean Architecture boundary.

---

# Module Structure

For example, an Order module:

```text
Orders/
│
├── Orders.Domain/
│   ├── Entities/
│   │   ├── Order.cs
│   │   └── OrderItem.cs
│   │
│   ├── ValueObjects/
│   ├── Events/
│   ├── Exceptions/
│   └── Repositories/
│
├── Orders.Application/
│
├── Orders.Infrastructure/
│   ├── Persistence/
│   ├── Repositories/
│   └── Services/
│
└── Orders.Presentation/
    ├── Controllers/
    └── Endpoints/
```

The dependency direction should generally be:

```text
Presentation
     |
     v
Application
     |
     v
Domain

Infrastructure
     |
     +------> Application
     |
     +------> Domain
```

The Domain should remain at the center.

---

# Application Layer Structure

There are two common ways to organize CQRS.

## Technical organization

```text
Application/
├── Commands/
├── Queries/
├── Validators/
├── Behaviors/
├── Interfaces/
└── DTOs/
```

This works for small applications.

However, as the application grows, these folders can become very large.

A better approach for a modular monolith is usually:

## Feature-based organization

```text
Orders.Application/
│
├── Orders/
│   ├── CreateOrder/
│   │   ├── CreateOrderCommand.cs
│   │   ├── CreateOrderCommandHandler.cs
│   │   └── CreateOrderCommandValidator.cs
│   │
│   ├── CancelOrder/
│   │   ├── CancelOrderCommand.cs
│   │   ├── CancelOrderCommandHandler.cs
│   │   └── CancelOrderCommandValidator.cs
│   │
│   ├── GetOrder/
│   │   ├── GetOrderQuery.cs
│   │   ├── GetOrderQueryHandler.cs
│   │   └── OrderDto.cs
│   │
│   └── GetOrders/
│       ├── GetOrdersQuery.cs
│       ├── GetOrdersQueryHandler.cs
│       └── OrderListItemDto.cs
│
├── Abstractions/
│   ├── Persistence/
│   │   └── IOrderRepository.cs
│   │
│   ├── Services/
│   │   ├── IEmailSender.cs
│   │   └── ICurrentUser.cs
│   │
│   └── Caching/
│       └── ICacheService.cs
│
├── Behaviors/
│   ├── ValidationBehavior.cs
│   ├── LoggingBehavior.cs
│   ├── TransactionBehavior.cs
│   └── PerformanceBehavior.cs
│
└── DependencyInjection.cs
```

This structure is recommended because a use case is kept together.

---

# Feature-Based Organization

Suppose the Orders module has these operations:

```text
Create Order
Cancel Order
Get Order
Get Orders
Update Order
```

Instead of:

```text
Commands/
    CreateOrderCommand.cs
    CancelOrderCommand.cs
    UpdateOrderCommand.cs

Queries/
    GetOrderQuery.cs
    GetOrdersQuery.cs

Handlers/
    CreateOrderCommandHandler.cs
    CancelOrderCommandHandler.cs
    ...
```

use:

```text
Orders/
├── CreateOrder/
│   ├── CreateOrderCommand.cs
│   ├── CreateOrderCommandHandler.cs
│   └── CreateOrderCommandValidator.cs
│
├── CancelOrder/
│   ├── CancelOrderCommand.cs
│   ├── CancelOrderCommandHandler.cs
│   └── CancelOrderCommandValidator.cs
│
├── GetOrder/
│   ├── GetOrderQuery.cs
│   ├── GetOrderQueryHandler.cs
│   └── OrderDto.cs
│
└── GetOrders/
    ├── GetOrdersQuery.cs
    ├── GetOrdersQueryHandler.cs
    └── OrderListItemDto.cs
```

This is often called **vertical slice organization**.

It works particularly well with CQRS.

---

# Commands

A command represents an operation that changes application state.

Examples:

```text
CreateOrder
UpdateOrder
CancelOrder
ApproveOrder
ShipOrder
```

Example:

```csharp
public sealed record CreateOrderCommand(
    Guid CustomerId,
    IReadOnlyCollection<CreateOrderItem> Items
) : IRequest<Guid>;
```

The command should primarily contain the input required by the use case.

It should not contain infrastructure logic.

---

# Command Handler

The handler coordinates the use case.

```csharp
public sealed class CreateOrderCommandHandler
    : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _orderRepository;

    public CreateOrderCommandHandler(
        IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public async Task<Guid> Handle(
        CreateOrderCommand request,
        CancellationToken cancellationToken)
    {
        var order = Order.Create(request.CustomerId);

        foreach (var item in request.Items)
        {
            order.AddItem(
                item.ProductId,
                item.Quantity);
        }

        await _orderRepository.AddAsync(
            order,
            cancellationToken);

        return order.Id;
    }
}
```

Notice that the handler does not know whether the repository uses:

- EF Core
- Dapper
- SQL
- Cosmos DB
- another persistence technology

It only knows the application abstraction:

```csharp
IOrderRepository
```

---

# Commands Should Express Intent

Prefer:

```csharp
CancelOrderCommand
```

over:

```csharp
UpdateOrderStatusCommand
```

when the business operation is actually cancellation.

The command should express business intent.

Good:

```text
ApproveOrderCommand
CancelOrderCommand
ShipOrderCommand
```

Less expressive:

```text
SetOrderStatusCommand
UpdateOrderCommand
ModifyOrderCommand
```

Business intent makes the application easier to understand.

---

# Queries

Queries retrieve information without changing application state.

Examples:

```text
GetOrder
GetOrders
GetCustomerOrders
SearchOrders
GetOrderSummary
```

Example:

```csharp
public sealed record GetOrderQuery(
    Guid OrderId
) : IRequest<OrderDto?>;
```

Handler:

```csharp
public sealed class GetOrderQueryHandler
    : IRequestHandler<GetOrderQuery, OrderDto?>
{
    private readonly IOrderQueries _queries;

    public GetOrderQueryHandler(IOrderQueries queries)
    {
        _queries = queries;
    }

    public async Task<OrderDto?> Handle(
        GetOrderQuery request,
        CancellationToken cancellationToken)
    {
        return await _queries.GetByIdAsync(
            request.OrderId,
            cancellationToken);
    }
}
```

---

# Queries Are Often Different from Commands

A useful CQRS rule is:

> Commands should generally work with domain objects. Queries can be optimized specifically for reading.

For example:

```text
Command
    |
    v
Order aggregate
    |
    v
Repository
```

while:

```text
Query
    |
    v
IOrderQueries
    |
    v
Dapper / EF Core projection
    |
    v
OrderDto
```

You don't necessarily need to use the same repository abstraction for both.

---

# DTOs

DTO means **Data Transfer Object**.

A query can return a DTO designed specifically for the UI/API.

Example:

```csharp
public sealed record OrderDto(
    Guid Id,
    Guid CustomerId,
    string Status,
    decimal Total);
```

For a list:

```csharp
public sealed record OrderListItemDto(
    Guid Id,
    string CustomerName,
    string Status,
    decimal Total,
    DateTime CreatedAt);
```

Avoid exposing domain entities directly from the Application Layer.

Instead of:

```csharp
IRequest<Order>
```

prefer:

```csharp
IRequest<OrderDto>
```

---

# Validators

Application validation verifies that the request is valid for the use case.

For example, with FluentValidation:

```csharp
public sealed class CreateOrderCommandValidator
    : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty();

        RuleFor(x => x.Items)
            .NotEmpty();

        RuleForEach(x => x.Items)
            .ChildRules(item =>
            {
                item.RuleFor(x => x.ProductId)
                    .NotEmpty();

                item.RuleFor(x => x.Quantity)
                    .GreaterThan(0);
            });
    }
}
```

Validation belongs in the Application Layer when it concerns the application request.

---

# Application Validation vs Domain Validation

This distinction is important.

## Application validation

Example:

```text
CustomerId is required.
Items collection cannot be empty.
Quantity must be greater than zero.
```

These validate the incoming request.

## Domain validation

Example:

```text
An order cannot be cancelled after it has shipped.
An order cannot contain duplicate products.
A paid order cannot be modified.
```

These are business rules and should normally be protected by the Domain.

Example:

```csharp
public void Cancel()
{
    if (Status == OrderStatus.Shipped)
    {
        throw new OrderCannotBeCancelledException(Id);
    }

    Status = OrderStatus.Cancelled;
}
```

The handler should call:

```csharp
order.Cancel();
```

rather than implementing the business rule itself.

---

# Application Abstractions

The Application Layer can define interfaces for things it needs.

Example:

```text
Abstractions/
├── Persistence/
├── Services/
├── Caching/
└── Identity/
```

Example:

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken);

    Task AddAsync(
        Order order,
        CancellationToken cancellationToken);
}
```

Infrastructure implements it:

```csharp
public sealed class OrderRepository : IOrderRepository
{
    private readonly OrdersDbContext _dbContext;

    public OrderRepository(OrdersDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<Order?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken)
    {
        return await _dbContext.Orders
            .Include(x => x.Items)
            .SingleOrDefaultAsync(
                x => x.Id == id,
                cancellationToken);
    }

    public async Task AddAsync(
        Order order,
        CancellationToken cancellationToken)
    {
        await _dbContext.Orders.AddAsync(
            order,
            cancellationToken);
    }
}
```

---

# Repository Abstractions

Do not automatically create a repository for every table.

A repository should generally represent an aggregate or meaningful domain boundary.

Good:

```csharp
IOrderRepository
ICustomerRepository
```

Potentially unnecessary:

```csharp
IOrderItemRepository
```

if `OrderItem` only exists inside the `Order` aggregate.

For example:

```text
Order
 └── OrderItem
```

The application should usually modify `OrderItem` through `Order`.

---

# Query Abstractions

For CQRS, a separate read abstraction can be useful.

Example:

```csharp
public interface IOrderQueries
{
    Task<OrderDto?> GetByIdAsync(
        Guid orderId,
        CancellationToken cancellationToken);

    Task<IReadOnlyList<OrderListItemDto>> GetListAsync(
        CancellationToken cancellationToken);
}
```

Infrastructure can implement it using:

- EF Core
- Dapper
- SQL views
- stored procedures
- another read database

The Application Layer does not need to know.

---

# External Service Abstractions

Suppose creating an order requires sending an email.

Application:

```csharp
public interface IEmailSender
{
    Task SendAsync(
        string recipient,
        string subject,
        string body,
        CancellationToken cancellationToken);
}
```

Infrastructure:

```csharp
public sealed class EmailSender : IEmailSender
{
    // SMTP, SendGrid, Azure Communication Services, etc.
}
```

Handler:

```csharp
await _emailSender.SendAsync(
    customer.Email,
    "Order Created",
    "Your order has been created.",
    cancellationToken);
```

The Application Layer does not care how email is delivered.

---

# Current User Abstraction

Another useful abstraction is:

```csharp
public interface ICurrentUser
{
    Guid? UserId { get; }

    bool IsAuthenticated { get; }
}
```

The Application Layer can use:

```csharp
var userId = _currentUser.UserId;
```

without depending on:

```csharp
HttpContext
ClaimsPrincipal
HttpContextAccessor
```

Those details belong outside the Application Layer.

---

# MediatR Pipeline Behaviors

MediatR pipeline behaviors are excellent for cross-cutting concerns.

Typical structure:

```text
Request
   |
   v
LoggingBehavior
   |
   v
ValidationBehavior
   |
   v
TransactionBehavior
   |
   v
PerformanceBehavior
   |
   v
Handler
```

Common behaviors include:

```text
Validation
Logging
Transaction
Authorization
Performance monitoring
Caching
Retry
```

---

# Validation Behavior

Example:

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(
        IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!_validators.Any())
        {
            return await next();
        }

        var context = new ValidationContext<TRequest>(request);

        var failures = _validators
            .SelectMany(x => x.Validate(context).Errors)
            .Where(x => x != null)
            .ToList();

        if (failures.Count > 0)
        {
            throw new ValidationException(failures);
        }

        return await next();
    }
}
```

This means individual handlers do not need:

```csharp
validator.Validate(...)
```

The pipeline handles it automatically.

---

# Logging Behavior

A logging behavior can surround every request:

```csharp
public sealed class LoggingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(
        ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "Handling {RequestName}",
            typeof(TRequest).Name);

        var response = await next();

        _logger.LogInformation(
            "Handled {RequestName}",
            typeof(TRequest).Name);

        return response;
    }
}
```

---

# Transaction Behavior

A transaction behavior can be applied to commands.

Conceptually:

```text
Command
   |
   v
Begin Transaction
   |
   v
Handler
   |
   +--> Domain changes
   |
   +--> Save changes
   |
   v
Commit
```

If an exception occurs:

```text
Rollback
```

A useful refinement is to apply transactions only to commands that require them rather than every request.

---

# Domain vs Application Responsibilities

A common source of confusion is deciding where logic belongs.

## Domain

The Domain answers:

> What is allowed according to the business?

Example:

```csharp
order.Cancel();
```

The Order entity decides whether cancellation is allowed.

## Application

The Application answers:

> What steps are required to perform this use case?

Example:

```text
Get order
Check permissions
Cancel order
Save order
Publish event
```

## Infrastructure

Infrastructure answers:

> How do we communicate with external technology?

Example:

```text
EF Core
SQL Server
Redis
Azure Service Bus
Email provider
HTTP API
```

## Presentation

Presentation answers:

> How does the outside world communicate with the application?

Example:

```text
HTTP
Blazor
Razor Pages
gRPC
```

---

# Complete Order Example

## Domain

```csharp
public sealed class Order
{
    private readonly List<OrderItem> _items = [];

    public Guid Id { get; private set; }

    public Guid CustomerId { get; private set; }

    public OrderStatus Status { get; private set; }

    public IReadOnlyCollection<OrderItem> Items => _items;

    private Order()
    {
    }

    private Order(Guid customerId)
    {
        Id = Guid.NewGuid();
        CustomerId = customerId;
        Status = OrderStatus.Pending;
    }

    public static Order Create(Guid customerId)
    {
        return new Order(customerId);
    }

    public void AddItem(Guid productId, int quantity)
    {
        if (quantity <= 0)
        {
            throw new InvalidOperationException(
                "Quantity must be greater than zero.");
        }

        _items.Add(
            new OrderItem(productId, quantity));
    }

    public void Cancel()
    {
        if (Status == OrderStatus.Shipped)
        {
            throw new InvalidOperationException(
                "A shipped order cannot be cancelled.");
        }

        Status = OrderStatus.Cancelled;
    }
}
```

---

# Create Order Feature

Directory:

```text
Orders.Application/
└── Orders/
    └── CreateOrder/
        ├── CreateOrderCommand.cs
        ├── CreateOrderCommandHandler.cs
        └── CreateOrderCommandValidator.cs
```

## Command

```csharp
public sealed record CreateOrderCommand(
    Guid CustomerId,
    IReadOnlyCollection<CreateOrderItem> Items
) : IRequest<Guid>;

public sealed record CreateOrderItem(
    Guid ProductId,
    int Quantity);
```

## Validator

```csharp
public sealed class CreateOrderCommandValidator
    : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty();

        RuleFor(x => x.Items)
            .NotEmpty();

        RuleForEach(x => x.Items)
            .ChildRules(item =>
            {
                item.RuleFor(x => x.ProductId)
                    .NotEmpty();

                item.RuleFor(x => x.Quantity)
                    .GreaterThan(0);
            });
    }
}
```

## Handler

```csharp
public sealed class CreateOrderCommandHandler
    : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _repository;

    public CreateOrderCommandHandler(
        IOrderRepository repository)
    {
        _repository = repository;
    }

    public async Task<Guid> Handle(
        CreateOrderCommand request,
        CancellationToken cancellationToken)
    {
        var order = Order.Create(request.CustomerId);

        foreach (var item in request.Items)
        {
            order.AddItem(
                item.ProductId,
                item.Quantity);
        }

        await _repository.AddAsync(
            order,
            cancellationToken);

        return order.Id;
    }
}
```

The handler coordinates the operation.

The Domain owns business rules.

---

# Get Order Feature

Directory:

```text
Orders.Application/
└── Orders/
    └── GetOrder/
        ├── GetOrderQuery.cs
        ├── GetOrderQueryHandler.cs
        └── OrderDto.cs
```

## Query

```csharp
public sealed record GetOrderQuery(
    Guid OrderId
) : IRequest<OrderDto?>;
```

## DTO

```csharp
public sealed record OrderDto(
    Guid Id,
    Guid CustomerId,
    string Status,
    decimal Total);
```

## Handler

```csharp
public sealed class GetOrderQueryHandler
    : IRequestHandler<GetOrderQuery, OrderDto?>
{
    private readonly IOrderQueries _queries;

    public GetOrderQueryHandler(
        IOrderQueries queries)
    {
        _queries = queries;
    }

    public async Task<OrderDto?> Handle(
        GetOrderQuery request,
        CancellationToken cancellationToken)
    {
        return await _queries.GetByIdAsync(
            request.OrderId,
            cancellationToken);
    }
}
```

---

# Cancel Order Feature

Directory:

```text
Orders.Application/
└── Orders/
    └── CancelOrder/
        ├── CancelOrderCommand.cs
        ├── CancelOrderCommandHandler.cs
        └── CancelOrderCommandValidator.cs
```

## Command

```csharp
public sealed record CancelOrderCommand(
    Guid OrderId
) : IRequest;
```

## Handler

```csharp
public sealed class CancelOrderCommandHandler
    : IRequestHandler<CancelOrderCommand>
{
    private readonly IOrderRepository _repository;

    public CancelOrderCommandHandler(
        IOrderRepository repository)
    {
        _repository = repository;
    }

    public async Task Handle(
        CancelOrderCommand request,
        CancellationToken cancellationToken)
    {
        var order = await _repository.GetByIdAsync(
            request.OrderId,
            cancellationToken);

        if (order is null)
        {
            throw new KeyNotFoundException(
                $"Order {request.OrderId} was not found.");
        }

        order.Cancel();

        await _repository.SaveChangesAsync(
            cancellationToken);
    }
}
```

Notice:

```csharp
order.Cancel();
```

instead of:

```csharp
order.Status = OrderStatus.Cancelled;
```

The latter would move business logic into the Application Layer.

---

# Caching Pattern

Suppose we want:

```text
Check cache
    |
    +-- cached -> return cached data
    |
    +-- not cached -> query database
                         |
                         v
                      cache data
                         |
                         v
                      return data
```

Create an abstraction:

```csharp
public interface ICacheService
{
    Task<T?> GetAsync<T>(
        string key,
        CancellationToken cancellationToken);

    Task SetAsync<T>(
        string key,
        T value,
        TimeSpan expiration,
        CancellationToken cancellationToken);
}
```

The query handler can then use it:

```csharp
public async Task<OrderDto?> Handle(
    GetOrderQuery request,
    CancellationToken cancellationToken)
{
    var cacheKey = $"order:{request.OrderId}";

    var cached = await _cache.GetAsync<OrderDto>(
        cacheKey,
        cancellationToken);

    if (cached is not null)
    {
        return cached;
    }

    var order = await _queries.GetByIdAsync(
        request.OrderId,
        cancellationToken);

    if (order is null)
    {
        return null;
    }

    await _cache.SetAsync(
        cacheKey,
        order,
        TimeSpan.FromMinutes(10),
        cancellationToken);

    return order;
}
```

For reusable caching behavior, a MediatR pipeline behavior can be considered instead.

Do not introduce a generic caching abstraction everywhere unless caching is actually needed.

---

# Transactions

A command that performs multiple changes may require a transaction.

Example:

```text
Create Order
   |
   +-- Insert Order
   |
   +-- Insert Order Items
   |
   +-- Update Inventory
   |
   +-- Save
```

If these changes must succeed or fail together, use a transaction.

The transaction mechanism belongs to Infrastructure, while the Application Layer can define an abstraction if necessary:

```csharp
public interface IUnitOfWork
{
    Task SaveChangesAsync(
        CancellationToken cancellationToken);
}
```

Depending on the architecture, EF Core's `DbContext` can itself implement the unit-of-work behavior, so creating an additional `IUnitOfWork` abstraction is not always necessary.

Avoid abstractions that exist only because a pattern says you should have them.

---

# Domain Events

Suppose an order is created.

The Domain can raise:

```csharp
OrderCreatedDomainEvent
```

Example:

```csharp
public sealed record OrderCreatedDomainEvent(
    Guid OrderId) : IDomainEvent;
```

The event represents something that happened in the domain.

Handlers can react to it:

```text
Order
  |
  +--> OrderCreated
          |
          +--> Send confirmation email
          |
          +--> Update reporting
          |
          +--> Notify inventory
```

This helps avoid putting unrelated operations inside the Order aggregate.

---

# Cross-Module Communication

This is especially important in a modular monolith.

Suppose:

```text
Orders Module
Inventory Module
Customers Module
```

Avoid:

```text
Orders.Application
    |
    +--> directly accesses InventoryDbContext
```

That creates tight coupling.

Prefer a module contract.

For example:

```csharp
public interface IInventoryService
{
    Task ReserveAsync(
        IReadOnlyCollection<InventoryReservation> items,
        CancellationToken cancellationToken);
}
```

Or use an integration/domain event approach:

```text
Orders
  |
  +--> OrderCreated
           |
           v
      Inventory
```

The exact approach depends on whether the interaction must be synchronous or asynchronous.

---

# Dependency Rules

A useful dependency rule is:

```text
Domain
  ^
  |
Application
  ^
  |
Presentation

Infrastructure
  |
  +----> Application
  |
  +----> Domain
```

The Application Layer should not depend on:

```text
Orders.Infrastructure
EntityFrameworkCore
SqlConnection
Redis client
HttpClient implementation details
ASP.NET Controller
Blazor component
```

Instead:

```text
Application
    |
    +--> IOrderRepository
    +--> IOrderQueries
    +--> IEmailSender
    +--> ICacheService
```

Infrastructure implements these abstractions.

---

# Dependency Injection

Create a registration class in the Application project.

```csharp
public static class DependencyInjection
{
    public static IServiceCollection AddOrdersApplication(
        this IServiceCollection services)
    {
        services.AddMediatR(configuration =>
        {
            configuration.RegisterServicesFromAssembly(
                typeof(DependencyInjection).Assembly);
        });

        services.AddValidatorsFromAssembly(
            typeof(DependencyInjection).Assembly);

        services.AddTransient(
            typeof(IPipelineBehavior<,>),
            typeof(ValidationBehavior<,>));

        services.AddTransient(
            typeof(IPipelineBehavior<,>),
            typeof(LoggingBehavior<,>));

        return services;
    }
}
```

Then the Host can use:

```csharp
builder.Services.AddOrdersApplication();
```

Infrastructure registration can be separate:

```csharp
builder.Services.AddOrdersInfrastructure(
    builder.Configuration);
```

This keeps composition clean.

---

# Presentation with MediatR

A controller should remain thin.

```csharp
[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly ISender _sender;

    public OrdersController(ISender sender)
    {
        _sender = sender;
    }

    [HttpPost]
    public async Task<IActionResult> Create(
        CreateOrderCommand command,
        CancellationToken cancellationToken)
    {
        var id = await _sender.Send(
            command,
            cancellationToken);

        return Ok(id);
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> Get(
        Guid id,
        CancellationToken cancellationToken)
    {
        var result = await _sender.Send(
            new GetOrderQuery(id),
            cancellationToken);

        if (result is null)
        {
            return NotFound();
        }

        return Ok(result);
    }
}
```

The controller should not contain business logic.

---

# Application Layer Project References

For example:

```text
Orders.Application
    |
    +----> Orders.Domain
```

Ideally:

```text
Orders.Application
    |
    +----> MediatR
    +----> FluentValidation
    +----> Orders.Domain
```

It should not reference:

```text
Orders.Infrastructure
```

This is one of the most important boundaries.

---

# What Goes Where?

## Domain

```text
Entities
Value Objects
Aggregates
Domain Events
Business Rules
Domain Exceptions
Domain Services
```

## Application

```text
Commands
Queries
Handlers
DTOs
Validators
Application Services
Interfaces
MediatR Behaviors
Use-case orchestration
```

## Infrastructure

```text
EF Core
DbContext
Repositories
Dapper
Redis
Email
Azure services
External APIs
Message brokers
File storage
```

## Presentation

```text
Controllers
Endpoints
Blazor components
Request/response HTTP concerns
Authentication middleware
```

---

# Anti-Patterns to Avoid

## 1. One giant Application folder

Avoid:

```text
Application/
├── Commands/
│   ├── Command1.cs
│   ├── Command2.cs
│   ├── Command3.cs
│   └── ...
├── Handlers/
│   ├── Handler1.cs
│   ├── Handler2.cs
│   └── ...
└── DTOs/
    ├── Dto1.cs
    └── ...
```

As the application grows, it becomes difficult to locate a use case.

Prefer feature folders.

---

## 2. Fat handlers

Avoid:

```csharp
public async Task<Guid> Handle(...)
{
    // 500 lines
    // database queries
    // business rules
    // email
    // HTTP calls
    // mapping
    // calculations
}
```

The handler should orchestrate.

If a complex business rule exists, move it into the Domain.

If infrastructure code is needed, create an abstraction and implement it in Infrastructure.

---

## 3. Domain logic in handlers

Avoid:

```csharp
if (order.Status == OrderStatus.Shipped)
{
    throw new Exception(...);
}

order.Status = OrderStatus.Cancelled;
```

Prefer:

```csharp
order.Cancel();
```

The Order aggregate owns the rule.

---

## 4. Returning entities directly

Avoid:

```csharp
IRequest<Order>
```

Prefer:

```csharp
IRequest<OrderDto>
```

This prevents the domain model from becoming an API contract.

---

## 5. Generic repositories everywhere

Avoid automatically creating:

```text
IGenericRepository<T>
```

for every entity.

CQRS often benefits from use-case-specific persistence operations.

For example:

```csharp
IOrderRepository
IOrderQueries
```

can be more useful than:

```csharp
IGenericRepository<Order>
```

---

## 6. Application depending on Infrastructure

Avoid:

```text
Orders.Application
      |
      v
Orders.Infrastructure
```

Instead:

```text
Orders.Application
      |
      v
Abstractions

Orders.Infrastructure
      |
      +--> implements Abstractions
```

---

# Testing

The architecture makes testing easier.

## Domain Tests

Test business rules:

```text
OrderTests
├── Cancel_WhenPending_ShouldCancel
├── Cancel_WhenShipped_ShouldFail
└── AddItem_WithInvalidQuantity_ShouldFail
```

These tests should not require a database.

---

## Application Tests

Test handlers:

```text
CreateOrderCommandHandlerTests
GetOrderQueryHandlerTests
CancelOrderCommandHandlerTests
```

Dependencies can be mocked/faked:

```text
IOrderRepository
IOrderQueries
ICacheService
IEmailSender
```

---

## Integration Tests

Test:

```text
Application
    +
Infrastructure
    +
Database
```

For example:

```text
CreateOrder
    -> EF Core
    -> SQL Server
    -> verify database state
```

---

# Recommended Final Application Structure

For a medium-to-large modular monolith, this is a strong starting point:

```text
Orders.Application/
│
├── Orders/
│   │
│   ├── CreateOrder/
│   │   ├── CreateOrderCommand.cs
│   │   ├── CreateOrderCommandHandler.cs
│   │   ├── CreateOrderCommandValidator.cs
│   │   └── CreateOrderResult.cs
│   │
│   ├── CancelOrder/
│   │   ├── CancelOrderCommand.cs
│   │   ├── CancelOrderCommandHandler.cs
│   │   └── CancelOrderCommandValidator.cs
│   │
│   ├── UpdateOrder/
│   │   ├── UpdateOrderCommand.cs
│   │   ├── UpdateOrderCommandHandler.cs
│   │   └── UpdateOrderCommandValidator.cs
│   │
│   ├── GetOrder/
│   │   ├── GetOrderQuery.cs
│   │   ├── GetOrderQueryHandler.cs
│   │   └── OrderDto.cs
│   │
│   └── GetOrders/
│       ├── GetOrdersQuery.cs
│       ├── GetOrdersQueryHandler.cs
│       └── OrderListItemDto.cs
│
├── Abstractions/
│   │
│   ├── Persistence/
│   │   ├── IOrderRepository.cs
│   │   └── IOrderQueries.cs
│   │
│   ├── Services/
│   │   ├── IEmailSender.cs
│   │   └── ICurrentUser.cs
│   │
│   └── Caching/
│       └── ICacheService.cs
│
├── Behaviors/
│   ├── ValidationBehavior.cs
│   ├── LoggingBehavior.cs
│   ├── TransactionBehavior.cs
│   └── PerformanceBehavior.cs
│
├── Exceptions/
│   ├── NotFoundException.cs
│   └── ValidationException.cs
│
└── DependencyInjection.cs
```

---

# A Larger Modular Monolith

At the solution level:

```text
src/
│
├── Modules/
│   │
│   ├── Orders/
│   │   ├── Orders.Domain/
│   │   ├── Orders.Application/
│   │   ├── Orders.Infrastructure/
│   │   └── Orders.Presentation/
│   │
│   ├── Inventory/
│   │   ├── Inventory.Domain/
│   │   ├── Inventory.Application/
│   │   ├── Inventory.Infrastructure/
│   │   └── Inventory.Presentation/
│   │
│   ├── Customers/
│   │   ├── Customers.Domain/
│   │   ├── Customers.Application/
│   │   ├── Customers.Infrastructure/
│   │   └── Customers.Presentation/
│   │
│   └── Payments/
│       ├── Payments.Domain/
│       ├── Payments.Application/
│       ├── Payments.Infrastructure/
│       └── Payments.Presentation/
│
├── BuildingBlocks/
│   ├── Domain/
│   ├── Application/
│   └── Infrastructure/
│
└── Host/
    └── Web/
```

Inside every Application project:

```text
{Module}.Application/
│
├── {Feature}/
│   ├── CommandOrQuery.cs
│   ├── Handler.cs
│   ├── Validator.cs
│   └── DTO.cs
│
├── Abstractions/
├── Behaviors/
└── DependencyInjection.cs
```

This provides two levels of organization:

```text
Module
   |
   +--> Feature
           |
           +--> Command/Query
                   |
                   +--> Handler
                   +--> Validator
                   +--> DTO
```

---

# Practical Rules

A simple set of rules can keep the architecture healthy.

### Rule 1

> One feature should have one obvious location.

If you need to modify `CreateOrder`, you should mostly work inside:

```text
Orders/CreateOrder/
```

---

### Rule 2

> Commands change state.

Examples:

```text
CreateOrderCommand
CancelOrderCommand
ApproveOrderCommand
```

---

### Rule 3

> Queries read state.

Examples:

```text
GetOrderQuery
SearchOrdersQuery
GetOrderSummaryQuery
```

---

### Rule 4

> Handlers orchestrate; Domain objects enforce business rules.

Handler:

```csharp
order.Cancel();
```

Domain:

```csharp
public void Cancel()
{
    // Business rules
}
```

---

### Rule 5

> Application defines abstractions; Infrastructure implements them.

```text
Application
    |
    +--> IOrderRepository
             ^
             |
       Infrastructure
```

---

### Rule 6

> Keep controllers/endpoints thin.

Prefer:

```csharp
await sender.Send(command);
```

instead of implementing the use case in the controller.

---

### Rule 7

> Do not create abstractions without a reason.

Clean Architecture is about dependency direction and boundaries, not creating interfaces for every class.

---

### Rule 8

> Keep module boundaries stronger than folder boundaries.

This:

```text
Orders.Application
```

should not casually access:

```text
Inventory.Infrastructure
```

just because both projects are in the same solution.

---

# Recommended Mental Model

The easiest way to think about the architecture is:

```text
                 USER / API / UI
                       |
                       v
              +------------------+
              |    Application   |
              |                  |
              | Command / Query  |
              |       Handler    |
              +--------+---------+
                       |
                       v
              +------------------+
              |      Domain      |
              |                  |
              | Business Rules   |
              | Aggregates       |
              +--------+---------+
                       |
                       v
              +------------------+
              |  Infrastructure  |
              |                  |
              | DB / Redis / API |
              +------------------+
```

With CQRS:

```text
                Application
                     |
             +-------+-------+
             |               |
          Command          Query
             |               |
          Handler          Handler
             |               |
          Domain       Read abstraction
             |               |
        Repository       DB projection
```

---

# Final Recommendation

For a **C# .NET modular monolith using Clean Architecture + CQRS + MediatR**, the recommended approach is:

```text
Module
│
├── Domain
│
├── Application
│   │
│   ├── Feature A
│   │   ├── Command
│   │   ├── Handler
│   │   └── Validator
│   │
│   ├── Feature B
│   │   ├── Query
│   │   ├── Handler
│   │   └── DTO
│   │
│   ├── Abstractions
│   ├── Behaviors
│   └── DependencyInjection
│
├── Infrastructure
│
└── Presentation
```

The key architectural principle is:

> **Organize the Application Layer around use cases/features, not around technical types.**

This gives you a codebase where related code stays together, MediatR naturally represents each use case, Domain objects protect business rules, and Infrastructure remains replaceable.

For a growing modular monolith, **feature-based/vertical-slice organization inside each module** is usually more maintainable than a global `Commands`, `Queries`, `Handlers`, and `DTOs` structure.
