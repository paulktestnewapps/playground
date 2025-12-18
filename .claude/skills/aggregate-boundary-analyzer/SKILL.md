---
name: aggregate-boundary-analyzer
description: Identify DDD aggregate boundaries and roots based on transactional consistency requirements. Use when designing domain models or determining entity relationships.
allowed-tools: Read, Glob, Grep
---

# Aggregate Boundary Analyzer

## Purpose

Identifies DDD aggregate boundaries based on transactional consistency requirements.

## When to Use

- Designing domain models
- Determining entity groupings
- Identifying aggregate roots
- Analyzing invariant enforcement

## Inputs

- Domain entities
- Business rules/invariants
- Transaction requirements
- Consistency needs

## Outputs

- Aggregate root identification
- Boundary recommendations
- Invariant documentation
- Relationship guidance

## Aggregate Concepts

### Aggregate Root
The entry point entity that:
- Guards all access to the aggregate
- Ensures invariants are maintained
- Has global identity
- Controls lifecycle of child entities

### Aggregate Boundaries
Define what must be consistent within a single transaction:
- Changes within aggregate = immediate consistency
- Changes across aggregates = eventual consistency

## Analysis Process

### 1. Identify Invariants
Business rules that must always be true:

```
Order Invariants:
- Order total must equal sum of line items
- Cannot have negative quantities
- Status transitions must be valid (Draft → Submitted → Shipped)

Inventory Invariants:
- Stock cannot go negative
- Reserved + Available = Total
```

### 2. Group by Transaction Boundary
Entities that must change together atomically:

```
✓ Order + OrderItems (same aggregate)
  - Adding item must update total
  - Must be consistent immediately

✗ Order + Inventory (separate aggregates)
  - Order placement reduces inventory
  - Can be eventually consistent
```

### 3. Identify Aggregate Roots

**Good Aggregate Roots:**
- Order (contains OrderItems, OrderStatus)
- Customer (contains Addresses, Preferences)
- Product (contains ProductVariants, Pricing)

**Not Aggregate Roots:**
- OrderItem (belongs to Order)
- Address (belongs to Customer)
- LineItem (belongs to Invoice)

## Boundary Patterns

### Small Aggregates (Preferred)
```
┌─────────────────┐     ┌─────────────────┐
│     Order       │     │    Customer     │
│  ┌───────────┐  │     │  ┌───────────┐  │
│  │ OrderItem │  │     │  │  Address  │  │
│  │ OrderItem │  │     │  │  Address  │  │
│  └───────────┘  │     │  └───────────┘  │
└─────────────────┘     └─────────────────┘
        │                       │
        └───── CustomerId ──────┘
            (reference by ID)
```

### Reference by Identity
Aggregates reference each other by ID, not object reference:

```csharp
// Good - reference by ID
public class Order
{
    public Guid Id { get; }
    public Guid CustomerId { get; }  // Reference to Customer aggregate
    public List<OrderItem> Items { get; }  // Owned by this aggregate
}

// Bad - direct reference across aggregates
public class Order
{
    public Customer Customer { get; }  // Don't do this
}
```

## Output Format

```yaml
aggregate: Order
root_entity: Order
child_entities:
  - OrderItem
  - OrderStatus
  - ShippingDetails

invariants:
  - name: Valid Total
    rule: Total = Sum(Items.Price * Items.Quantity)
    enforcement: Recalculate on item change

  - name: Status Transition
    rule: Status changes follow state machine
    enforcement: Guard in SetStatus method

boundaries:
  internal:
    - OrderItem (cascade delete, same transaction)
    - ShippingDetails (1:1, same lifecycle)
  external:
    - Customer (reference by CustomerId)
    - Product (reference by ProductId in OrderItem)
    - Payment (separate aggregate, eventual consistency)

recommendations:
  - Keep Order aggregate small
  - Use domain events for cross-aggregate coordination
  - OrderItem should not be queryable outside Order context
```

## Anti-Patterns to Avoid

1. **Giant Aggregates**: Including too many entities
2. **Anemic Roots**: Roots without behavior
3. **Cross-Aggregate Transactions**: Trying to update multiple aggregates atomically
4. **Ignoring Invariants**: Not protecting business rules
