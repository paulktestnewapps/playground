---
allowed-tools: Read, Glob, Grep
description: Analyze codebase and recommend architectural patterns
---

## Context

Analyze the codebase for:
- Domain complexity
- Read/write patterns
- Transaction boundaries
- Team size and experience (if known)

## Task

Provide architectural pattern recommendations:

1. Assess current state
2. Identify pain points
3. Recommend patterns with rationale:
   - Hexagonal/Ports & Adapters
   - Clean Architecture
   - Vertical Slice
   - CQRS
   - Event Sourcing
   - Simple layered

For each recommendation, explain:
- Why it fits
- Migration complexity
- Trade-offs
- When to avoid it
