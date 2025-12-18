---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: [entity-name]
description: Generate database schema artifacts (ORM code, entities)
---

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

Follow the project's existing ORM conventions. If no ORM is detected, ask which to use (EF Core, Dapper, Prisma, etc.).
