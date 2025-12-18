---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: [platform] (azure-pipelines|github-actions|gitlab-ci)
description: Generate CI/CD pipeline configuration
---

## Context

Analyze:
- Project type and structure
- Build requirements
- Test frameworks
- Deployment targets

## Task

Generate CI/CD pipeline for: $ARGUMENTS

1. **Build Stage**
   - Restore dependencies
   - Compile/build
   - Run linters

2. **Test Stage**
   - Unit tests
   - Integration tests
   - Code coverage

3. **Package Stage**
   - Create artifacts
   - Container build (if applicable)

4. **Deploy Stages**
   - Development
   - Staging
   - Production (with approval gates)

Include:
- Environment variables
- Secret references
- Caching strategies
- Parallel job optimization
