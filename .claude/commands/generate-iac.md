---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: [tool] [cloud] (e.g., bicep azure, terraform aws)
description: Generate Infrastructure as Code modules
---

## Context

Analyze:
- Application architecture
- Resource requirements
- Existing IaC if present

## Task

Generate IaC using $1 for $2:

1. **Compute Resources**
   - App Service / ECS / Cloud Run
   - Container orchestration if needed

2. **Data Resources**
   - Database (SQL, NoSQL)
   - Cache (Redis)
   - Storage accounts

3. **Networking**
   - Virtual networks
   - Load balancers
   - DNS configuration

4. **Security**
   - Key Vault / Secrets Manager
   - Managed identities
   - Network security groups

Output:
- Modular, reusable components
- Variable definitions
- Output values for cross-module references
- Environment-specific overrides
