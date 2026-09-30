# .NET Entity to DTO Mapping Practices

In modern .NET development, the industry trend has moved sharply away from heavy, reflection-based NuGet mappers like AutoMapper due to performance overhead, debugging complexity ("magic" mapping bugs), and the introduction of advanced C# language features. 

The latest industry practice for mapping entities to DTOs without NuGet dependencies relies on explicit, compile-time type transformations using native C# features.

---

## Core Mapping Strategies

### 1. Static Extension Methods (The Dominant Standard)
Writing **static extension methods** is currently the most widely adopted practice for standard in-memory object transformations. 

* **Why:** It decouples the mapping logic from both the Entity and the DTO. It provides a clean, discoverable syntax (`entity.ToDto()`) and supports zero-allocation optimizations.

```csharp
// The Entity (Database Model)
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public DateTime CreatedAt { get; set; }
}

// The DTO (Data Contract) using C# Records
public record ProductResponse(int Id, string Name, decimal Price);

// The Mapper Class
public static class ProductMappingExtensions
{
    public static ProductResponse ToDto(this Product product)
    {
        return new ProductResponse(
            product.Id,
            product.Name,
            product.Price
        );
    }
}
```

**Usage:**
```csharp
ProductResponse dto = product.ToDto();
```

---

### 2. LINQ Projection via `Select` (The Database Standard)
When querying data using **Entity Framework Core (EF Core)**, the industry standard is projecting directly into the DTO inside the LINQ query using `Select`.

* **Why:** EF Core translates this projection directly into the generated SQL query. It only retrieves the columns requested by the DTO from the database (avoiding `SELECT *`), heavily optimizing I/O and database performance.

```csharp
public async Task<List<ProductResponse>> GetActiveProductsAsync(AppDbContext context)
{
    return await context.Products
        .AsNoTracking()
        .Where(p => p.Price > 0)
        .Select(p => new ProductResponse(p.Id, p.Name, p.Price)) // Native LINQ Projection
        .ToListAsync();
}
```

---

### 3. Target-Typed New Expressions
For mapping standard incoming requests (e.g., a `CreateProductRequest` DTO back into a domain `Product` entity), using C# **Target-Typed New Expressions** keeps code clean and readable:

```csharp
public record CreateProductRequest(string Name, decimal Price);

public static class ProductMappingExtensions
{
    public static Product ToEntity(this CreateProductRequest request) => new()
    {
        Name = request.Name,
        Price = request.Price,
        CreatedAt = DateTime.UtcNow // Set system properties during mapping
    };
}
```

---

## Architectural Comparison

| Approach | Best Used For | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **Extension Methods** | General memory mapping, business layer formatting | Fast execution, easy to debug, compile-time safe | Writing boilerplate code |
| **LINQ Projections** | Database read operations (`GET` endpoints) | Ultra-fast database performance, pulls only needed columns | Logic is bound to query structures |
| **Implicit Operators** | Highly repetitive, strict 1:1 model relationships | Extremely clean syntax (`ProductResponse dto = product;`) | Obfuscates code execution flow; hard to scale |

---

## Why the Industry Ditched NuGet Mappers

* **Compilation Over Reflection:** Modern C# focuses heavily on Native AOT (Ahead-of-Time compilation). Traditional third-party mappers rely on runtime reflection, which breaks AOT compatibility and balloons memory usage.
* **Maintainability:** While writing extension methods requires manual boilerplate, "magic" runtime mappers make debugging difficult when tracking down why a nested field mapped incorrectly or failed silently.
* **C# Evolution:** Language features like `records`, positional constructors, and target-typed `new()` expressions have made manual object instantiation incredibly brief, reducing the historical need for auto-mapping utilities.