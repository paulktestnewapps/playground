---
name: iac-module-generator
description: Generate Infrastructure as Code using Bicep for Azure deployments. Use when provisioning Azure resources, setting up App Services, SQL databases, or API Management.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# IaC Module Generator

## Purpose

Generates Infrastructure as Code modules using Bicep for Azure deployments.

## When to Use

- Provisioning Azure infrastructure
- Setting up App Service, SQL Database, Storage
- Configuring Azure Container Apps
- Publishing APIs to API Management
- Setting up monitoring (Application Insights, Log Analytics)

## Project-Specific Conventions

### Primary Tool: Bicep

Use **Bicep** for all Azure IaC deployments (not Terraform).

### IaC Use Cases

1. Publication of APIs to API Management
2. Management of Azure resources (App Service, SQL Database, Storage)
3. Azure Container App configuration (port, image, env variables)
4. Managing environment variables (Azure App Configuration)
5. Setting up monitoring (Application Insights, Log Analytics)

## Bicep Module Structure

```
infra/
├── main.bicep              # Main deployment file
├── main.parameters.json    # Parameter file
├── modules/
│   ├── appService.bicep
│   ├── sqlDatabase.bicep
│   ├── keyVault.bicep
│   ├── appInsights.bicep
│   ├── appConfiguration.bicep
│   ├── containerApp.bicep
│   └── apiManagement.bicep
└── environments/
    ├── dev.parameters.json
    ├── test.parameters.json
    └── prod.parameters.json
```

## Core Modules

### App Service

```bicep
// modules/appService.bicep
@description('The name of the App Service')
param appServiceName string

@description('The location for resources')
param location string = resourceGroup().location

@description('The SKU of the App Service Plan')
@allowed(['B1', 'B2', 'S1', 'S2', 'P1v3', 'P2v3'])
param sku string = 'B1'

@description('Environment name')
@allowed(['dev', 'test', 'prod'])
param environment string

@description('Application Insights connection string')
param appInsightsConnectionString string = ''

var appServicePlanName = 'asp-${appServiceName}-${environment}'

resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: appServicePlanName
  location: location
  sku: {
    name: sku
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

resource appService 'Microsoft.Web/sites@2022-03-01' = {
  name: '${appServiceName}-${environment}'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'
      alwaysOn: environment == 'prod'
      healthCheckPath: '/health'
      appSettings: [
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: appInsightsConnectionString
        }
      ]
    }
    httpsOnly: true
  }
  identity: {
    type: 'SystemAssigned'
  }
}

output appServiceId string = appService.id
output appServiceHostName string = appService.properties.defaultHostName
output principalId string = appService.identity.principalId
```

### SQL Database

```bicep
// modules/sqlDatabase.bicep
@description('SQL Server name')
param sqlServerName string

@description('Database name')
param databaseName string

@description('Location')
param location string = resourceGroup().location

@description('Environment name')
@allowed(['dev', 'test', 'prod'])
param environment string

@description('Administrator login')
@secure()
param administratorLogin string

@description('Administrator password')
@secure()
param administratorPassword string

var skuName = environment == 'prod' ? 'S1' : 'Basic'

resource sqlServer 'Microsoft.Sql/servers@2022-05-01-preview' = {
  name: '${sqlServerName}-${environment}'
  location: location
  properties: {
    administratorLogin: administratorLogin
    administratorLoginPassword: administratorPassword
    minimalTlsVersion: '1.2'
    publicNetworkAccess: 'Enabled'
  }
}

resource sqlDatabase 'Microsoft.Sql/servers/databases@2022-05-01-preview' = {
  parent: sqlServer
  name: databaseName
  location: location
  sku: {
    name: skuName
  }
  properties: {
    collation: 'SQL_Latin1_General_CP1_CI_AS'
  }
}

resource firewallRule 'Microsoft.Sql/servers/firewallRules@2022-05-01-preview' = {
  parent: sqlServer
  name: 'AllowAzureServices'
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}

output sqlServerFqdn string = sqlServer.properties.fullyQualifiedDomainName
output databaseId string = sqlDatabase.id
output connectionString string = 'Server=tcp:${sqlServer.properties.fullyQualifiedDomainName},1433;Database=${databaseName};'
```

### Key Vault

```bicep
// modules/keyVault.bicep
@description('Key Vault name')
param keyVaultName string

@description('Location')
param location string = resourceGroup().location

@description('Environment name')
param environment string

@description('Principal IDs for access policies')
param accessPolicies array = []

resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: '${keyVaultName}-${environment}'
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enableRbacAuthorization: true
    enablePurgeProtection: environment == 'prod'
    networkAcls: {
      defaultAction: 'Allow'
      bypass: 'AzureServices'
    }
  }
}

output keyVaultId string = keyVault.id
output keyVaultUri string = keyVault.properties.vaultUri
output keyVaultName string = keyVault.name
```

