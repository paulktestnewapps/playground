---
name: endpoint-intent-classifier
description: Classify API endpoints by intent (CRUD, Command, Query, Workflow, Saga) and recommend patterns. Use when analyzing endpoints for architectural decisions or pattern selection.
allowed-tools: Read, Glob, Grep
---

# Endpoint Intent Classifier

## Purpose

Analyzes endpoint requirements to classify intent and suggest appropriate patterns.

## When to Use

- Analyzing new endpoint requirements
- Determining architectural approach
- Supporting Design Pattern Decider agent
- Assessing complexity before implementation

## Inputs

- Endpoint description or user story
- Existing endpoints (for context)
- Business requirements
- Expected usage patterns

## Outputs

- Intent classification
- Recommended patterns
- Complexity score (1-10)
- Implementation guidance

## Intent Categories

### 1. CRUD Operations
Simple create, read, update, delete operations on a single resource.

**Characteristics:**
- Single entity focus
- Direct database mapping
- No complex business rules
- Idempotent reads, non-idempotent writes

**Pattern:** Simple Repository + Controller

**Examples:**
- `GET /users/{id}` - Read user
- `POST /products` - Create product
- `DELETE /orders/{id}` - Delete order

### 2. Command (State Change)
Operations that change system state with business logic.

**Characteristics:**
- Validates business rules
- May affect multiple entities
- Side effects (notifications, events)
- Non-idempotent

**Pattern:** Command Handler / Service Layer

**Examples:**
- `POST /orders/{id}/submit` - Submit order for processing
- `POST /accounts/{id}/transfer` - Transfer funds
- `POST /users/{id}/deactivate` - Deactivate with cleanup

### 3. Query (Read-Only)
Complex data retrieval operations.

**Characteristics:**
- Read-only, no side effects
- May aggregate multiple sources
- Often involves filtering, sorting, pagination
- Cacheable

**Pattern:** Query Handler / CQRS Read Side

**Examples:**
- `GET /reports/sales-summary` - Aggregated report
- `GET /dashboard/metrics` - Multi-source dashboard
- `GET /search?q=term` - Complex search

### 4. Workflow (Multi-Step Process)
Operations involving multiple steps with potential compensation.

**Characteristics:**
- Multiple sequential operations
- Partial completion possible
- May require rollback
- Long-running

**Pattern:** Workflow/Orchestration Pattern

**Examples:**
- `POST /checkout` - Multi-step checkout process
- `POST /onboarding/complete` - User onboarding flow
- `POST /import/process` - Data import pipeline

### 5. Saga (Distributed Transaction)
Operations spanning multiple services/bounded contexts.

**Characteristics:**
- Cross-service coordination
- Eventual consistency
- Compensation on failure
- Event-driven

**Pattern:** Saga Pattern (Choreography or Orchestration)

**Examples:**
- `POST /orders` - Order spanning inventory, payment, shipping
- `POST /bookings` - Hotel + flight + car rental
- `POST /transfers/international` - Multi-bank transfer

## Classification Matrix

| Factor | CRUD | Command | Query | Workflow | Saga |
|--------|------|---------|-------|----------|------|
| Entities Affected | 1 | 1-3 | 0 (read) | 3+ | Multiple services |
| Business Rules | Minimal | Moderate | None | Complex | Complex |
| Transaction Scope | Single | Single | None | Local | Distributed |
| Idempotent | Reads only | No | Yes | No | Compensatable |
| Cacheable | Yes | No | Yes | No | No |

## Complexity Scoring

**Score 1-3 (Low):** CRUD operations, simple queries
- Direct database operations
- Minimal validation
- Single responsibility

**Score 4-6 (Medium):** Commands, complex queries
- Business rule validation
- Multiple entity involvement
- Moderate error handling

**Score 7-10 (High):** Workflows, Sagas
- Multi-step processes
- Distributed coordination
- Complex failure handling
- Compensation logic

## Output Format

```yaml
endpoint: POST /orders/{id}/submit
intent: Command
complexity: 5
confidence: 0.85
rationale: |
  - Changes order state (pending → submitted)
  - Validates inventory availability
  - Triggers notification side effects
  - Single bounded context
recommended_patterns:
  primary: Command Handler with MediatR
  alternatives:
    - Service Layer with explicit transaction
considerations:
  - Add idempotency key for retry safety
  - Consider async processing for notifications
```
