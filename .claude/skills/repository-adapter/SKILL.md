---
name: repository-adapter
description: Generate repository layer for data access following the Service-Repository Hybrid pattern. Use when implementing data access, creating query extensions, or setting up EF Core repositories.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# Repository Adapter

## Purpose

Generates data access layer following the Service-Repository Hybrid pattern with Entity Framework Core.

## When to Use

- Creating data access abstractions
- Implementing CRUD operations
- Setting up query extension methods
- Building the repository layer

## Project-Specific Pattern: Service-Repository Hybrid

```
Controller → Service → Repository → DbContext
```

### Key Principles

1. **Repository interacts directly with DbContext**
2. **Contains all EF Core queries**
3. **Injected into service layer via DI**
4. **Not unit tested directly** (mocked in service tests)
5. **Use query extension methods** for common patterns

## Repository Implementation

### Base Repository (Optional for complex projects)

```csharp
namespace MyApi.Data.Repositories;

public abstract class BaseRepository<TEntity>(AppDbContext context)
    where TEntity : class
{
    protected readonly AppDbContext Context = context;
    protected readonly DbSet<TEntity> DbSet = context.Set<TEntity>();

    public virtual async Task<TEntity?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await DbSet.FindAsync([id], ct);

    public virtual async Task<List<TEntity>> GetAllAsync(CancellationToken ct = default)
        => await DbSet.ToListAsync(ct);

    public virtual async Task AddAsync(TEntity entity, CancellationToken ct = default)
        => await DbSet.AddAsync(entity, ct);

    public virtual void Update(TEntity entity)
        => DbSet.Update(entity);

    public virtual void Remove(TEntity entity)
        => DbSet.Remove(entity);

    public virtual async Task<bool> ExistsAsync(Guid id, CancellationToken ct = default)
        => await DbSet.FindAsync([id], ct) is not null;

    public virtual async Task SaveChangesAsync(CancellationToken ct = default)
        => await Context.SaveChangesAsync(ct);
}
```

### Entity-Specific Repository

```csharp
namespace MyApi.Data.Repositories;

public interface ICustomerRepository
{
    Task<CustomerDetailResponse?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<List<CustomerSummaryResponse>> GetAllAsync(CustomerFilterRequest? filter, CancellationToken ct = default);
    Task<bool> ExistsByEmailAsync(string email, CancellationToken ct = default);
    Task AddAsync(Customer customer, CancellationToken ct = default);
    Task UpdateAsync(Customer customer, CancellationToken ct = default);
    Task DeleteAsync(Guid id, CancellationToken ct = default);
}

public class CustomerRepository(AppDbContext context) : ICustomerRepository
{
    /// <summary>
    /// Get customer by ID with projection to DTO
    /// </summary>
    public async Task<CustomerDetailResponse?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        return await context.Customers
            .Where(c => c.Id == id)
            .Select(c => new CustomerDetailResponse(
                c.Id,
                c.Name,
                c.Email,
                c.PhoneNumber,
                c.Notes,
                c.CreatedAt,
                c.UpdatedAt
            ))
            .FirstOrDefaultAsync(ct);
    }

    /// <summary>
    /// Get all customers with optional filtering and projection
    /// </summary>
    public async Task<List<CustomerSummaryResponse>> GetAllAsync(
        CustomerFilterRequest? filter,
        CancellationToken ct = default)
    {
        return await context.Customers
            .ApplyFilter(filter)
            .Select(c => new CustomerSummaryResponse(
                c.Id,
                c.Name,
                c.Email
            ))
            .ToListAsync(ct);
    }

    public async Task<bool> ExistsByEmailAsync(string email, CancellationToken ct = default)
    {
        return await context.Customers
            .AnyAsync(c => c.Email == email, ct);
    }

    public async Task AddAsync(Customer customer, CancellationToken ct = default)
    {
        await context.Customers.AddAsync(customer, ct);
        await context.SaveChangesAsync(ct);
    }

    public async Task UpdateAsync(Customer customer, CancellationToken ct = default)
    {
        context.Customers.Update(customer);
        await context.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(Guid id, CancellationToken ct = default)
    {
        var customer = await context.Customers.FindAsync([id], ct)
            ?? throw new NotFoundException($"Customer with ID '{id}' was not found.");

        context.Customers.Remove(customer);
        await context.SaveChangesAsync(ct);
    }
}
```

