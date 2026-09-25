# EF Core Save Interceptors

Save-time hooks registered on the `DbContext` that handle cross-cutting
persistence concerns (auditing, soft-delete, domain event dispatch) without
polluting entity or repository code. Each implements `SaveChangesInterceptor`
and is chained together in `ApplicationDbContext` via `AddInterceptors(...)`.

## AuditableEntityInterceptor — `CleanArchitecture.Infrastructure/Persistence/Interceptors/AuditableEntityInterceptor.cs`

Stamps `CreatedAt`/`CreatedBy` and `LastModifiedAt`/`LastModifiedBy` on any
entity implementing `IAuditableEntity`, just before changes are saved.

```csharp
using CleanArchitecture.Application.Abstractions.Services;
using CleanArchitecture.Domain.Abstractions;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.ChangeTracking;
using Microsoft.EntityFrameworkCore.Diagnostics;

namespace CleanArchitecture.Infrastructure.Persistence.Interceptors;

public sealed class AuditableEntityInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUserService _currentUserService;
    private readonly IDateTimeProvider _dateTimeProvider;

    public AuditableEntityInterceptor(
        ICurrentUserService currentUserService,
        IDateTimeProvider dateTimeProvider)
    {
        _currentUserService = currentUserService;
        _dateTimeProvider = dateTimeProvider;
    }

    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData, InterceptionResult<int> result)
    {
        UpdateAuditableEntities(eventData.Context);
        return base.SavingChanges(eventData, result);
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        UpdateAuditableEntities(eventData.Context);
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }

    private void UpdateAuditableEntities(DbContext? context)
    {
        if (context is null)
        {
            return;
        }

        var utcNow = _dateTimeProvider.UtcNow;
        var userId = _currentUserService.UserId;

        foreach (var entry in context.ChangeTracker.Entries<IAuditableEntity>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = utcNow;
                entry.Entity.CreatedBy = userId;
            }

            if (entry.State is EntityState.Added or EntityState.Modified || entry.HasChangedOwnedEntities())
            {
                entry.Entity.LastModifiedAt = utcNow;
                entry.Entity.LastModifiedBy = userId;
            }
        }
    }
}

file static class EntityEntryExtensions
{
    public static bool HasChangedOwnedEntities(this EntityEntry entry) =>
        entry.References.Any(r =>
            r.TargetEntry is { Metadata.IsOwned: true } &&
            r.TargetEntry.State is EntityState.Added or EntityState.Modified);
}
```

## SoftDeleteInterceptor — `CleanArchitecture.Infrastructure/Persistence/Interceptors/SoftDeleteInterceptor.cs`

Converts a hard delete on any `ISoftDeletable` entity into an update, setting
`IsDeleted`/`DeletedAt` instead of removing the row.

```csharp
using CleanArchitecture.Application.Abstractions.Services;
using CleanArchitecture.Domain.Abstractions;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Diagnostics;

namespace CleanArchitecture.Infrastructure.Persistence.Interceptors;

public sealed class SoftDeleteInterceptor : SaveChangesInterceptor
{
    private readonly IDateTimeProvider _dateTimeProvider;

    public SoftDeleteInterceptor(IDateTimeProvider dateTimeProvider)
    {
        _dateTimeProvider = dateTimeProvider;
    }

    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData, InterceptionResult<int> result)
    {
        ApplySoftDeletes(eventData.Context);
        return base.SavingChanges(eventData, result);
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        ApplySoftDeletes(eventData.Context);
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }

    private void ApplySoftDeletes(DbContext? context)
    {
        if (context is null)
        {
            return;
        }

        foreach (var entry in context.ChangeTracker.Entries<ISoftDeletable>())
        {
            if (entry.State != EntityState.Deleted)
            {
                continue;
            }

            entry.State = EntityState.Modified;
            entry.Entity.IsDeleted = true;
            entry.Entity.DeletedAt = _dateTimeProvider.UtcNow;
        }
    }
}
```

> Pair this with a global EF Core query filter (`HasQueryFilter(e => !e.IsDeleted)`
> in each `IEntityTypeConfiguration<T>`) so soft-deleted rows are excluded from
> reads automatically.

## DispatchDomainEventsInterceptor — `CleanArchitecture.Infrastructure/Persistence/Interceptors/DispatchDomainEventsInterceptor.cs`

Collects domain events raised on tracked aggregates and publishes them
**after** `SaveChangesAsync` succeeds, so handlers only ever react to events
that actually persisted.

```csharp
using CleanArchitecture.Domain.Abstractions;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.ChangeTracking;
using Microsoft.EntityFrameworkCore.Diagnostics;

namespace CleanArchitecture.Infrastructure.Persistence.Interceptors;

public sealed class DispatchDomainEventsInterceptor : SaveChangesInterceptor
{
    private readonly IPublisher _publisher;

    public DispatchDomainEventsInterceptor(IPublisher publisher)
    {
        _publisher = publisher;
    }

    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken cancellationToken = default)
    {
        await DispatchDomainEvents(eventData.Context);
        return await base.SavedChangesAsync(eventData, result, cancellationToken);
    }

    private async Task DispatchDomainEvents(DbContext? context)
    {
        if (context is null)
        {
            return;
        }

        var entities = context.ChangeTracker
            .Entries<AggregateRoot>()
            .Where(e => e.Entity.DomainEvents.Count > 0)
            .Select(e => e.Entity)
            .ToList();

        var domainEvents = entities
            .SelectMany(e => e.DomainEvents)
            .ToList();

        entities.ForEach(e => e.ClearDomainEvents());

        foreach (var domainEvent in domainEvents)
        {
            await _publisher.Publish(domainEvent);
        }
    }
}
```

> `IPublisher` here stands in for whichever dispatch mechanism you use — a
> mediator (MediatR/Mediator), or a hand-rolled `IDomainEventDispatcher` if
> you've moved off a mediator library elsewhere in the app.

## Registration — `ApplicationDbContext.cs`

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.AddInterceptors(
        _auditableEntityInterceptor,
        _softDeleteInterceptor,
        _dispatchDomainEventsInterceptor);
}
```

Or, if constructing the `DbContext` via `AddDbContext`, register the
interceptors as services and pull them in through `optionsBuilder.AddInterceptors(...)`
inside the `UseSqlServer`/`UseNpgsql` configuration delegate so they're
resolved from DI rather than newed up directly.