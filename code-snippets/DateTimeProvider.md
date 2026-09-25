# DateTimeProvider

Thin wrapper around the system clock. Application and Domain code depend on
`IDateTimeProvider` instead of calling `DateTime.UtcNow` directly, which keeps
time-dependent logic (entities, validators, handlers) deterministic and testable.

## Interface — `CleanArchitecture.Application/Abstractions/Services/IDateTimeProvider.cs`

```csharp
namespace CleanArchitecture.Application.Abstractions.Services;

/// <summary>
/// Abstraction over the system clock. Application and Domain code
/// should never call DateTime.UtcNow directly — inject this instead
/// so tests can substitute a fixed or controllable time.
/// </summary>
public interface IDateTimeProvider
{
    DateTime UtcNow { get; }

    DateOnly Today { get; }
}
```

## Implementation — `CleanArchitecture.Infrastructure/Services/DateTimeProvider.cs`

```csharp
using CleanArchitecture.Application.Abstractions.Services;

namespace CleanArchitecture.Infrastructure.Services;

/// <summary>
/// Thin wrapper around the system clock. Exists so Application code
/// depends on IDateTimeProvider instead of calling DateTime.UtcNow
/// directly, which keeps time-dependent logic (entities, validators,
/// handlers) deterministic and testable.
/// </summary>
public sealed class DateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow => DateTime.UtcNow;

    public DateOnly Today => DateOnly.FromDateTime(DateTime.UtcNow);
}
```

## Registration — `Program.cs` / `DependencyInjection.cs`

```csharp
builder.Services.AddSingleton<IDateTimeProvider, DateTimeProvider>();
```

Registered as a singleton since it's stateless.

## Test double — `CleanArchitecture.Application.UnitTests/TestUtils/FakeDateTimeProvider.cs`

```csharp
namespace CleanArchitecture.Application.UnitTests.TestUtils;

public sealed class FakeDateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow { get; set; } = new(2026, 1, 1, 0, 0, 0, DateTimeKind.Utc);

    public DateOnly Today => DateOnly.FromDateTime(UtcNow);
}
```