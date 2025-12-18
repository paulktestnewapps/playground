---
allowed-tools: Read, Edit, Write, Glob, Grep
description: Regenerate API layer from updated entities
---

## Context

Detect changes in:
- Entity definitions
- DbContext configuration
- Business rules

## Task

Regenerate API layer to reflect entity changes:

1. Update DTOs to match entity changes
2. Regenerate mappers
3. Update repository interfaces/implementations
4. Modify controller actions if needed
5. Update validation rules

Preserve:
- Custom business logic in services
- Manual customizations (marked with // CUSTOM comments)
- Test files
