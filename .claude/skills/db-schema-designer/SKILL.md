---
name: db-schema-designer
description: Generate database schema artifacts including ORM code, entities, and SQL scripts. Use when working with database design, entity creation, migrations, or ER diagrams.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# DB Schema Designer

## Purpose

Generates database schema artifacts from API specifications and domain models.

## Required References

Before generating schemas, consult:
- [Database Design Guide](docs/Guides/database_design_guide.md) - Naming, audit columns, normalization
- [Primary Key Decision Guide](docs/Guides/pk_type_decision_guide.md) - UUID vs integer criteria

## When to Use

- Creating new entities/tables
- Designing database relationships
- Generating ORM configurations (DbContext, entities)
- Creating migration scripts
- Producing ER diagrams

## Inputs

- API specifications (OpenAPI, Swagger)
- Controller code with endpoint definitions
- Domain model descriptions
- Existing entity code (for modifications)

## Outputs

- Entity classes with ORM annotations
- DbContext configuration
- Migration scripts (SQL)
- ER diagrams (Mermaid, PlantUML, DBML)

## Process

1. **Analyze Requirements**
   - Parse API spec or controller to identify data needs
   - Identify entities and their properties
   - Determine relationships (1:1, 1:N, N:M)

2. **Design Schema**
   - Choose primary key type per [PK Decision Guide](docs/Guides/pk_type_decision_guide.md):
     - Default to incremental integers
     - Use UUIDs only for API-exposed sensitive data, distributed systems, or cross-DB sync
     - Use hybrid approach (int PK + UUID column) when needed
   - Establish foreign key relationships
   - Add required audit columns: `CreatedOn`, `CreatedBy`, `UpdatedOn`, `UpdatedBy`
   - Add `IsActive` for tables with DELETE operations
   - Define indexes for query optimization

3. **Generate Artifacts**
   - Create entity classes following project conventions
   - Configure DbContext with Fluent API
   - Generate migration scripts

## Conventions

### Entity Naming
- Singular PascalCase for class names (User, Order, Product)
- Plural for DbSet properties (Users, Orders, Products)
- Id suffix for foreign keys (UserId, OrderId)

### Base Entity Pattern
```csharp
// For tables with integer PKs (default)
public abstract class BaseEntity
{
    public int Id { get; set; }
    public DateTime CreatedOn { get; set; }
    public string CreatedBy { get; set; } = null!;
    public DateTime? UpdatedOn { get; set; }
    public string? UpdatedBy { get; set; }
}

// For tables with UUID PKs (API-exposed sensitive data)
public abstract class BaseEntityWithGuid
{
    public Guid Id { get; set; }
    public DateTime CreatedOn { get; set; }
    public string CreatedBy { get; set; } = null!;
    public DateTime? UpdatedOn { get; set; }
    public string? UpdatedBy { get; set; }
}

// For tables with soft delete
public abstract class SoftDeleteEntity : BaseEntity
{
    public bool IsActive { get; set; } = true;
}
```

### Relationship Configuration
```csharp
// One-to-Many
builder.HasMany(o => o.OrderItems)
    .WithOne(i => i.Order)
    .HasForeignKey(i => i.OrderId);

// Many-to-Many (EF Core 5+)
builder.HasMany(p => p.Categories)
    .WithMany(c => c.Products);
```

## Examples

### Input: API Endpoint
```
POST /api/orders
{
  "customerId": "guid",
  "items": [{ "productId": "guid", "quantity": 1 }]
}
```

### Output: Entities
```csharp
// Order uses GUID PK because it's API-exposed with potentially sensitive data
public class Order : BaseEntityWithGuid
{
    public Guid CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
    public ICollection<OrderItem> Items { get; set; } = new List<OrderItem>();
    public int StatusId { get; set; }
    public OrderStatus Status { get; set; } = null!;
    public bool IsActive { get; set; } = true;
}

public class OrderItem : BaseEntity  // Integer PK - internal only
{
    public Guid OrderId { get; set; }
    public Order Order { get; set; } = null!;
    public int ProductId { get; set; }
    public Product Product { get; set; } = null!;
    public int Quantity { get; set; }
}

// Lookup table - always uses integer PK
public class OrderStatus
{
    public int Id { get; set; }
    public string Value { get; set; } = null!;
}
```
