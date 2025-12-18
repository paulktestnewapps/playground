---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: [pattern-name] (hexagonal|clean|vertical-slice|cqrs)
description: Apply architectural pattern to codebase
---

## Context

Analyze current structure and target pattern: $ARGUMENTS

## Task

Restructure codebase to follow the specified pattern:

### Hexagonal (Ports & Adapters)
- Core domain (entities, services)
- Ports (interfaces)
- Adapters (implementations)

### Clean Architecture
- Domain layer
- Application layer
- Infrastructure layer
- Presentation layer

### Vertical Slice
- Feature folders
- Each slice contains all layers

### CQRS
- Command handlers
- Query handlers
- Separate read/write models

Provide migration steps and file moves required.
