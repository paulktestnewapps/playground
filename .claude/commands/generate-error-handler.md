---
allowed-tools: Read, Edit, Write, Glob, Grep
description: Generate global error handling middleware
---

## Context

Analyze:
- Exception types in codebase
- API style (REST, GraphQL)
- Existing error handling patterns

## Task

Generate comprehensive error handling:

1. Global exception handler middleware
2. Problem Details responses (RFC 7807)
3. Error code catalog/enum
4. Logging integration
5. Environment-specific error details (dev vs prod)

Map common exceptions:
- ValidationException → 400
- NotFoundException → 404
- UnauthorizedException → 401
- ForbiddenException → 403
- ConflictException → 409
