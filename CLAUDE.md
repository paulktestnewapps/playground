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

# Format code
dotnet format

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

## Database Design

Follow the conventions in [docs/Guides/database_design_guide.md](docs/Guides/database_design_guide.md) and [docs/Guides/pk_type_decision_guide.md](docs/Guides/pk_type_decision_guide.md).

### Table Naming
- Concise and descriptive (2-3 words ideal)
- No "Bunzl" prefix by default
- Use casing consistent with existing tables or database provider conventions

### Required Audit Columns
Every table must include:
| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| CreatedOn | datetime (UTC) | No | Set on insert |
| CreatedBy | string | No | Set on insert |
| UpdatedOn | datetime (UTC) | Yes | Set on update only |
| UpdatedBy | string | Yes | Set on update only |

### Soft Deletes
- Include `IsActive` (bool, not-null) for tables with API DELETE operations
- Avoid `DateOnly` types; use `DateTime` instead

### Primary Key Selection

**Default: Incremental integers** for most internal/performance-sensitive systems.

**Use UUIDs only when:**
- Primary key exposed via APIs with private/sensitive data (e.g., User PII, Orders)
- Distributed systems requiring independent ID generation
- Data merging/sync across multiple databases

**Do NOT use UUIDs for:**
- Lookup/reference tables (e.g., OrderStatus, ResourceType)
- Internal-only tables not exposed via APIs

**Hybrid approach:** Use integer PK internally + separate UUID column for external APIs.

### Normalization
- Use join tables for many-to-many relationships
- Avoid comma-separated values in columns
- Create separate tables for status/type lookups (e.g., OrderStatus with Id + Value)
- Limit tables to ~10 value columns; more may indicate need for optimization

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
- **Never unit tested directly** - repositories are mocked in service tests
- Use query extension methods for common patterns
- **Keep repositories "thin"** - minimal to no business logic
  - Repositories should only perform CRUD operations and simple queries
  - All business logic belongs in the service layer
  - This ensures services are fully testable via mocked repositories
  - If a repository method requires complex logic, move it to the service layer

### Services

- Use base service class to reduce repetition
- Throw `NotFoundException` for 404 responses
- Throw `AlreadyExistsException` for duplicate constraint violations

### DTOs and Mapping

- Use **AutoMapper** for object-object mapping
- Mappings and DTOs should be in different classes/files
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

### Mandatory Testing Rule

**ALWAYS add or modify unit tests when code is added or modified.** This is not optional:
- New service methods require corresponding unit tests
- Modified service methods require updated tests
- New business logic in any layer requires test coverage
- Bug fixes should include a regression test

### Task Completion Criteria

**A task is NOT complete until:**
1. All new/modified code has corresponding unit tests
2. All unit tests pass (`dotnet test` exits with code 0)
3. No skipped or ignored tests for the new functionality
4. Code formatting is correct (`dotnet format` produces no changes)

**Before marking any feature or task as done:**
- Run `dotnet format` to ensure consistent code style
- Run `dotnet test` and verify all tests pass
- If tests fail, fix the code or tests before proceeding
- Never leave a task with failing tests or formatting issues

### What to Test (and What Not To)

| Layer | Test? | Reason |
|-------|-------|--------|
| Services | **Yes** | Contains business logic; mock repositories |
| AutoMapper Profiles | **Yes** | Verify mapping configurations are correct |
| Controllers | Optional | Thin layer; mostly delegates to services |
| Repositories | **No** | Thin data access; no business logic to test |
| DTOs | **No** | Data containers only; no logic to test |

**Never set up in-memory databases or DbContext for unit tests.** Mock the repository interface instead.

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

### Testing AutoMapper Profiles

Test mapping configurations to catch mismatches early:

```csharp
public class CustomerMappingProfileTests
{
    private readonly IMapper _mapper;

    public CustomerMappingProfileTests()
    {
        var config = new MapperConfiguration(cfg => cfg.AddProfile<CustomerMappingProfile>());
        config.AssertConfigurationIsValid(); // Validates all mappings
        _mapper = config.CreateMapper();
    }

    [Fact]
    public void Map_CustomerToCustomerResponse_MapsCorrectly()
    {
        // Arrange
        var customer = TestDataFactory.CreateCustomer();

        // Act
        var result = _mapper.Map<CustomerResponse>(customer);

        // Assert
        Assert.Equal(customer.Id, result.Id);
        Assert.Equal(customer.Name, result.Name);
    }
}
```

### Testing Best Practices

- No hard-coded values; use constants or configuration
- Base assertions on expected output data
- Create shared test data classes for repeated models
- Only create helpers when used in 2+ tests AND more than 2 lines
- Mock all dependencies (repositories, external services) - never use real implementations
- Test both happy paths and error cases (e.g., `NotFoundException` scenarios)

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
- `/api-spec-to-postgres [path]` - Generate PostgreSQL DDL from OpenAPI spec
- `/derive-entities [path]` - Extract entity list summary from API spec
- `/generate-efcore-from-ddl [path]` - Generate EF Core entities and DbContext from PostgreSQL DDL
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
