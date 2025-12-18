---
name: secrets-wiring
description: Configure secret management with Key Vault, Secrets Manager, or similar services. Use when setting up secure credential storage, application secret bindings, or rotation policies.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# Secrets Wiring

## Purpose

Configures secure secret management and application integration.

## When to Use

- Setting up secret stores
- Configuring application secret access
- Implementing secret rotation
- Securing connection strings and API keys

## Inputs

- Application secrets inventory
- Secret store choice (Key Vault, Secrets Manager, etc.)
- Access requirements
- Rotation policies

## Outputs

- Secret store configuration
- Application bindings
- Rotation policy setup
- Access control configuration

## Secret Store Setup

### Azure Key Vault
```bicep
// modules/keyVault.bicep
@description('Key Vault name')
param keyVaultName string

@description('Location')
param location string = resourceGroup().location

@description('Principal IDs for access')
param accessPrincipalIds array = []

@description('Enable soft delete')
param enableSoftDelete bool = true

resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: keyVaultName
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableSoftDelete: enableSoftDelete
    softDeleteRetentionInDays: 90
    enableRbacAuthorization: true
    enablePurgeProtection: true
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
    }
  }
}

// Grant access to specified principals
resource keyVaultAccess 'Microsoft.Authorization/roleAssignments@2022-04-01' = [for principalId in accessPrincipalIds: {
  name: guid(keyVault.id, principalId, 'Key Vault Secrets User')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6') // Key Vault Secrets User
    principalId: principalId
    principalType: 'ServicePrincipal'
  }
}]

output keyVaultUri string = keyVault.properties.vaultUri
output keyVaultId string = keyVault.id
```

### AWS Secrets Manager
```hcl
# modules/secrets/main.tf
variable "secret_name" {
  type = string
}

variable "secret_value" {
  type      = string
  sensitive = true
}

variable "rotation_lambda_arn" {
  type    = string
  default = null
}

resource "aws_secretsmanager_secret" "main" {
  name                    = var.secret_name
  recovery_window_in_days = 7

  tags = {
    ManagedBy = "Terraform"
  }
}

resource "aws_secretsmanager_secret_version" "main" {
  secret_id     = aws_secretsmanager_secret.main.id
  secret_string = var.secret_value
}

resource "aws_secretsmanager_secret_rotation" "main" {
  count               = var.rotation_lambda_arn != null ? 1 : 0
  secret_id           = aws_secretsmanager_secret.main.id
  rotation_lambda_arn = var.rotation_lambda_arn

  rotation_rules {
    automatically_after_days = 30
  }
}

output "secret_arn" {
  value = aws_secretsmanager_secret.main.arn
}
```

## Application Integration

### ASP.NET Core + Azure Key Vault
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add Key Vault configuration
if (!builder.Environment.IsDevelopment())
{
    var keyVaultUri = builder.Configuration["KeyVault:Uri"];
    builder.Configuration.AddAzureKeyVault(
        new Uri(keyVaultUri),
        new DefaultAzureCredential());
}

// Use secrets in configuration
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration["ConnectionStrings:Database"]));
```

### App Service Key Vault Reference
```json
// App Settings
{
  "ConnectionStrings__Database": "@Microsoft.KeyVault(VaultName=myvault;SecretName=db-connection-string)",
  "ExternalApi__ApiKey": "@Microsoft.KeyVault(VaultName=myvault;SecretName=external-api-key)"
}
```

### Node.js + AWS Secrets Manager
```javascript
const { SecretsManagerClient, GetSecretValueCommand } = require("@aws-sdk/client-secrets-manager");

const client = new SecretsManagerClient({ region: "us-east-1" });

async function getSecret(secretName) {
  const response = await client.send(
    new GetSecretValueCommand({ SecretId: secretName })
  );
  return JSON.parse(response.SecretString);
}

// Usage
const dbCredentials = await getSecret("myapp/database");
const connection = await createConnection({
  host: dbCredentials.host,
  user: dbCredentials.username,
  password: dbCredentials.password,
});
```

## Secret Categories

| Category | Examples | Rotation Frequency |
|----------|----------|-------------------|
| Database | Connection strings | 30-90 days |
| API Keys | External service keys | 90 days |
| Certificates | TLS/SSL certs | Before expiry |
| Encryption | Data encryption keys | 365 days |
| OAuth | Client secrets | 180 days |

## Security Best Practices

1. **Never commit secrets** - Use .gitignore, pre-commit hooks
2. **Use managed identities** - Avoid storing credentials when possible
3. **Least privilege** - Only grant necessary access
4. **Audit access** - Enable logging on secret stores
5. **Rotate regularly** - Automate rotation where possible
6. **Separate by environment** - Different secrets per env