### Application Insights

```bicep
// modules/appInsights.bicep
@description('Application name')
param appName string

@description('Location')
param location string = resourceGroup().location

@description('Environment name')
param environment string

resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: 'log-${appName}-${environment}'
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: environment == 'prod' ? 90 : 30
  }
}

resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'appi-${appName}-${environment}'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: logAnalytics.id
  }
}

output appInsightsId string = appInsights.id
output connectionString string = appInsights.properties.ConnectionString
output instrumentationKey string = appInsights.properties.InstrumentationKey
```

### Azure Container App

```bicep
// modules/containerApp.bicep
@description('Container App name')
param containerAppName string

@description('Location')
param location string = resourceGroup().location

@description('Environment name')
param environment string

@description('Container image')
param containerImage string

@description('Container port')
param containerPort int = 8080

@description('Environment variables')
param envVars array = []

resource containerAppEnv 'Microsoft.App/managedEnvironments@2023-05-01' = {
  name: 'cae-${containerAppName}-${environment}'
  location: location
  properties: {
    zoneRedundant: environment == 'prod'
  }
}

resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: '${containerAppName}-${environment}'
  location: location
  properties: {
    managedEnvironmentId: containerAppEnv.id
    configuration: {
      ingress: {
        external: true
        targetPort: containerPort
        transport: 'http'
      }
    }
    template: {
      containers: [
        {
          name: containerAppName
          image: containerImage
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
          env: envVars
        }
      ]
      scale: {
        minReplicas: environment == 'prod' ? 2 : 0
        maxReplicas: environment == 'prod' ? 10 : 2
      }
    }
  }
}

output fqdn string = containerApp.properties.configuration.ingress.fqdn
output containerAppId string = containerApp.id
```

### App Configuration

```bicep
// modules/appConfiguration.bicep
@description('App Configuration name')
param appConfigName string

@description('Location')
param location string = resourceGroup().location

@description('Environment name')
param environment string

@description('SKU')
@allowed(['Free', 'Standard'])
param sku string = environment == 'prod' ? 'Standard' : 'Free'

resource appConfig 'Microsoft.AppConfiguration/configurationStores@2023-03-01' = {
  name: '${appConfigName}-${environment}'
  location: location
  sku: {
    name: sku
  }
  properties: {
    disableLocalAuth: false
    enablePurgeProtection: environment == 'prod'
  }
}

output appConfigId string = appConfig.id
output appConfigEndpoint string = appConfig.properties.endpoint
```

## Main Deployment File

```bicep
// main.bicep
@description('Application name')
param appName string

@description('Location')
param location string = resourceGroup().location

@description('Environment')
@allowed(['dev', 'test', 'prod'])
param environment string

@description('SQL admin login')
@secure()
param sqlAdminLogin string

@description('SQL admin password')
@secure()
param sqlAdminPassword string

// Application Insights
module appInsights 'modules/appInsights.bicep' = {
  name: 'appInsights'
  params: {
    appName: appName
    location: location
    environment: environment
  }
}

// Key Vault
module keyVault 'modules/keyVault.bicep' = {
  name: 'keyVault'
  params: {
    keyVaultName: 'kv-${appName}'
    location: location
    environment: environment
  }
}

// SQL Database
module sqlDatabase 'modules/sqlDatabase.bicep' = {
  name: 'sqlDatabase'
  params: {
    sqlServerName: 'sql-${appName}'
    databaseName: appName
    location: location
    environment: environment
    administratorLogin: sqlAdminLogin
    administratorPassword: sqlAdminPassword
  }
}

// App Service
module appService 'modules/appService.bicep' = {
  name: 'appService'
  params: {
    appServiceName: appName
    location: location
    environment: environment
    sku: environment == 'prod' ? 'P1v3' : 'B1'
    appInsightsConnectionString: appInsights.outputs.connectionString
  }
}

// Outputs
output appServiceUrl string = 'https://${appService.outputs.appServiceHostName}'
output sqlServerFqdn string = sqlDatabase.outputs.sqlServerFqdn
output keyVaultUri string = keyVault.outputs.keyVaultUri
output appInsightsConnectionString string = appInsights.outputs.connectionString
```

## Deployment Commands

```bash
# Deploy to dev
az deployment group create \
  --resource-group rg-myapi-dev \
  --template-file main.bicep \
  --parameters @environments/dev.parameters.json

# Deploy to prod
az deployment group create \
  --resource-group rg-myapi-prod \
  --template-file main.bicep \
  --parameters @environments/prod.parameters.json
```

## Best Practices

1. **Parameterize everything** - No hardcoded values
2. **Environment-specific sizing** - Different SKUs per environment
3. **Use outputs** - Enable module composition
4. **Tag resources** - Environment, cost center, owner
5. **Enable logging** - Application Insights everywhere
6. **Security by default** - HTTPS, TLS 1.2, managed identities
