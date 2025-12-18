---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: --db [postgres|sqlserver|mysql|sqlite]
description: Generate SQL migration scripts for the specified database
---

## Context

Review existing:
- Entity definitions
- DbContext configurations
- Previous migrations

## Task

Generate SQL scripts for database: $ARGUMENTS

1. Create table definitions with proper types for the target database
2. Generate foreign key constraints
3. Add indexes from entity configurations
4. Create migration script with up/down operations

Output files:
- migrations/[timestamp]_[description].sql
- migrations/[timestamp]_[description]_rollback.sql
