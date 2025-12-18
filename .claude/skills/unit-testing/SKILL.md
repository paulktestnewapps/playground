---
name: unit-testing
description: Generate unit tests using xUnit and Moq following project conventions. Use when creating tests for services, writing test data factories, or implementing Arrange-Act-Assert patterns.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# Unit Testing

## Purpose

Generates unit tests following project conventions with xUnit and Moq.

## When to Use

- Creating unit tests for services
- Writing test data factories
- Implementing mock setups
- Testing business logic

## Project-Specific Conventions

### CRITICAL: Follow These Standards

1. **Framework**: xUnit
2. **Mocking**: Moq
3. **Pattern**: Arrange-Act-Assert with section comments
4. **No hard-coded values** - use constants or base assertions on expected output
5. **DRY test files** - create shared data classes for repeated models
6. **Helper functions** only when used in 2+ tests AND more than 2 lines

## Test Class Template

```csharp
using Moq;
using Xunit;

namespace MyApi.Tests.Services;

public class CustomerServiceTests
{
    private readonly Mock<ICustomerRepository> _mockRepository;
    private readonly Mock<IMapper> _mockMapper;
    private readonly Mock<ILogger<CustomerService>> _mockLogger;
    private readonly CustomerService _sut; // System Under Test

    public CustomerServiceTests()
    {
        _mockRepository = new Mock<ICustomerRepository>();
        _mockMapper = new Mock<IMapper>();
        _mockLogger = new Mock<ILogger<CustomerService>>();
        _sut = new CustomerService(
            _mockRepository.Object,
            _mockMapper.Object,
            _mockLogger.Object);
    }

    [Fact]
    public async Task GetByIdAsync_WhenCustomerExists_ReturnsCustomer()
    {
        // Arrange
        var expectedCustomer = CustomerTestData.CreateDetailResponse();
        _mockRepository
            .Setup(r => r.GetByIdAsync(expectedCustomer.Id, It.IsAny<CancellationToken>()))
            .ReturnsAsync(expectedCustomer);

        // Act
        var result = await _sut.GetByIdAsync(expectedCustomer.Id, CancellationToken.None);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(expectedCustomer.Id, result.Id);
        Assert.Equal(expectedCustomer.Name, result.Name);
        Assert.Equal(expectedCustomer.Email, result.Email);
    }

    [Fact]
    public async Task GetByIdAsync_WhenCustomerNotFound_ThrowsNotFoundException()
    {
        // Arrange
        var customerId = Guid.NewGuid();
        _mockRepository
            .Setup(r => r.GetByIdAsync(customerId, It.IsAny<CancellationToken>()))
            .ReturnsAsync((CustomerDetailResponse?)null);

        // Act & Assert
        await Assert.ThrowsAsync<NotFoundException>(
            () => _sut.GetByIdAsync(customerId, CancellationToken.None));
    }

    [Fact]
    public async Task CreateAsync_WhenEmailExists_ThrowsAlreadyExistsException()
    {
        // Arrange
        var request = CustomerTestData.CreateRequest();
        _mockRepository
            .Setup(r => r.ExistsByEmailAsync(request.Email, It.IsAny<CancellationToken>()))
            .ReturnsAsync(true);

        // Act & Assert
        await Assert.ThrowsAsync<AlreadyExistsException>(
            () => _sut.CreateAsync(request, CancellationToken.None));
    }

    [Fact]
    public async Task CreateAsync_WhenValidRequest_CreatesCustomer()
    {
        // Arrange
        var request = CustomerTestData.CreateRequest();
        var mappedCustomer = CustomerTestData.CreateEntity();

        _mockRepository
            .Setup(r => r.ExistsByEmailAsync(request.Email, It.IsAny<CancellationToken>()))
            .ReturnsAsync(false);

        _mockMapper
            .Setup(m => m.Map<Customer>(request))
            .Returns(mappedCustomer);

        // Act
        await _sut.CreateAsync(request, CancellationToken.None);

        // Assert
        _mockRepository.Verify(
            r => r.AddAsync(mappedCustomer, It.IsAny<CancellationToken>()),
            Times.Once);
    }
}
```

## Test Data Factory

Create shared test data classes to keep tests DRY:

