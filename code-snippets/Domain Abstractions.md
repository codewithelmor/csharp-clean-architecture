# Domain Abstractions

Base building blocks in the Domain project — `Entity`, `AggregateRoot`,
`ValueObject`, `Result`, `Error`, plus the marker interfaces used by the
Infrastructure interceptors. These are pure domain concepts with zero
outside dependencies.

## Entity.cs — `CleanArchitecture.Domain/Abstractions/Entity.cs`

Base class for anything with identity. Equality is by ID, not by value.

```csharp
namespace CleanArchitecture.Domain.Abstractions;

public abstract class Entity : IEquatable<Entity>
{
    public Guid Id { get; protected init; }

    protected Entity(Guid id) => Id = id;

    protected Entity() { }

    public bool Equals(Entity? other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        if (GetType() != other.GetType()) return false;

        return Id == other.Id;
    }

    public override bool Equals(object? obj) => Equals(obj as Entity);

    public override int GetHashCode() => (GetType(), Id).GetHashCode();

    public static bool operator ==(Entity? left, Entity? right) =>
        left is null ? right is null : left.Equals(right);

    public static bool operator !=(Entity? left, Entity? right) => !(left == right);
}
```

## AggregateRoot.cs — `CleanArchitecture.Domain/Abstractions/AggregateRoot.cs`

An `Entity` that is also a consistency boundary and the source of domain
events. Only aggregate roots are loaded directly via repositories.

```csharp
namespace CleanArchitecture.Domain.Abstractions;

public abstract class AggregateRoot : Entity
{
    private readonly List<IDomainEvent> _domainEvents = new();

    protected AggregateRoot(Guid id) : base(id) { }

    protected AggregateRoot() { }

    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void RaiseDomainEvent(IDomainEvent domainEvent) =>
        _domainEvents.Add(domainEvent);

    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

## IAuditableEntity.cs — `CleanArchitecture.Domain/Abstractions/IAuditableEntity.cs`

Opted into by any entity that should be stamped with created/modified
metadata. Picked up by `AuditableEntityInterceptor`.

```csharp
namespace CleanArchitecture.Domain.Abstractions;

public interface IAuditableEntity
{
    DateTime CreatedAt { get; set; }
    string? CreatedBy { get; set; }
    DateTime? LastModifiedAt { get; set; }
    string? LastModifiedBy { get; set; }
}
```

## ISoftDeletable.cs — `CleanArchitecture.Domain/Abstractions/ISoftDeletable.cs`

Opted into by any entity that should be soft-deleted rather than physically
removed. Picked up by `SoftDeleteInterceptor` and by the global EF Core
query filter on each entity's configuration.

```csharp
namespace CleanArchitecture.Domain.Abstractions;

public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
}
```

## IDomainEvent.cs — `CleanArchitecture.Domain/Abstractions/IDomainEvent.cs`

Marker interface for anything raised via `AggregateRoot.RaiseDomainEvent`.
Extends `INotification` so it can be published through a mediator without
Domain referencing the mediator package directly — swap this base if you're
using a hand-rolled dispatcher instead.

```csharp
using MediatR;

namespace CleanArchitecture.Domain.Abstractions;

public interface IDomainEvent : INotification
{
    Guid EventId => Guid.NewGuid();
    DateTime OccurredOnUtc => DateTime.UtcNow;
}
```

## ValueObject.cs — `CleanArchitecture.Domain/Abstractions/ValueObject.cs`

Base class for immutable, equality-by-value types like `Money` and `Address`.

```csharp
namespace CleanArchitecture.Domain.Abstractions;

public abstract class ValueObject : IEquatable<ValueObject>
{
    protected abstract IEnumerable<object?> GetEqualityComponents();

    public bool Equals(ValueObject? other)
    {
        if (other is null || GetType() != other.GetType())
        {
            return false;
        }

        return GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    }

    public override bool Equals(object? obj) => Equals(obj as ValueObject);

    public override int GetHashCode() =>
        GetEqualityComponents()
            .Aggregate(17, (hash, component) => HashCode.Combine(hash, component));

    public static bool operator ==(ValueObject? left, ValueObject? right) =>
        left is null ? right is null : left.Equals(right);

    public static bool operator !=(ValueObject? left, ValueObject? right) => !(left == right);
}
```

## Error.cs — `CleanArchitecture.Domain/Abstractions/Error.cs`

A structured, comparable error type used with `Result` instead of throwing
exceptions for expected failure cases (validation, not-found, conflict).

```csharp
namespace CleanArchitecture.Domain.Abstractions;

public sealed record Error(string Code, string Message, ErrorType Type = ErrorType.Failure)
{
    public static readonly Error None = new(string.Empty, string.Empty);

    public static Error NotFound(string code, string message) =>
        new(code, message, ErrorType.NotFound);

    public static Error Validation(string code, string message) =>
        new(code, message, ErrorType.Validation);

    public static Error Conflict(string code, string message) =>
        new(code, message, ErrorType.Conflict);
}

public enum ErrorType
{
    Failure,
    Validation,
    NotFound,
    Conflict
}
```

## Result.cs — `CleanArchitecture.Domain/Abstractions/Result.cs`

Represents success/failure without exceptions for control flow. Used as the
return type from domain and application service methods.

```csharp
namespace CleanArchitecture.Domain.Abstractions;

public class Result
{
    protected Result(bool isSuccess, Error error)
    {
        if (isSuccess && error != Error.None)
        {
            throw new InvalidOperationException("A successful result cannot contain an error.");
        }

        if (!isSuccess && error == Error.None)
        {
            throw new InvalidOperationException("A failed result must contain an error.");
        }

        IsSuccess = isSuccess;
        Error = error;
    }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }

    public static Result Success() => new(true, Error.None);
    public static Result Failure(Error error) => new(false, error);

    public static Result<TValue> Success<TValue>(TValue value) => new(value, true, Error.None);
    public static Result<TValue> Failure<TValue>(Error error) => new(default, false, error);
}

public class Result<TValue> : Result
{
    private readonly TValue? _value;

    protected internal Result(TValue? value, bool isSuccess, Error error)
        : base(isSuccess, error)
    {
        _value = value;
    }

    public TValue Value => IsSuccess
        ? _value!
        : throw new InvalidOperationException("The value of a failed result cannot be accessed.");

    public static implicit operator Result<TValue>(TValue value) => Success(value);
}
```