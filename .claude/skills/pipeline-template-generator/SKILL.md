---
name: pipeline-template-generator
description: Generate CI/CD pipeline configurations for Azure Pipelines, GitHub Actions, or GitLab CI. Use when setting up build, test, and deployment automation.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# Pipeline Template Generator

## Purpose

Generates CI/CD pipeline configurations based on project structure and requirements.

## When to Use

- Setting up new CI/CD pipelines
- Migrating between CI/CD platforms
- Adding deployment stages
- Optimizing build performance

## Inputs

- Project type (API, library, container)
- CI/CD platform (Azure Pipelines, GitHub Actions, GitLab CI)
- Branch strategy
- Deployment targets

## Outputs

- Pipeline YAML configuration
- Stage definitions (build, test, deploy)
- Environment gates
- Variable templates

## Platform Templates

### Azure Pipelines
```yaml
trigger:
  branches:
    include:
      - main
      - release/*
  paths:
    exclude:
      - '**/*.md'
      - docs/*

pr:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: common-variables
  - name: buildConfiguration
    value: 'Release'

stages:
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: BuildJob
        steps:
          - task: UseDotNet@2
            inputs:
              version: '8.x'

          - task: DotNetCoreCLI@2
            displayName: 'Restore'
            inputs:
              command: restore

          - task: DotNetCoreCLI@2
            displayName: 'Build'
            inputs:
              command: build
              arguments: '--configuration $(buildConfiguration) --no-restore'

          - task: DotNetCoreCLI@2
            displayName: 'Test'
            inputs:
              command: test
              arguments: '--configuration $(buildConfiguration) --collect:"XPlat Code Coverage"'

          - task: PublishCodeCoverageResults@1
            inputs:
              codeCoverageTool: 'Cobertura'
              summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'

  - stage: DeployDev
    displayName: 'Deploy to Development'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployDev
        environment: 'development'
        strategy:
          runOnce:
            deploy:
              steps:
                - template: templates/deploy-steps.yml
                  parameters:
                    environment: 'dev'
```

### GitHub Actions
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, 'release/*']
    paths-ignore:
      - '**/*.md'
      - 'docs/**'
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '8.0.x'
  BUILD_CONFIGURATION: Release

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration ${{ env.BUILD_CONFIGURATION }} --no-restore

      - name: Test
        run: dotnet test --configuration ${{ env.BUILD_CONFIGURATION }} --no-build --collect:"XPlat Code Coverage"

      - name: Upload coverage
        uses: codecov/codecov-action@v4

  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Dev
        run: |
          # Deployment steps here
```

### GitLab CI
```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOTNET_VERSION: "8.0"

.dotnet-cache:
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - .nuget/

build:
  stage: build
  image: mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION}
  extends: .dotnet-cache
  script:
    - dotnet restore
    - dotnet build --configuration Release --no-restore
  artifacts:
    paths:
      - bin/
    expire_in: 1 hour

test:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION}
  extends: .dotnet-cache
  needs: [build]
  script:
    - dotnet test --configuration Release --no-build
  coverage: '/Total\s*\|\s*(\d+\.?\d*)%/'

deploy-dev:
  stage: deploy
  needs: [test]
  environment:
    name: development
    url: https://dev.example.com
  only:
    - main
  script:
    - echo "Deploying to development"
```

## Pipeline Components

### Build Optimization
- Caching (NuGet, npm, pip)
- Incremental builds
- Parallel jobs
- Matrix builds for multi-platform

### Testing Stages
- Unit tests
- Integration tests
- E2E tests (optional, can be separate pipeline)
- Code coverage reporting

### Security Scanning
- SAST (Static Application Security Testing)
- Dependency scanning
- Container scanning
- Secret detection

### Deployment Strategies
- Blue-green deployment
- Canary releases
- Rolling updates
- Feature flags integration
