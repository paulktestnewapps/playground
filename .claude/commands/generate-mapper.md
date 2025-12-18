---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: [source-type] [target-type]
description: Generate mapping logic between types
---

## Context

Identify:
- Mapping library in use (AutoMapper, Mapster, manual)
- Existing mapping profiles
- Source and target type definitions

## Task

Generate mapper from $1 to $2

1. Create mapping profile/configuration
2. Handle nested object mappings
3. Configure any custom value resolvers needed
4. Add reverse mapping if applicable

If no mapping library is detected, generate extension methods for manual mapping.
