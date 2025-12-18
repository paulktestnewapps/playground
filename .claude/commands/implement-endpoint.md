---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: [resource.action] (e.g., users.create)
description: Implement a specific endpoint
---

## Context

Parse endpoint specification: $ARGUMENTS
Format: [resource].[action] (e.g., users.create, orders.list, products.update)

## Task

Implement the specified endpoint:

1. Add controller action method
2. Create request/response DTOs
3. Add service layer method
4. Implement repository method if needed
5. Add validation rules
6. Include proper HTTP status codes
7. Add OpenAPI annotations

Supported actions:
- create (POST)
- list (GET collection)
- get (GET single)
- update (PUT)
- patch (PATCH)
- delete (DELETE)
- [custom] (ask for HTTP method)
