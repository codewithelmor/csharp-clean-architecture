# Domain Value Objects

Immutable, equality-by-value types that encode business invariants. Each
derives from `ValueObject` (see `Domain/Abstractions/ValueObject.cs`) so
equality compares components rather than reference identity, and each
enforces its own construction rules via factory methods rather than public
constructors.

## Money.cs — `CleanArchitecture.Domain/ValueObjects/Money.cs`

Pairs an amount with a currency so values from different currencies can
never be silently mixed. Arithmetic operators enforce matching currencies.

```csharp
using CleanArchitecture.Domain.Abstractions;
using CleanArchitecture.Domain.Errors;

namespace CleanArchitecture.Domain.ValueObjects;

public sealed class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    private Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    public static Result<Money> Create(decimal amount, string currency)
    {
        if (amount < 0)
        {
            return Result.Failure<Money>(MoneyErrors.NegativeAmount);
        }

        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
        {
            return Result.Failure<Money>(MoneyErrors.InvalidCurrency);
        }

        return new Money(amount, currency.ToUpperInvariant());
    }

    public static Money Zero(string currency) => new(0m, currency.ToUpperInvariant());

    public static Money operator +(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        return new Money(left.Amount + right.Amount, left.Currency);
    }

    public static Money operator -(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        return new Money(left.Amount - right.Amount, left.Currency);
    }

    private static void EnsureSameCurrency(Money left, Money right)
    {
        if (left.Currency != right.Currency)
        {
            throw new InvalidOperationException(
                $"Cannot combine amounts in different currencies: {left.Currency} and {right.Currency}.");
        }
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }

    public override string ToString() => $"{Amount:0.00} {Currency}";
}
```

## Address.cs — `CleanArchitecture.Domain/ValueObjects/Address.cs`

Groups the parts of a postal address together so they're always created,
validated, and compared as a unit rather than as loose strings on an entity.

```csharp
using CleanArchitecture.Domain.Abstractions;
using CleanArchitecture.Domain.Errors;

namespace CleanArchitecture.Domain.ValueObjects;

public sealed class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string PostalCode { get; }
    public string Country { get; }

    private Address(string street, string city, string postalCode, string country)
    {
        Street = street;
        City = city;
        PostalCode = postalCode;
        Country = country;
    }

    public static Result<Address> Create(string street, string city, string postalCode, string country)
    {
        if (string.IsNullOrWhiteSpace(street) ||
            string.IsNullOrWhiteSpace(city) ||
            string.IsNullOrWhiteSpace(postalCode) ||
            string.IsNullOrWhiteSpace(country))
        {
            return Result.Failure<Address>(AddressErrors.Incomplete);
        }

        return new Address(street.Trim(), city.Trim(), postalCode.Trim(), country.Trim());
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Street;
        yield return City;
        yield return PostalCode;
        yield return Country;
    }

    public override string ToString() => $"{Street}, {City} {PostalCode}, {Country}";
}
```

## Email.cs — `CleanArchitecture.Domain/ValueObjects/Email.cs`

Wraps a validated email string so "is this a well-formed email" is checked
once, at construction, instead of scattered across the codebase.

```csharp
using System.Text.RegularExpressions;
using CleanArchitecture.Domain.Abstractions;
using CleanArchitecture.Domain.Errors;

namespace CleanArchitecture.Domain.ValueObjects;

public sealed partial class Email : ValueObject
{
    public string Value { get; }

    private Email(string value) => Value = value;

    public static Result<Email> Create(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
        {
            return Result.Failure<Email>(EmailErrors.Empty);
        }

        var normalized = email.Trim().ToLowerInvariant();

        if (!EmailRegex().IsMatch(normalized))
        {
            return Result.Failure<Email>(EmailErrors.InvalidFormat);
        }

        return new Email(normalized);
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }

    public override string ToString() => Value;

    [GeneratedRegex(@"^[^@\s]+@[^@\s]+\.[^@\s]+$")]
    private static partial Regex EmailRegex();
}
```

> `Money`, `Address`, and `Email` all return `Result<T>` from `Create` rather
> than throwing, keeping invalid-input handling consistent with the rest of
> the Domain's `Result`/`Error` pattern. They rely on corresponding
> `MoneyErrors`, `AddressErrors`, and `EmailErrors` static classes under
> `Domain/Errors/`, following the same convention as `OrderErrors` and
> `CustomerErrors`.