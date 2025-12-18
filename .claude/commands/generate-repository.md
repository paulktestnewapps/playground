---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: [entity-name]
description: Generate repository interface and implementation
---

## Context

Identify:
- ORM in use (EF Core, Dapper, Prisma)
- Existing repository patterns
- Unit of Work implementation if present

## Task

Generate repository for entity: $ARGUMENTS

1. Create IRepository interface with CRUD operations
2. Generate concrete implementation
3. Add custom query methods based on entity relationships
4. Register in DI container
5. Include Unit of Work pattern if used in project

Standard operations:
- GetById, GetAll, Find (with predicate)
- Add, AddRange
- Update, UpdateRange
- Delete, DeleteRange
- SaveChanges (if not using UoW)
