---
name: db-schema-designer
description: Generate database schema artifacts including ORM code, entities, and SQL scripts. Use when working with database design, entity creation, migrations, or ER diagrams.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# DB Schema Designer

## Purpose

Generates database schema artifacts from API specifications and domain models.

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
   - Define primary keys (prefer GUIDs for distributed systems)
   - Establish foreign key relationships
   - Add audit fields (CreatedAt, UpdatedAt, CreatedBy)
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

### Common Patterns
```csharp
public class BaseEntity
{
    public Guid Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
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
public class Order : BaseEntity
{
    public Guid CustomerId { get; set; }
    public Customer Customer { get; set; }
    public ICollection<OrderItem> Items { get; set; }
    public OrderStatus Status { get; set; }
}

public class OrderItem
{
    public Guid Id { get; set; }
    public Guid OrderId { get; set; }
    public Order Order { get; set; }
    public Guid ProductId { get; set; }
    public Product Product { get; set; }
    public int Quantity { get; set; }
}
```
