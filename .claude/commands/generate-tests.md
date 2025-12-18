---
allowed-tools: Read, Edit, Write, Glob, Grep
argument-hint: [class-or-method-name]
description: Generate unit tests using xUnit and Moq
---

## Context

Analyze the target for testing: $ARGUMENTS

Review:
- The class or method to test
- Its dependencies (for mocking)
- Existing test patterns in the project

## Task

Generate comprehensive unit tests following project conventions:

### Standards
1. **Framework**: xUnit
2. **Mocking**: Moq
3. **Pattern**: Arrange-Act-Assert with section comments (`// Arrange`, `// Act`, `// Assert`)
4. **Naming**: `MethodName_StateUnderTest_ExpectedBehavior`

### Requirements
- No hard-coded values - use test data factories
- Base assertions on expected output data
- Create shared test data classes for repeated models
- Only create helper functions when used in 2+ tests AND more than 2 lines

### Test Coverage
Generate tests for:
- Happy path scenarios
- Edge cases (null, empty, boundary values)
- Exception scenarios (NotFoundException, AlreadyExistsException, ValidationException)
- Verify mock interactions where appropriate

### Output Structure
1. Test class with proper setup (constructor with mocks)
2. Test data factory if models are reused
3. All test methods with proper naming
4. Theory tests for parameterized scenarios

### Example Pattern
```csharp
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
}
```
