---
name: transaction-complexity-analyzer
description: Evaluate transaction scope and recommend Saga vs ACID patterns. Use when operations span multiple aggregates or services and you need to choose a consistency strategy.
allowed-tools: Read, Glob, Grep
---

# Transaction Complexity Analyzer

## Purpose

Evaluates whether operations need distributed transactions or simpler patterns.

## When to Use

- Operations affecting multiple aggregates
- Cross-service coordination
- Choosing between ACID and eventual consistency
- Designing compensation strategies

## Inputs

- Operation description
- Affected entities/services
- Consistency requirements
- Failure scenarios

## Outputs

- Transaction scope assessment
- Saga vs ACID recommendation
- Compensation strategy (if saga)
- Implementation guidance

## Transaction Patterns

### 1. Local ACID Transaction
Single database, immediate consistency.

**Use When:**
- Single aggregate
- Single database
- Strong consistency required
- Simple operations

```csharp
await using var transaction = await _context.Database.BeginTransactionAsync();
try
{
    _context.Orders.Add(order);
    _context.OrderItems.AddRange(items);
    await _context.SaveChangesAsync();
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

### 2. Saga - Choreography
Event-driven, decentralized coordination.

**Use When:**
- Loose coupling preferred
- Simple workflows
- Services can react independently
- No central coordinator needed

```
Order Created → [Inventory Service] → Stock Reserved
                                    ↓
Stock Reserved → [Payment Service] → Payment Processed
                                    ↓
Payment Processed → [Shipping Service] → Shipment Created
```

### 3. Saga - Orchestration
Central coordinator manages the flow.

**Use When:**
- Complex workflows
- Need visibility into progress
- Conditional branching
- Centralized error handling

```csharp
public class OrderSagaOrchestrator
{
    public async Task<OrderResult> ExecuteAsync(CreateOrderCommand command)
    {
        var sagaId = Guid.NewGuid();

        // Step 1: Reserve Inventory
        var inventoryResult = await _inventoryService.ReserveAsync(command.Items);
        if (!inventoryResult.Success)
            return OrderResult.Failed("Insufficient inventory");

        // Step 2: Process Payment
        var paymentResult = await _paymentService.ProcessAsync(command.Payment);
        if (!paymentResult.Success)
        {
            // Compensate Step 1
            await _inventoryService.ReleaseReservationAsync(inventoryResult.ReservationId);
            return OrderResult.Failed("Payment failed");
        }

        // Step 3: Create Order
        var order = await _orderService.CreateAsync(command);

        // Step 4: Schedule Shipping
        await _shippingService.ScheduleAsync(order.Id);

        return OrderResult.Success(order);
    }
}
```

## Decision Matrix

| Factor | ACID | Choreography | Orchestration |
|--------|------|--------------|---------------|
| Scope | Single DB | Multi-service | Multi-service |
| Consistency | Immediate | Eventual | Eventual |
| Coupling | N/A | Loose | Medium |
| Visibility | Full | Distributed | Centralized |
| Complexity | Low | Medium | High |
| Failure Handling | Automatic | Per-service | Centralized |

## Complexity Assessment

### Low Complexity (ACID suitable)
- Single aggregate modification
- Single database
- No external service calls
- Score: 1-3

### Medium Complexity (Consider Saga)
- Multiple aggregates
- 2-3 services involved
- Some operations can be async
- Score: 4-6

### High Complexity (Saga required)
- Multiple services/databases
- Long-running operations
- Complex compensation logic
- Score: 7-10

## Compensation Strategies

### Semantic Compensation
Undo the business effect, not the technical change:

```yaml
operation: Reserve Inventory
compensation: Release Reservation

operation: Charge Payment
compensation: Refund Payment

operation: Create Shipment
compensation: Cancel Shipment
```

### Pivot Transaction
Point of no return after which only forward recovery:

```
Reserve → Charge → [PIVOT] → Ship → Deliver
          ↓ can compensate    ↓ must complete
```

## Output Format

```yaml
operation: Place Order
affected_services:
  - Order Service (primary)
  - Inventory Service
  - Payment Service
  - Notification Service

assessment:
  complexity_score: 7
  recommended_pattern: Orchestrated Saga
  consistency_model: Eventual

rationale: |
  - Spans 4 services
  - Payment requires compensation on failure
  - Need centralized visibility for support

saga_steps:
  - step: 1
    service: Inventory
    action: Reserve Stock
    compensation: Release Reservation
    timeout: 5s

  - step: 2
    service: Payment
    action: Process Payment
    compensation: Refund
    timeout: 30s
    pivot: true  # Point of no return

  - step: 3
    service: Order
    action: Confirm Order
    compensation: null  # After pivot

  - step: 4
    service: Notification
    action: Send Confirmation
    compensation: null  # Fire and forget

failure_scenarios:
  - scenario: Inventory unavailable
    action: Reject order immediately

  - scenario: Payment fails
    action: Release inventory, notify customer

  - scenario: Post-pivot failure
    action: Retry with backoff, alert ops
```
