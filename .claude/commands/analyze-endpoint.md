---
allowed-tools: Read, Glob, Grep
argument-hint: [endpoint-path-or-name]
description: Analyze an endpoint for patterns and complexity
---

## Context

Locate and analyze endpoint: $ARGUMENTS

## Task

Provide comprehensive endpoint analysis:

1. **Intent Classification**
   - CRUD operation
   - Command (state change)
   - Query (read-only)
   - Workflow/orchestration
   - Saga (distributed transaction)

2. **Complexity Assessment**
   - Cyclomatic complexity
   - Dependencies count
   - Transaction scope

3. **Pattern Recommendations**
   - Current pattern in use
   - Suggested improvements
   - CQRS applicability

4. **Performance Considerations**
   - N+1 query risks
   - Caching opportunities
   - Async potential
