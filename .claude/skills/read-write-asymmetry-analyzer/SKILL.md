---
name: read-write-asymmetry-analyzer
description: Determine if CQRS or read/write separation would benefit an endpoint. Use when analyzing query patterns, performance optimization, or considering event sourcing.
allowed-tools: Read, Glob, Grep
---

# Read/Write Asymmetry Analyzer

## Purpose

Determines if read/write separation (CQRS) would benefit the endpoint.

## When to Use

- Analyzing query vs command patterns
- Performance optimization decisions
- Considering event sourcing
- Scaling read vs write workloads independently

## Inputs

- Endpoint usage patterns
- Query complexity
- Read/write ratio
- Performance requirements

## Outputs

- CQRS recommendation score
- Read model suggestions
- Event sourcing applicability
- Implementation guidance

## CQRS Concepts

### Command Query Responsibility Segregation
Separate models for reading and writing data:

```
┌─────────────┐     ┌─────────────┐
│   Command   │     │    Query    │
│   Model     │     │    Model    │
│ (Write DB)  │     │  (Read DB)  │
└──────┬──────┘     └──────┬──────┘
       │                    │
       │    ┌────────┐     │
       └────│ Events │─────┘
            └────────┘
```

### When CQRS Helps

1. **Different Read/Write Patterns**
   - Writes: Complex validation, business rules
   - Reads: Simple projections, denormalized views

2. **Scaling Asymmetry**
   - 100:1 read to write ratio
   - Read replicas needed
   - Different caching strategies

3. **Complex Queries**
   - Reporting/analytics
   - Search with many filters
   - Aggregations across entities

## Analysis Factors

### Read/Write Ratio

| Ratio | Recommendation |
|-------|----------------|
| < 10:1 | Probably not needed |
| 10:1 - 100:1 | Consider for hot paths |
| > 100:1 | Strong candidate |

### Query Complexity

| Type | Single Model | CQRS |
|------|--------------|------|
| Simple by ID | ✓ | |
| Filtered list | ✓ | |
| Multi-join | | ✓ |
| Aggregations | | ✓ |
| Full-text search | | ✓ |
| Real-time dashboard | | ✓ |

### Write Complexity

| Type | Single Model | CQRS |
|------|--------------|------|
| Simple CRUD | ✓ | |
| Validation rules | ✓ | |
| Complex invariants | | ✓ |
| Audit trail needed | | ✓ |
| Event sourcing | | ✓ |

## Implementation Patterns

### Simple CQRS (Same Database)
```csharp
// Command side - rich domain model
public class Order
{
    public void AddItem(Product product, int quantity)
    {
        // Business logic, validation
        Items.Add(new OrderItem(product, quantity));
        RecalculateTotal();
    }
}

// Query side - optimized read model
public class OrderSummaryQuery
{
    public async Task<OrderSummaryDto> GetAsync(Guid orderId)
    {
        return await _context.Orders
            .Where(o => o.Id == orderId)
            .Select(o => new OrderSummaryDto
            {
                Id = o.Id,
                CustomerName = o.Customer.Name,
                Total = o.Total,
                ItemCount = o.Items.Count
            })
            .FirstOrDefaultAsync();
    }
}
```

### Full CQRS (Separate Databases)
```csharp
// Write side
public class OrderCommandHandler
{
    public async Task Handle(CreateOrderCommand command)
    {
        var order = Order.Create(command);
        await _writeDb.Orders.AddAsync(order);
        await _eventBus.PublishAsync(new OrderCreatedEvent(order));
    }
}

// Read side projection
public class OrderReadModelProjection
{
    public async Task Handle(OrderCreatedEvent @event)
    {
        var readModel = new OrderReadModel
        {
            Id = @event.OrderId,
            CustomerName = @event.CustomerName,
            Total = @event.Total,
            Status = "Created"
        };
        await _readDb.Orders.InsertAsync(readModel);
    }
}
```

## Event Sourcing Applicability

### Good Fit
- Audit requirements (financial, compliance)
- Complex domain with temporal queries
- Need to replay/rebuild state
- Event-driven architecture

### Poor Fit
- Simple CRUD applications
- Real-time consistency required
- Simple queries dominate
- Team unfamiliar with pattern

## Output Format

```yaml
endpoint: GET /api/orders/dashboard
current_pattern: Single Model

analysis:
  read_write_ratio: 50:1
  query_complexity: High
  write_complexity: Medium

  read_characteristics:
    - Aggregates from multiple tables
    - Needs filtering by 5+ fields
    - Real-time updates preferred

  write_characteristics:
    - Standard order creation
    - Moderate validation
    - No complex invariants

recommendation:
  cqrs_score: 7.5/10
  suggested_approach: Simple CQRS with read model

  rationale: |
    High read/write ratio and complex dashboard queries
    justify separate read model. Write complexity doesn't
    warrant full event sourcing.

read_model_design:
  type: Materialized view or separate table
  fields:
    - orderId
    - customerName
    - orderDate
    - total
    - itemCount
    - status
  update_strategy: Sync on write (same transaction)

event_sourcing:
  recommended: false
  reason: Write complexity doesn't justify overhead
```

## Decision Tree

```
Is read/write ratio > 10:1?
├─ No → Single model probably fine
└─ Yes → Are queries complex (joins, aggregations)?
         ├─ No → Consider caching
         └─ Yes → Is audit trail critical?
                  ├─ Yes → Event Sourcing + CQRS
                  └─ No → Simple CQRS with read model
```