## Query Extension Methods

Create reusable query patterns:

```csharp
namespace MyApi.Data.Extensions;

public static class CustomerQueryExtensions
{
    /// <summary>
    /// Apply filters to customer query
    /// </summary>
    public static IQueryable<Customer> ApplyFilter(
        this IQueryable<Customer> query,
        CustomerFilterRequest? filter)
    {
        if (filter is null) return query;

        if (!string.IsNullOrWhiteSpace(filter.SearchTerm))
        {
            var term = filter.SearchTerm.ToLower();
            query = query.Where(c =>
                c.Name.ToLower().Contains(term) ||
                c.Email.ToLower().Contains(term));
        }

        if (filter.CreatedAfter.HasValue)
        {
            query = query.Where(c => c.CreatedAt >= filter.CreatedAfter.Value);
        }

        if (filter.CreatedBefore.HasValue)
        {
            query = query.Where(c => c.CreatedAt <= filter.CreatedBefore.Value);
        }

        return query;
    }

    /// <summary>
    /// Apply sorting to customer query
    /// </summary>
    public static IQueryable<Customer> ApplySort(
        this IQueryable<Customer> query,
        string? sortBy,
        bool descending = false)
    {
        return sortBy?.ToLower() switch
        {
            "name" => descending
                ? query.OrderByDescending(c => c.Name)
                : query.OrderBy(c => c.Name),
            "email" => descending
                ? query.OrderByDescending(c => c.Email)
                : query.OrderBy(c => c.Email),
            "created" => descending
                ? query.OrderByDescending(c => c.CreatedAt)
                : query.OrderBy(c => c.CreatedAt),
            _ => query.OrderBy(c => c.Name)
        };
    }

    /// <summary>
    /// Apply pagination to customer query
    /// </summary>
    public static IQueryable<Customer> ApplyPagination(
        this IQueryable<Customer> query,
        int page,
        int pageSize)
    {
        return query
            .Skip((page - 1) * pageSize)
            .Take(pageSize);
    }
}
```

## PATCH Operations with Expression Trees

```csharp
namespace MyApi.Data.Extensions;

public static class PatchExtensions
{
    /// <summary>
    /// Apply partial updates using expression trees for type safety
    /// </summary>
    public static void ApplyPatch<TEntity>(
        this TEntity entity,
        params (bool HasValue, Action<TEntity> Apply)[] updates)
    {
        foreach (var (hasValue, apply) in updates)
        {
            if (hasValue)
            {
                apply(entity);
            }
        }
    }
}

// Usage in repository or service:
public async Task PatchAsync(Guid id, PatchCustomerRequest request, CancellationToken ct)
{
    var customer = await context.Customers.FindAsync([id], ct)
        ?? throw new NotFoundException($"Customer with ID '{id}' was not found.");

    customer.ApplyPatch(
        (request.Name is not null, c => c.Name = request.Name!),
        (request.Email is not null, c => c.Email = request.Email!),
        (request.PhoneNumber is not null, c => c.PhoneNumber = request.PhoneNumber),
        (request.Notes is not null, c => c.Notes = request.Notes)
    );

    customer.UpdatedAt = DateTime.UtcNow;
    await context.SaveChangesAsync(ct);
}
```

## DI Registration

```csharp
// In Program.cs or ServiceCollectionExtensions
services.AddScoped<ICustomerRepository, CustomerRepository>();
services.AddScoped<IOrderRepository, OrderRepository>();
// Add more repositories as needed
```

## Best Practices

1. **Always project to DTOs** - Never return entities from repository
2. **Include only needed fields** in projections
3. **Use query extensions** for reusable filter/sort/pagination logic
4. **Primary constructors** for all repository classes
5. **Async throughout** with CancellationToken support
6. **Let EF Core track changes** - Don't manually manage state