```csharp
namespace MyApi.Tests.TestData;

public static class CustomerTestData
{
    public static CreateCustomerRequest CreateRequest(
        string? name = null,
        string? email = null)
    {
        return new CreateCustomerRequest(
            Name: name ?? "Test Customer",
            Email: email ?? $"test-{Guid.NewGuid():N}@example.com",
            PhoneNumber: "+1234567890",
            Notes: "Test notes"
        );
    }

    public static Customer CreateEntity(
        Guid? id = null,
        string? name = null,
        string? email = null)
    {
        return new Customer
        {
            Id = id ?? Guid.NewGuid(),
            Name = name ?? "Test Customer",
            Email = email ?? $"test-{Guid.NewGuid():N}@example.com",
            PhoneNumber = "+1234567890",
            Notes = "Test notes",
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = null
        };
    }

    public static CustomerDetailResponse CreateDetailResponse(
        Guid? id = null,
        string? name = null)
    {
        var entityId = id ?? Guid.NewGuid();
        return new CustomerDetailResponse(
            Id: entityId,
            Name: name ?? "Test Customer",
            Email: $"test-{entityId:N}@example.com",
            PhoneNumber: "+1234567890",
            Notes: "Test notes",
            CreatedAt: DateTime.UtcNow,
            UpdatedAt: null
        );
    }

    public static CustomerSummaryResponse CreateSummaryResponse(
        Guid? id = null,
        string? name = null)
    {
        var entityId = id ?? Guid.NewGuid();
        return new CustomerSummaryResponse(
            Id: entityId,
            Name: name ?? "Test Customer",
            Email: $"test-{entityId:N}@example.com"
        );
    }

    public static List<CustomerSummaryResponse> CreateSummaryResponseList(int count = 3)
    {
        return Enumerable.Range(1, count)
            .Select(i => CreateSummaryResponse(name: $"Customer {i}"))
            .ToList();
    }
}
```

## Testing Patterns

### Theory with InlineData

```csharp
[Theory]
[InlineData("")]
[InlineData(" ")]
[InlineData(null)]
public async Task CreateAsync_WhenNameInvalid_ThrowsValidationException(string? invalidName)
{
    // Arrange
    var request = CustomerTestData.CreateRequest(name: invalidName!);

    // Act & Assert
    await Assert.ThrowsAsync<ValidationException>(
        () => _sut.CreateAsync(request, CancellationToken.None));
}
```

### Theory with MemberData

```csharp
public static IEnumerable<object[]> InvalidEmailTestData =>
    new List<object[]>
    {
        new object[] { "notanemail" },
        new object[] { "@missing-local.com" },
        new object[] { "missing-domain@" },
        new object[] { "spaces in@email.com" }
    };

[Theory]
[MemberData(nameof(InvalidEmailTestData))]
public async Task CreateAsync_WhenEmailInvalid_ThrowsValidationException(string invalidEmail)
{
    // Arrange
    var request = CustomerTestData.CreateRequest(email: invalidEmail);

    // Act & Assert
    await Assert.ThrowsAsync<ValidationException>(
        () => _sut.CreateAsync(request, CancellationToken.None));
}
```

### Verifying Method Calls

```csharp
[Fact]
public async Task DeleteAsync_WhenCustomerExists_CallsRepositoryDelete()
{
    // Arrange
    var customerId = Guid.NewGuid();

    // Act
    await _sut.DeleteAsync(customerId, CancellationToken.None);

    // Assert
    _mockRepository.Verify(
        r => r.DeleteAsync(customerId, It.IsAny<CancellationToken>()),
        Times.Once);
}

[Fact]
public async Task GetAllAsync_WhenCalled_PassesFilterToRepository()
{
    // Arrange
    var filter = new CustomerFilterRequest(
        SearchTerm: "test",
        CreatedAfter: null,
        CreatedBefore: null);

    _mockRepository
        .Setup(r => r.GetAllAsync(filter, It.IsAny<CancellationToken>()))
        .ReturnsAsync(new List<CustomerSummaryResponse>());

    // Act
    await _sut.GetAllAsync(filter, CancellationToken.None);

    // Assert
    _mockRepository.Verify(
        r => r.GetAllAsync(
            It.Is<CustomerFilterRequest>(f => f.SearchTerm == "test"),
            It.IsAny<CancellationToken>()),
        Times.Once);
}
```

## Naming Conventions

```
MethodName_StateUnderTest_ExpectedBehavior

Examples:
- GetByIdAsync_WhenCustomerExists_ReturnsCustomer
- GetByIdAsync_WhenCustomerNotFound_ThrowsNotFoundException
- CreateAsync_WhenEmailExists_ThrowsAlreadyExistsException
- CreateAsync_WhenValidRequest_CreatesCustomer
- DeleteAsync_WhenCustomerNotFound_ThrowsNotFoundException
```

## Best Practices

1. **One assertion concept per test** - test one behavior
2. **Independent tests** - no test should depend on another
3. **Fast tests** - mock external dependencies
4. **Readable** - test should document the expected behavior
5. **Deterministic** - same result every time
6. **Base assertions on expected data** - don't hard-code magic values

## Helper Functions (Use Sparingly)

Only create when:
- Used in 2+ tests
- More than 2 lines of code

```csharp
// Good helper - used multiple times, reduces repetition
private void SetupRepositoryToReturnCustomer(CustomerDetailResponse customer)
{
    _mockRepository
        .Setup(r => r.GetByIdAsync(customer.Id, It.IsAny<CancellationToken>()))
        .ReturnsAsync(customer);
}

// Bad helper - only 1 line, just inline it
private void SetupLoggerMock() => _mockLogger = new Mock<ILogger>();
```
