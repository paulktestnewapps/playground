---
allowed-tools: Read, Glob, Grep
argument-hint: [path-to-openapi-spec]
description: Extract entity list summary from API specification
---

## Context

Parse the OpenAPI specification at: $ARGUMENTS

## Task

Provide a quick summary of entities derived from the API specification. This is a lightweight analysis - use `/analyze-api-spec` for full DDL generation.

### 1. Identify Entities
From paths, list all entities:
- `/users` → User
- `/users/{id}/orders` → Order (belongs to User)
- `/orders/{orderId}/items` → OrderItem (belongs to Order)
- `/products` → Product

### 2. Output Format

```
Entities Identified:
--------------------
1. User
   - Source: /users, /users/{id}
   - Operations: GET (list), GET (single), POST, PUT, DELETE

2. Order
   - Source: /orders, /users/{id}/orders
   - Operations: GET (list), GET (single), POST
   - Relationships: belongs_to User

3. OrderItem
   - Source: /orders/{id}/items
   - Operations: GET (list), POST, DELETE
   - Relationships: belongs_to Order

4. Product
   - Source: /products
   - Operations: GET (list), GET (single)

Relationships:
--------------
- User 1:N Order
- Order 1:N OrderItem
- OrderItem N:1 Product

Next Steps:
-----------
Run `/analyze-api-spec [path]` to generate full PostgreSQL DDL
```

### Notes
- This command provides a quick overview only
- No DDL or C# code is generated
- Use for initial analysis before full schema generation
