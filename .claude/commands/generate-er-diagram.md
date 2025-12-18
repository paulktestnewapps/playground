---
allowed-tools: Read, Glob, Grep, Write
argument-hint: [output-format]
description: Generate ER diagram from entity definitions
---

## Context

Scan the codebase for:
- Entity classes
- Relationship configurations
- DbContext definitions

## Task

Generate an ER diagram in format: $ARGUMENTS (default: mermaid)

Supported formats:
- mermaid (default) - outputs .md file with mermaid diagram
- plantuml - outputs .puml file
- dbml - outputs .dbml file

Include:
- All entities with their properties
- Primary keys and foreign keys
- Relationship cardinality (1:1, 1:N, N:M)
- Index annotations
