---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: [entity-name]
description: Generate DTOs for an entity
---

## Context

Review:
- Domain entity definition
- API contracts
- Existing DTO patterns in codebase

## Task

Generate DTOs for entity: $ARGUMENTS

Create the following:
1. Request DTOs (Create, Update, Patch)
2. Response DTOs (Detail, Summary, List item)
3. Validation attributes
4. Nullable annotations where appropriate

Follow existing naming conventions (e.g., [Entity]Request, [Entity]Response).
