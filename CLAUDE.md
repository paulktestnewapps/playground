# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview

This is a playground repository for API development with Claude Code, focused on building .NET APIs with best practices.

## Development Commands

```bash
# Build
dotnet build

# Run tests
dotnet test

# Run with watch
dotnet watch run

# Generate migrations
dotnet ef migrations add <MigrationName>

# Apply migrations
dotnet ef database update
```

## Architecture

### Default Pattern: Service-Repository Hybrid

Use a **Service-Repository Hybrid pattern** for simple backend operations with Entity Framework Core.

```
Controller → Service → Repository → DbContext
```

**Evolve this pattern when:**
- Entities become complex → Add dedicated Repository layer
- Read/write patterns diverge → Add CQRS pattern
- Business rules become complex → Add Specification pattern
- Performance is critical → Consider Dapper over EF Core

### Layer Responsibilities

| Layer | Responsibility |
|-------|---------------|
| Controller | HTTP request/response, validation, routing |
| Service | Business logic, orchestration |
| Repository | Data access, EF Core queries |

## Code Conventions

### C# Style

- **Always use primary constructors** (IDE0290) for all classes
- **Use coalesce expression** for null checks (IDE0270)
- Use class inheritance to DRY up repeating code
- Prefer projection to DTOs over returning entities directly

### Controllers

- Name after the resource: `CustomerController`, `OrderController`
- Use `[ApiController]` attribute for automatic model validation
- Use `[Route("/")]` as base route; each action defines its own route
- POST operations return `201 Created` with no body: `return StatusCode(201);`
- Include `[ProducesResponseType]` attributes for all actions
- Use data annotations in request body classes for validation
- Inject services via constructor (primary constructor)

```csharp
[ApiController]
[Route("/")]
[Produces("application/json")]
public class CustomersController(ICustomerService customerService) : ControllerBase
{
    [HttpGet("customers/{id}")]
    [ProducesResponseType(typeof(CustomerResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<CustomerResponse>> GetById(Guid id)
    {
        var customer = await customerService.GetByIdAsync(id);
        return Ok(customer);
    }

    [HttpPost("customers")]
    [ProducesResponseType(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create([FromBody] CreateCustomerRequest request)
    {
        await customerService.CreateAsync(request);
        return StatusCode(201);
    }
}
```

### Repository Layer

- Interacts directly with DbContext
- Contains all EF Core queries
- Injected into service layer via DI
- Not unit tested directly (mocked in service tests)
- Use query extension methods for common patterns

### Services

- Use base service class to reduce repetition
- Throw `NotFoundException` for 404 responses
- Throw `AlreadyExistsException` for duplicate constraint violations

### DTOs and Mapping

- Use **AutoMapper** for object-object mapping
- Use DTO projections to reduce data transfer
- Only include fields needed for the projection
- PATCH operations should use expression trees

## Libraries

| Library | Purpose |
|---------|---------|
| AutoMapper | Object-object mapping |
| Swashbuckle | API documentation, Swagger UI |
| Entity Framework Core | ORM |
| Bunzl.APIMiddleware | Non-success response formatting, exception handling |
| Identity/IdentityServer | Authentication and authorization |
| xUnit | Testing framework |
| Moq | Mocking in unit tests |

## Testing

### Unit Testing Standards

- **Framework**: xUnit
- **Mocking**: Moq
- **Pattern**: Arrange-Act-Assert with section comments

```csharp
[Fact]
public async Task GetById_WhenCustomerExists_ReturnsCustomer()
{
    // Arrange
    var expectedCustomer = TestDataFactory.CreateCustomer();
    _mockRepository.Setup(r => r.GetByIdAsync(expectedCustomer.Id))
        .ReturnsAsync(expectedCustomer);

    // Act
    var result = await _service.GetByIdAsync(expectedCustomer.Id);

    // Assert
    Assert.NotNull(result);
    Assert.Equal(expectedCustomer.Id, result.Id);
}
```

### Testing Best Practices

- No hard-coded values; use constants or configuration
- Base assertions on expected output data
- Create shared test data classes for repeated models
- Only create helpers when used in 2+ tests AND more than 2 lines

## Infrastructure

### Configuration

- **Azure App Configuration** for managing settings across environments
- **appsettings.json** for non-sensitive local settings
- **secrets.json** for sensitive local development data (never commit)

### Infrastructure as Code

- Use **Bicep** for Azure deployments
- Automate API publication to API Management via CI/CD

**IaC Use Cases:**
- API Management publication
- Azure resources (App Service, SQL Database, Storage)
- Azure Container App configuration
- Environment variables via App Configuration
- Monitoring (Application Insights, Log Analytics)

## Exception Handling

Use Bunzl.APIMiddleware for consistent error responses:

| Exception | HTTP Status |
|-----------|-------------|
| `NotFoundException` | 404 Not Found |
| `AlreadyExistsException` | 409 Conflict |
| `ValidationException` | 400 Bad Request |

## Available Slash Commands

### Database & Schema
- `/generate-db-schema [entity]` - Generate ORM entities and DbContext
- `/generate-sql --db postgres` - Generate SQL migration scripts
- `/generate-er-diagram` - Generate ER diagram from entities

### API Implementation
- `/generate-controller [endpoint]` - Generate controller with routes
- `/generate-dtos [entity]` - Generate request/response DTOs
- `/generate-mapper [source] [target]` - Generate AutoMapper profile
- `/generate-repository [entity]` - Generate repository interface/implementation
- `/generate-error-handler` - Generate global exception handling
- `/implement-endpoint [resource.action]` - Implement a specific endpoint
- `/implement-api [spec-path]` - Implement full API from OpenAPI spec
- `/regen-api-layer` - Regenerate API layer from updated entities

### Architecture
- `/recommend-pattern` - Analyze and recommend architectural patterns
- `/analyze-endpoint [endpoint]` - Analyze endpoint complexity and intent
- `/compare-patterns [p1] [p2]` - Compare two patterns for your codebase
- `/apply-pattern [pattern]` - Apply architectural pattern (hexagonal, clean, etc.)

### Testing
- `/generate-tests [target]` - Generate unit tests for a class or method

### DevOps
- `/generate-pipeline [platform]` - Generate CI/CD pipeline
- `/generate-iac [tool] [cloud]` - Generate Infrastructure as Code
- `/devops-review` - Review DevOps configuration
- `/deploy-architecture` - Generate deployment architecture docs
