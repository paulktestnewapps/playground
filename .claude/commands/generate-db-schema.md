---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: [entity-name]
description: Generate database schema artifacts (ORM code, entities)
---

## Required References

Before generating, read these project guides:
- [Database Design Guide](docs/Guides/database_design_guide.md)
- [Primary Key Decision Guide](docs/Guides/pk_type_decision_guide.md)

## Context

Analyze the existing codebase for:
- API specifications
- Controller code
- Existing entities

## Task

Generate database schema artifacts for $ARGUMENTS:

1. Create entity classes with proper annotations
2. Generate DbContext configuration
3. Define relationships and constraints
4. Add indexes for common query patterns

## Conventions to Follow

### Primary Keys
- **Default**: Use incremental integers for most tables
- **Use UUIDs only when**: Exposed via API with sensitive data, distributed systems, or cross-DB sync
- **Lookup tables**: Always use integer PKs (e.g., OrderStatus, ResourceType)

### Required Audit Columns
Every entity must include:
- `CreatedOn` (DateTime, UTC, not-null)
- `CreatedBy` (string, not-null)
- `UpdatedOn` (DateTime?, UTC, nullable)
- `UpdatedBy` (string?, nullable)

### Soft Deletes
- Add `IsActive` (bool, not-null, default true) for tables with DELETE operations

### Normalization
- Use join tables for many-to-many relationships
- Create lookup tables for status/type values
- Avoid comma-separated values in columns

Follow the project's existing ORM conventions. If no ORM is detected, ask which to use (EF Core, Dapper, Prisma, etc.).
