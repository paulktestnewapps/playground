---
name: dto-mapper
description: Create DTOs and AutoMapper profiles for object-object mapping. Use when defining request/response shapes, creating mapping configurations, or implementing DTO projections.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# DTO Mapper

## Purpose

Creates data transfer objects and AutoMapper profiles following project conventions.

## When to Use

- Defining API request/response shapes
- Creating AutoMapper profiles
- Implementing DTO projections in EF Core
- Setting up mapping configurations

## Project-Specific Conventions

1. **Use AutoMapper** for object-object mapping
2. **Use DTO projections** to reduce data transfer
3. **Only include fields needed** for the projection
4. **PATCH operations** should use expression trees (see Repository Adapter skill)

## DTO Structure

### Request DTOs

```csharp
using System.ComponentModel.DataAnnotations;

namespace MyApi.Contracts.Requests;

/// <summary>
/// Request to create a new customer
/// </summary>
public record CreateCustomerRequest(
    [Required]
    [StringLength(100, MinimumLength = 1)]
    string Name,

    [Required]
    [EmailAddress]
    string Email,

    [Phone]
    string? PhoneNumber,

    [StringLength(500)]
    string? Notes
);

/// <summary>
/// Request to fully update a customer
/// </summary>
public record UpdateCustomerRequest(
    [Required]
    [StringLength(100, MinimumLength = 1)]
    string Name,

    [Required]
    [EmailAddress]
    string Email,

    [Phone]
    string? PhoneNumber,

    [StringLength(500)]
    string? Notes
);

/// <summary>
/// Request for partial customer update (all fields optional)
/// </summary>
public record PatchCustomerRequest
{
    [StringLength(100, MinimumLength = 1)]
    public string? Name { get; init; }

    [EmailAddress]
    public string? Email { get; init; }

    [Phone]
    public string? PhoneNumber { get; init; }

    [StringLength(500)]
    public string? Notes { get; init; }
}

/// <summary>
/// Filter request for customer queries
/// </summary>
public record CustomerFilterRequest(
    string? SearchTerm,
    DateTime? CreatedAfter,
    DateTime? CreatedBefore,
    int Page = 1,
    int PageSize = 20
);
```

### Response DTOs

```csharp
namespace MyApi.Contracts.Responses;

/// <summary>
/// Detailed customer response (single item)
/// </summary>
public record CustomerDetailResponse(
    Guid Id,
    string Name,
    string Email,
    string? PhoneNumber,
    string? Notes,
    DateTime CreatedAt,
    DateTime? UpdatedAt
);

/// <summary>
/// Summary customer response (list items)
/// </summary>
public record CustomerSummaryResponse(
    Guid Id,
    string Name,
    string Email
);

/// <summary>
/// Paginated list response
/// </summary>
public record PagedResponse<T>(
    IEnumerable<T> Items,
    int TotalCount,
    int Page,
    int PageSize,
    int TotalPages
);
```

## AutoMapper Profiles

### Basic Profile

```csharp
using AutoMapper;

namespace MyApi.Mapping;

public class CustomerMappingProfile : Profile
{
    public CustomerMappingProfile()
    {
        // Entity to Response DTOs
        CreateMap<Customer, CustomerDetailResponse>();
        CreateMap<Customer, CustomerSummaryResponse>();

        // Request to Entity
        CreateMap<CreateCustomerRequest, Customer>()
            .ForMember(dest => dest.Id, opt => opt.MapFrom(_ => Guid.NewGuid()))
            .ForMember(dest => dest.CreatedAt, opt => opt.MapFrom(_ => DateTime.UtcNow))
            .ForMember(dest => dest.UpdatedAt, opt => opt.Ignore());

        CreateMap<UpdateCustomerRequest, Customer>()
            .ForMember(dest => dest.Id, opt => opt.Ignore())
            .ForMember(dest => dest.CreatedAt, opt => opt.Ignore())
            .ForMember(dest => dest.UpdatedAt, opt => opt.MapFrom(_ => DateTime.UtcNow));
    }
}
```

