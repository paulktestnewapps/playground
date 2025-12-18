---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: [endpoint-name]
description: Generate controller with routes and DI setup
---

## Context

Analyze:
- Existing controllers for patterns
- OpenAPI spec if available
- Target framework (ASP.NET, Express, FastAPI)

## Task

Generate controller for endpoint: $ARGUMENTS

1. Create controller class with appropriate base class
2. Add route definitions following REST conventions
3. Generate request/response models
4. Configure dependency injection
5. Add XML documentation comments

Follow the architectural pattern established in the codebase (or ask if unclear).
