---
allowed-tools: Read, Edit, Write, Glob, Grep, Bash
argument-hint: [api-spec-path]
description: Implement full API from specification
---

## Context

Load and analyze:
- OpenAPI/Swagger specification
- Existing project structure
- Framework conventions

## Task

Implement API from spec: $ARGUMENTS

This is a comprehensive command that orchestrates:
1. Generate all controllers
2. Create DTOs for all endpoints
3. Generate repository layer
4. Set up error handling
5. Configure routing
6. Add validation

After generation, provide a summary of created files and any manual steps needed.