### Profile with Custom Mappings

```csharp
using AutoMapper;

namespace MyApi.Mapping;

public class OrderMappingProfile : Profile
{
    public OrderMappingProfile()
    {
        CreateMap<Order, OrderDetailResponse>()
            .ForMember(dest => dest.CustomerName,
                opt => opt.MapFrom(src => src.Customer.Name))
            .ForMember(dest => dest.TotalAmount,
                opt => opt.MapFrom(src => src.Items.Sum(i => i.Quantity * i.UnitPrice)))
            .ForMember(dest => dest.ItemCount,
                opt => opt.MapFrom(src => src.Items.Count));

        CreateMap<Order, OrderSummaryResponse>()
            .ForMember(dest => dest.CustomerName,
                opt => opt.MapFrom(src => src.Customer.Name));

        CreateMap<CreateOrderRequest, Order>()
            .ForMember(dest => dest.Id, opt => opt.MapFrom(_ => Guid.NewGuid()))
            .ForMember(dest => dest.CreatedAt, opt => opt.MapFrom(_ => DateTime.UtcNow))
            .ForMember(dest => dest.Status, opt => opt.MapFrom(_ => OrderStatus.Pending))
            .ForMember(dest => dest.Items, opt => opt.Ignore()); // Map separately
    }
}
```

## DTO Projection in EF Core (Preferred)

**IMPORTANT**: Prefer EF Core projections over AutoMapper for database queries to avoid N+1 issues:

```csharp
// PREFERRED: Direct projection in repository
public async Task<CustomerDetailResponse?> GetByIdAsync(Guid id, CancellationToken ct)
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

// PREFERRED: List projection
public async Task<List<CustomerSummaryResponse>> GetAllAsync(CancellationToken ct)
{
    return await context.Customers
        .Select(c => new CustomerSummaryResponse(
            c.Id,
            c.Name,
            c.Email
        ))
        .ToListAsync(ct);
}
```

### AutoMapper ProjectTo (Alternative)

```csharp
// Using AutoMapper's ProjectTo for complex mappings
public async Task<OrderDetailResponse?> GetByIdAsync(Guid id, CancellationToken ct)
{
    return await context.Orders
        .Where(o => o.Id == id)
        .ProjectTo<OrderDetailResponse>(_mapper.ConfigurationProvider)
        .FirstOrDefaultAsync(ct);
}
```

## Registration

```csharp
// In Program.cs
builder.Services.AddAutoMapper(typeof(Program).Assembly);

// Or explicit registration
builder.Services.AddAutoMapper(config =>
{
    config.AddProfile<CustomerMappingProfile>();
    config.AddProfile<OrderMappingProfile>();
});
```

## Service Layer Usage

```csharp
namespace MyApi.Services;

public class CustomerService(
    ICustomerRepository repository,
    IMapper mapper) : ICustomerService
{
    public async Task CreateAsync(CreateCustomerRequest request, CancellationToken ct)
    {
        // Check for duplicates
        if (await repository.ExistsByEmailAsync(request.Email, ct))
        {
            throw new AlreadyExistsException($"Customer with email '{request.Email}' already exists.");
        }

        var customer = mapper.Map<Customer>(request);
        await repository.AddAsync(customer, ct);
    }

    public async Task UpdateAsync(Guid id, UpdateCustomerRequest request, CancellationToken ct)
    {
        var customer = await GetEntityByIdAsync(id, ct);
        mapper.Map(request, customer);
        await repository.UpdateAsync(customer, ct);
    }
}
```

## Best Practices

1. **Prefer EF Core projection** over AutoMapper for queries
2. **Use records** for immutable DTOs
3. **Include validation attributes** on request DTOs
4. **Create separate DTOs** for Create, Update, Patch, and Response
5. **Only map needed fields** - avoid over-fetching
6. **Use `ProjectTo`** for complex AutoMapper scenarios with EF Core
