---
name: environment-matrix
description: Generate multi-environment configurations with proper separation for dev, test, staging, and production. Use when setting up environment-specific settings, variable groups, or promotion gates.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# Environment Matrix

## Purpose

Generates multi-environment configuration with proper separation.

## When to Use

- Setting up dev/test/staging/prod environments
- Defining environment-specific configurations
- Creating promotion workflows
- Managing feature flags per environment

## Inputs

- Environment list (dev, test, staging, prod)
- Configuration differences per environment
- Resource sizing requirements
- Promotion criteria

## Outputs

- Environment-specific configs
- Variable group definitions
- Promotion gates
- Environment comparison matrix

## Environment Configuration

### Configuration Matrix
```yaml
environments:
  dev:
    purpose: Development and debugging
    data: Synthetic/mock data
    access: Developers only
    resources:
      compute: Minimal (B1, t3.micro)
      database: Basic tier, small
      replicas: 1
    features:
      debug_mode: true
      verbose_logging: true
      synthetic_data: true
    deployment:
      trigger: Automatic on main
      approval: None

  test:
    purpose: Integration and QA testing
    data: Anonymized production subset
    access: Dev + QA team
    resources:
      compute: Small (S1, t3.small)
      database: Standard tier
      replicas: 1
    features:
      debug_mode: true
      verbose_logging: true
      synthetic_data: false
    deployment:
      trigger: After dev success
      approval: None

  staging:
    purpose: Pre-production validation
    data: Production mirror (anonymized)
    access: Extended team
    resources:
      compute: Production-like (P1v3, m5.large)
      database: Production tier
      replicas: 2
    features:
      debug_mode: false
      verbose_logging: false
      synthetic_data: false
    deployment:
      trigger: Manual
      approval: Tech lead

  prod:
    purpose: Production workload
    data: Real production data
    access: Operations team
    resources:
      compute: Scaled (P2v3, m5.xlarge)
      database: Production tier, HA
      replicas: 3+
    features:
      debug_mode: false
      verbose_logging: false
      synthetic_data: false
    deployment:
      trigger: Manual
      approval: Product owner + Tech lead
```

### Azure DevOps Variable Groups
```yaml
# azure-pipelines/variables/dev.yml
variables:
  - name: Environment
    value: dev
  - name: ResourceGroup
    value: rg-myapp-dev
  - name: AppServicePlan
    value: B1
  - name: SqlSku
    value: Basic
  - name: EnableDebug
    value: true
  - name: LogLevel
    value: Debug

# azure-pipelines/variables/prod.yml
variables:
  - name: Environment
    value: prod
  - name: ResourceGroup
    value: rg-myapp-prod
  - name: AppServicePlan
    value: P2v3
  - name: SqlSku
    value: S3
  - name: EnableDebug
    value: false
  - name: LogLevel
    value: Warning
```

### GitHub Environments
```yaml
# .github/workflows/deploy.yml
jobs:
  deploy-dev:
    environment: development
    env:
      AZURE_WEBAPP_NAME: myapp-dev
    steps:
      - uses: azure/webapps-deploy@v2

  deploy-staging:
    needs: deploy-dev
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
      - uses: azure/webapps-deploy@v2

  deploy-prod:
    needs: deploy-staging
    environment:
      name: production
      url: https://myapp.com
    # Requires manual approval configured in GitHub
    steps:
      - uses: azure/webapps-deploy@v2
```

## Application Configuration

### ASP.NET Core
```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug"
    }
  },
  "Features": {
    "EnableDebugEndpoints": true,
    "UseMockServices": true
  }
}

// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "Features": {
    "EnableDebugEndpoints": false,
    "UseMockServices": false
  }
}
```

### Environment Transform
```csharp
// Program.cs
builder.Configuration
    .AddJsonFile("appsettings.json", optional: false)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true)
    .AddEnvironmentVariables()
    .AddAzureKeyVault(/* for secrets */);
```

## Promotion Gates

### Quality Gates
```yaml
promotion_gates:
  dev_to_test:
    - All unit tests pass
    - Code coverage > 80%
    - No critical security issues

  test_to_staging:
    - All integration tests pass
    - Performance baseline met
    - QA sign-off

  staging_to_prod:
    - Smoke tests pass
    - No P1 bugs
    - Change approval board
    - Deployment window confirmed
```

### Rollback Criteria
```yaml
rollback_triggers:
  automatic:
    - Error rate > 5% for 5 minutes
    - Latency p99 > 2x baseline
    - Health check failures > 3

  manual:
    - Critical bug discovered
    - Security vulnerability
    - Business decision
```

## Resource Sizing Guide

| Environment | CPU | Memory | Storage | Cost Factor |
|-------------|-----|--------|---------|-------------|
| Dev | 1 core | 1.75 GB | 10 GB | 1x |
| Test | 2 cores | 3.5 GB | 50 GB | 2x |
| Staging | 4 cores | 8 GB | 100 GB | 4x |
| Prod | 8+ cores | 16+ GB | 500+ GB | 10x+ |
