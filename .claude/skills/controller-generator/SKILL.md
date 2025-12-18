---
name: controller-generator
description: Generate API controllers with routes, DI setup, and request/response models following project conventions. Use when creating new endpoints, implementing REST APIs, or scaffolding controller boilerplate.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# Controller Generator

## Purpose

Generates controller boilerplate following established project conventions for ASP.NET Core APIs.

## When to Use

- Creating new API endpoints
- Scaffolding CRUD operations
- Implementing REST controllers
- Adding new actions to existing controllers

## Project-Specific Conventions

### CRITICAL: Follow These Standards

1. **Always use primary constructors** (IDE0290)
2. **Use `[Route("/")]`** as base route - each action defines its own route
3. **POST returns 201 with no body**: `return StatusCode(201);`
4. **Use coalesce expressions** for null checks (IDE0270)
5. **Include `[ProducesResponseType]`** attributes for all actions

## Controller Template

```csharp
using Microsoft.AspNetCore.Mvc;

namespace MyApi.Controllers;

[ApiController]
[Route("/")]
[Produces("application/json")]
public class CustomersController(
    ICustomerService customerService,
    ILogger<CustomersController> logger) : ControllerBase
{
    /// <summary>
    /// Get all customers with optional filtering
    /// </summary>
    [HttpGet("customers")]
    [ProducesResponseType(typeof(IEnumerable<CustomerSummaryResponse>), StatusCodes.Status200OK)]
    public async Task<ActionResult<IEnumerable<CustomerSummaryResponse>>> GetAll(
        [FromQuery] CustomerFilterRequest? filter,
        CancellationToken ct)
    {
        var customers = await customerService.GetAllAsync(filter, ct);
        return Ok(customers);
    }

    /// <summary>
    /// Get customer by ID
    /// </summary>
    [HttpGet("customers/{id:guid}")]
    [ProducesResponseType(typeof(CustomerDetailResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<CustomerDetailResponse>> GetById(Guid id, CancellationToken ct)
    {
        var customer = await customerService.GetByIdAsync(id, ct);
        return Ok(customer);
    }

    /// <summary>
    /// Create a new customer
    /// </summary>
    [HttpPost("customers")]
    [ProducesResponseType(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<IActionResult> Create(
        [FromBody] CreateCustomerRequest request,
        CancellationToken ct)
    {
        await customerService.CreateAsync(request, ct);
        return StatusCode(201);
    }

    /// <summary>
    /// Update an existing customer
    /// </summary>
    [HttpPut("customers/{id:guid}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Update(
        Guid id,
        [FromBody] UpdateCustomerRequest request,
        CancellationToken ct)
    {
        await customerService.UpdateAsync(id, request, ct);
        return Ok();
    }

    /// <summary>
    /// Partially update a customer
    /// </summary>
    [HttpPatch("customers/{id:guid}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Patch(
        Guid id,
        [FromBody] PatchCustomerRequest request,
        CancellationToken ct)
    {
        await customerService.PatchAsync(id, request, ct);
        return Ok();
    }

    /// <summary>
    /// Delete a customer
    /// </summary>
    [HttpDelete("customers/{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(Guid id, CancellationToken ct)
    {
        await customerService.DeleteAsync(id, ct);
        return NoContent();
    }
}
```

## Request DTO with Validation

```csharp
using System.ComponentModel.DataAnnotations;

namespace MyApi.Contracts.Requests;

public record CreateCustomerRequest(
    [Required]
    [StringLength(100, MinimumLength = 1)]
    string Name,

    [Required]
    [EmailAddress]
    string Email,

    [Phone]
    string? PhoneNumber,

    [StringLength(500)]
    string? Notes
);

public record UpdateCustomerRequest(
    [Required]
    [StringLength(100, MinimumLength = 1)]
    string Name,

    [Required]
    [EmailAddress]
    string Email,

    [Phone]
    string? PhoneNumber,

    [StringLength(500)]
    string? Notes
);

public record PatchCustomerRequest
{
    [StringLength(100, MinimumLength = 1)]
    public string? Name { get; init; }

    [EmailAddress]
    public string? Email { get; init; }

    [Phone]
    public string? PhoneNumber { get; init; }

    [StringLength(500)]
    public string? Notes { get; init; }
}
```

## Response DTOs

```csharp
namespace MyApi.Contracts.Responses;

public record CustomerDetailResponse(
    Guid Id,
    string Name,
    string Email,
    string? PhoneNumber,
    string? Notes,
    DateTime CreatedAt,
    DateTime? UpdatedAt
);

public record CustomerSummaryResponse(
    Guid Id,
    string Name,
    string Email
);
```

## Exception Handling

The controller should NOT catch exceptions. Use Bunzl.APIMiddleware for consistent error handling:

```csharp
// In the service layer, throw these exceptions:
throw new NotFoundException($"Customer with ID '{id}' was not found.");
throw new AlreadyExistsException($"Customer with email '{email}' already exists.");
```

## REST Conventions

| Action | HTTP Method | Route | Success Status | Response Body |
|--------|-------------|-------|----------------|---------------|
| List | GET | /resources | 200 OK | Collection |
| Get | GET | /resources/{id} | 200 OK | Single item |
| Create | POST | /resources | 201 Created | None |
| Update | PUT | /resources/{id} | 200 OK | None |
| Patch | PATCH | /resources/{id} | 200 OK | None |
| Delete | DELETE | /resources/{id} | 204 No Content | None |

## Checklist Before Generation

- [ ] Primary constructor used (no field declarations)
- [ ] Route base is `[Route("/")]`
- [ ] All actions have `[ProducesResponseType]`
- [ ] POST returns `StatusCode(201)` with no body
- [ ] CancellationToken included in all async methods
- [ ] XML documentation on public methods
- [ ] Request DTOs have validation attributes
