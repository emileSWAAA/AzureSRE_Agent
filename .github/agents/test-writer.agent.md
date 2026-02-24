---
description: 'Writes unit tests for code files based on user requirements and best practices.'
---
# Unit Test Generation Guide

This document provides comprehensive instructions for generating unit tests across any .NET solution to ensure uniformity, quality, and maintainability.

## Required Packages and Libraries

### Core Testing Frameworks
- **xUnit** - Primary testing framework
- **Moq** - Mocking framework for dependencies
- **AutoFixture** - Automatic test data generation
- **Shouldly** - Readable assertion syntax

### Package References
```xml
<PackageReference Include="xunit" Version="2.x.x" />
<PackageReference Include="xunit.runner.visualstudio" Version="2.x.x" />
<PackageReference Include="Moq" Version="4.x.x" />
<PackageReference Include="AutoFixture" Version="4.x.x" />
<PackageReference Include="AutoFixture.Xunit2" Version="4.x.x" />
<PackageReference Include="AutoFixture.AutoMoq" Version="4.x.x" />
<PackageReference Include="Shouldly" Version="4.x.x" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.x.x" />
```

## Test Project Structure

### Directory Structure
```
tests/
├── {ProjectName}.Tests/
│   ├── Services/
│   ├── Repositories/
│   ├── Models/
│   └── Endpoints/
└── Common/
    └── TestCategories.cs
```

## Test Categories

Use test categories from `tests/Common/TestCategories.cs` for organization:

```csharp
[Trait("Category", TestCategories.Unit)]
[Trait("Category", TestCategories.Integration)]
[Trait("Category", TestCategories.Security)]
```

## AutoFixture Configuration

### Standard Setup
```csharp
private readonly IFixture _fixture;

public YourServiceTests()
{
    _fixture = new Fixture()
        .Customize(new AutoMoqCustomization { ConfigureMembers = true });
}
```

### Custom Customizations
```csharp
// For complex objects
_fixture.Customize<YourEntity>(c => c
    .Without(x => x.NavigationProperty)
    .With(x => x.RequiredProperty, "value"));

// For value objects
_fixture.Customize<ValueObject>(c => c
    .FromFactory(() => ValueObject.Create("value").Value));
```

## Test Patterns

### AAA Pattern (Arrange-Act-Assert)
All tests must follow the Arrange-Act-Assert pattern with clear separation:

```csharp
[Fact]
public async Task MethodName_Scenario_ExpectedResult()
{
    // Arrange
    var entity = _fixture.Create<Entity>();
    var mock = _fixture.Freeze<Mock<IDependency>>();
    mock.Setup(x => x.Method(It.IsAny<Type>()))
        .ReturnsAsync(expectedValue);
    var sut = _fixture.Create<ClassUnderTest>();

    // Act
    var result = await sut.MethodUnderTest(entity);

    // Assert
    result.Should().NotBeNull();
    result.Property.Should().Be(expectedValue);
    mock.Verify(x => x.Method(It.IsAny<Type>()), Times.Once);
}
```

### Theory with InlineData for Multiple Scenarios
```csharp
[Theory]
[InlineData("", false)]
[InlineData(null, false)]
[InlineData("valid", true)]
public void Validate_VariousInputs_ReturnsExpectedResult(string input, bool expected)
{
    // Arrange & Act
    var result = Validator.IsValid(input);

    // Assert
    result.Should().Be(expected);
}
```

## Happy Path and Unhappy Path Testing

### Happy Path (Positive Scenarios)
Test the expected successful flow:
```csharp
[Fact]
[Trait("Category", TestCategories.Unit)]
public async Task CreateEntity_ValidData_ReturnsCreatedEntity()
{
    // Test successful creation with valid data
}

[Fact]
public async Task UpdateEntity_ExistingEntity_ReturnsUpdatedEntity()
{
    // Test successful update
}
```

### Unhappy Path (Negative Scenarios)
Test failure scenarios and edge cases:
```csharp
[Fact]
public async Task CreateEntity_NullInput_ThrowsArgumentNullException()
{
    // Arrange
    var sut = _fixture.Create<Service>();

    // Act
    Func<Task> act = async () => await sut.Create(null);

    // Assert
    await act.Should().ThrowAsync<ArgumentNullException>();
}

[Fact]
public async Task GetEntity_NonExistentId_ThrowsNotFoundException()
{
    // Test entity not found scenario
}

[Fact]
public async Task UpdateEntity_ConcurrentModification_ThrowsConcurrencyException()
{
    // Test concurrency issues
}
```

## Input Validation Testing

### Unicode and Special Characters
```csharp
[Theory]
[InlineData("Test with émoji 🚀")]
[InlineData("Unicode: 你好世界")]
[InlineData("Special: <>?*|")]
[InlineData("RTL: مرحبا")]
[InlineData("Emoji sequence: 👨‍👩‍👧‍👦")]
public async Task HandleInput_UnicodeAndEmoji_HandlesCorrectly(string input)
{
    // Arrange
    var entity = _fixture.Build<Entity>()
        .With(x => x.Name, input)
        .Create();
    var sut = _fixture.Create<Service>();

    // Act
    var result = await sut.Process(entity);

    // Assert
    result.Should().NotBeNull();
    result.Name.Should().Be(input);
}
```

### Boundary Testing
```csharp
[Theory]
[InlineData(0)]           // Minimum
[InlineData(1)]           // Just above minimum
[InlineData(255)]         // Maximum
[InlineData(256)]         // Just above maximum
[InlineData(-1)]          // Below minimum
public void ValidateLength_BoundaryValues_ReturnsExpectedResult(int length)
{
    // Test boundary conditions
}
```

### Null and Empty String Handling
```csharp
[Theory]
[InlineData(null)]
[InlineData("")]
[InlineData("   ")]
[InlineData("\t\n")]
public async Task ProcessInput_NullOrWhitespace_ThrowsValidationException(string input)
{
    // Test null/empty/whitespace handling
}
```

## Security Testing

### SQL Injection Testing
```csharp
[Theory]
[Trait("Category", TestCategories.Security)]
[InlineData("'; DROP TABLE Users; --")]
[InlineData("1' OR '1'='1")]
[InlineData("admin'--")]
[InlineData("' UNION SELECT * FROM Users--")]
public async Task Search_SqlInjectionAttempt_DoesNotExecuteInjection(string maliciousInput)
{
    // Arrange
    var mockRepository = _fixture.Freeze<Mock<IRepository>>();
    mockRepository.Setup(x => x.Search(It.IsAny<string>()))
        .ReturnsAsync(new List<Entity>());
    var sut = _fixture.Create<Service>();

    // Act
    var result = await sut.Search(maliciousInput);

    // Assert
    result.Should().NotBeNull();
    mockRepository.Verify(x => x.Search(maliciousInput), Times.Once);
    // Verify parameterized queries are used, not string concatenation
}
```

### XSS Testing
```csharp
[Theory]
[Trait("Category", TestCategories.Security)]
[InlineData("<script>alert('XSS')</script>")]
[InlineData("<img src=x onerror=alert('XSS')>")]
[InlineData("javascript:alert('XSS')")]
public async Task CreateEntity_XssAttempt_SanitizesInput(string xssInput)
{
    // Test XSS prevention
}
```

### Role-Based Access Control (RBAC) Testing
```csharp
[Theory]
[Trait("Category", TestCategories.Security)]
[InlineData("Admin", true)]
[InlineData("User", false)]
[InlineData("Guest", false)]
[InlineData(null, false)]
public async Task DeleteEntity_VariousRoles_ReturnsExpectedAuthorization(string role, bool shouldSucceed)
{
    // Arrange
    var mockAuthService = _fixture.Freeze<Mock<IAuthorizationService>>();
    mockAuthService.Setup(x => x.HasRole(role, "Delete"))
        .Returns(shouldSucceed);
    var sut = _fixture.Create<Service>();

    // Act
    if (shouldSucceed)
    {
        var result = await sut.Delete(Guid.NewGuid(), role);
        result.Should().BeTrue();
    }
    else
    {
        Func<Task> act = async () => await sut.Delete(Guid.NewGuid(), role);
        await act.Should().ThrowAsync<UnauthorizedAccessException>();
    }
}

[Fact]
[Trait("Category", TestCategories.Security)]
public async Task AccessResource_UnauthorizedUser_ThrowsUnauthorizedException()
{
    // Test unauthorized access
}
```

## Mocking Best Practices

### Setup Mocks with AutoFixture
```csharp
// Freeze mock to use same instance throughout test
var mockRepository = _fixture.Freeze<Mock<IRepository>>();
mockRepository.Setup(x => x.GetById(It.IsAny<Guid>()))
    .ReturnsAsync(_fixture.Create<Entity>());
```

### Verify Mock Interactions
```csharp
// Verify method was called exactly once
mock.Verify(x => x.Method(It.IsAny<Type>()), Times.Once);

// Verify method was never called
mock.Verify(x => x.Method(It.IsAny<Type>()), Times.Never);

// Verify with specific parameters
mock.Verify(x => x.Method(specificValue), Times.Once);

// Verify all setups were called
mock.VerifyAll();
```

### Setup Async Methods
```csharp
mock.Setup(x => x.MethodAsync(It.IsAny<Type>()))
    .ReturnsAsync(value);

// With callback
mock.Setup(x => x.MethodAsync(It.IsAny<Type>()))
    .Callback<Type>(param => { /* side effect */ })
    .ReturnsAsync(value);
```

## Shouldly Patterns

### Object Assertions
```csharp
result.ShouldNotBeNull();
result.ShouldBeOfType<YourDto>();
result.ShouldBeEquivalentTo(expected, options => options
    .Excluding(x => x.Id)
    .Excluding(x => x.CreatedAt));
```

### Collection Assertions
```csharp
result.ShouldNotBeEmpty();
result.ShouldHaveCount(3);
result.ShouldContain(x => x.Name == "Test");
result.ShouldAllBe(x => x.IsActive);
result.ShouldBeInAscendingOrder(x => x.Name);
```

### Exception Assertions
```csharp
Func<Task> act = async () => await sut.Method(invalidInput);
await act.ShouldThrowAsync<ValidationException>()
    .WithMessage("*required*");

await act.ShouldThrowAsync<ArgumentNullException>()
    .Where(e => e.ParamName == "input");
```

### String Assertions
```csharp
result.Name.ShouldNotBeNullOrWhiteSpace();
result.Name.ShouldStartWith("Test");
result.Name.ShouldContain("expected");
result.Name.ShouldMatchRegex(@"^\d{3}-\d{3}-\d{4}$");
```

## Code Coverage Requirements

### Minimum Coverage Targets
- **Overall Solution**: 80% minimum
- **Application Layer**: 90% minimum
- **Domain Layer**: 95% minimum
- **Infrastructure Layer**: 70% minimum (due to framework code)

### Coverage Exemptions
Use `[ExcludeFromCodeCoverage]` attribute for:
- DTOs with only properties
- Migration files
- Program.cs / Startup.cs
- Auto-generated code

### Measuring Coverage
```bash
# Run tests with coverage
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover

# Generate report
reportgenerator -reports:coverage.opencover.xml -targetdir:coverage-report
```

## Test Data Management

### Use AutoFixture for Test Data
```csharp
// Simple objects
var entity = _fixture.Create<Entity>();

// Collections
var entities = _fixture.CreateMany<Entity>(5).ToList();

// With specific values
var entity = _fixture.Build<Entity>()
    .With(x => x.Status, EntityStatus.Active)
    .Without(x => x.NavigationProperty)
    .Create();
```

### Test Data Builders (for complex scenarios)
```csharp
public class EntityBuilder
{
    private readonly Fixture _fixture = new();
    private string _name = "Default";
    private EntityStatus _status = EntityStatus.Active;

    public EntityBuilder WithName(string name)
    {
        _name = name;
        return this;
    }

    public EntityBuilder WithStatus(EntityStatus status)
    {
        _status = status;
        return this;
    }

    public Entity Build()
    {
        return _fixture.Build<Entity>()
            .With(x => x.Name, _name)
            .With(x => x.Status, _status)
            .Create();
    }
}
```

## Repository Testing Patterns

### Unit Tests (with mocked DbContext)
```csharp
[Fact]
public async Task GetById_ExistingEntity_ReturnsEntity()
{
    // Arrange
    var entity = _fixture.Create<YourEntity>();
    var mockSet = CreateMockDbSet(new[] { entity });
    var mockContext = _fixture.Freeze<Mock<YourDbContext>>();
    mockContext.Setup(x => x.Set<YourEntity>()).Returns(mockSet.Object);
    var repository = _fixture.Create<YourRepository>();

    // Act
    var result = await repository.GetById(entity.Id);

    // Assert
    result.Should().BeEquivalentTo(entity);
}
```

### Integration Tests (with in-memory database)
```csharp
[Fact]
[Trait("Category", TestCategories.Integration)]
public async Task GetById_ExistingEntity_ReturnsEntity()
{
    // Use real database context with in-memory provider
}
```

## Service Testing Patterns

### Application Service Tests
```csharp
[Fact]
public async Task Create_ValidDto_CallsRepositoryAndReturnsDto()
{
    // Arrange
    var createDto = _fixture.Create<CreateEntityDto>();
    var entity = _fixture.Create<YourEntity>();
    var mockRepository = _fixture.Freeze<Mock<IEntityRepository>>();
    mockRepository.Setup(x => x.AddAsync(It.IsAny<YourEntity>()))
        .ReturnsAsync(entity);
    var sut = _fixture.Create<EntityService>();

    // Act
    var result = await sut.CreateAsync(createDto);

    // Assert
    result.Should().NotBeNull();
    result.Should().BeOfType<EntityDto>();
    mockRepository.Verify(x => x.AddAsync(It.Is<YourEntity>(e => 
        e.Name == createDto.Name)), Times.Once);
}
```

## Domain Entity Testing

### Value Object Tests
```csharp
[Fact]
public void Create_ValidValue_ReturnsSuccessResult()
{
    // Arrange
    var value = "valid-value";

    // Act
    var result = ValueObject.Create(value);

    // Assert
    result.IsSuccess.Should().BeTrue();
    result.Value.Should().NotBeNull();
    result.Value.Value.Should().Be(value);
}

[Fact]
public void Create_InvalidValue_ReturnsFailureResult()
{
    // Test validation logic
}
```

### Entity Behavior Tests
```csharp
[Fact]
public void UpdateStatus_ValidTransition_UpdatesStatus()
{
    // Arrange
    var entity = _fixture.Build<YourEntity>()
        .With(x => x.Status, EntityStatus.Draft)
        .Create();

    // Act
    entity.Submit();

    // Assert
    entity.Status.Should().Be(EntityStatus.Submitted);
    entity.SubmittedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(1));
}

[Fact]
public void UpdateStatus_InvalidTransition_ThrowsDomainException()
{
    // Test business rule validation
}
```

## DTO Validation Testing

```csharp
[Fact]
public void Validate_ValidDto_PassesValidation()
{
    // Arrange
    var dto = _fixture.Create<CreateEntityDto>();
    var validator = new CreateEntityDtoValidator();

    // Act
    var result = validator.Validate(dto);

    // Assert
    result.IsValid.Should().BeTrue();
}

[Fact]
public void Validate_MissingRequiredField_FailsValidation()
{
    // Arrange
    var dto = _fixture.Build<CreateEntityDto>()
        .Without(x => x.Name)
        .Create();
    var validator = new CreateEntityDtoValidator();

    // Act
    var result = validator.Validate(dto);

    // Assert
    result.IsValid.Should().BeFalse();
    result.Errors.Should().Contain(e => e.PropertyName == nameof(dto.Name));
}
```

## Async Testing Best Practices

```csharp
// Always use async/await for async methods
[Fact]
public async Task MethodAsync_Scenario_ExpectedResult()
{
    // Arrange
    var sut = _fixture.Create<Service>();

    // Act
    var result = await sut.MethodAsync();

    // Assert
    result.Should().NotBeNull();
}

// Test cancellation
[Fact]
public async Task MethodAsync_CancellationRequested_ThrowsOperationCanceledException()
{
    // Arrange
    var cts = new CancellationTokenSource();
    cts.Cancel();
    var sut = _fixture.Create<Service>();

    // Act
    Func<Task> act = async () => await sut.MethodAsync(cts.Token);

    // Assert
    await act.Should().ThrowAsync<OperationCanceledException>();
}
```

## Common Test Scenarios Checklist

For each method under test, consider:

- [ ] Happy path with valid input
- [ ] Null input
- [ ] Empty/whitespace input
- [ ] Boundary values (min, max, just below, just above)
- [ ] Unicode and special characters
- [ ] SQL injection attempts
- [ ] XSS attempts
- [ ] Unauthorized access
- [ ] Role-based permissions
- [ ] Entity not found
- [ ] Duplicate entries
- [ ] Concurrent modifications
- [ ] Cancellation token handling
- [ ] Timeout scenarios
- [ ] Large data sets
- [ ] Invalid state transitions
- [ ] Validation errors
- [ ] Exception handling

## Test Organization

### One Test Class Per Class Under Test
```csharp
// YourService.cs -> YourServiceTests.cs
public class YourServiceTests
{
    // Group related tests with nested classes
    public class CreateAsync
    {
        [Fact]
        public async Task ValidDto_ReturnsCreatedEntity() { }
        
        [Fact]
        public async Task NullDto_ThrowsArgumentNullException() { }
    }

    public class UpdateAsync
    {
        [Fact]
        public async Task ValidDto_ReturnsUpdatedEntity() { }
    }
}
```

### Test Naming Consistency
Follow the pattern: `MethodName_Scenario_ExpectedResult`

Good examples:
- `Create_ValidInput_ReturnsEntityDto`
- `GetById_NonExistentId_ThrowsNotFoundException`
- `Update_ConcurrentUpdate_ThrowsConcurrencyException`

## CI/CD Integration

Tests should:
- Run automatically on every PR
- Block merges if tests fail
- Generate coverage reports
- Run quickly (unit tests < 5 seconds total)
- Be deterministic (no flaky tests)

## Additional Guidelines

### Test Independence
- Each test should be independent and able to run in isolation
- Use `[Fact]` for single tests, `[Theory]` for parameterized tests
- Clean up resources in Dispose if needed

### Test Readability
- Use descriptive variable names
- Keep tests focused on a single behavior
- Add comments only when business logic is complex
- Use helper methods to reduce duplication

### Test Maintainability
- Don't test implementation details, test behavior
- Avoid testing private methods directly
- Keep mocks simple and focused
- Refactor tests when refactoring production code

### Performance
- Unit tests should be fast (milliseconds)
- Use async properly to avoid deadlocks
- Avoid Thread.Sleep, use Task.Delay with CancellationToken
- Mock external dependencies

## Example Complete Test Class

```csharp
using AutoFixture;
using AutoFixture.AutoMoq;
using Shouldly;
using Moq;
using Xunit;

namespace YourProject.Tests.Services
{
    public class YourServiceTests
    {
        private readonly IFixture _fixture;
        private readonly Mock<IYourRepository> _mockRepository;
        private readonly YourService _sut;

        public YourServiceTests()
        {
            _fixture = new Fixture()
                .Customize(new AutoMoqCustomization { ConfigureMembers = true });
            
            _mockRepository = _fixture.Freeze<Mock<IYourRepository>>();
            _sut = _fixture.Create<YourService>();
        }

        [Fact]
        [Trait("Category", TestCategories.Unit)]
        public async Task CreateAsync_ValidDto_ReturnsEntityDto()
        {
            // Arrange
            var createDto = _fixture.Create<CreateEntityDto>();
            var entity = _fixture.Create<YourEntity>();
            _mockRepository.Setup(x => x.AddAsync(It.IsAny<YourEntity>()))
                .ReturnsAsync(entity);

            // Act
            var result = await _sut.CreateAsync(createDto);

            // Assert
            result.ShouldNotBeNull();
            result.ShouldBeOfType<EntityDto>();
            _mockRepository.Verify(x => x.AddAsync(It.IsAny<YourEntity>()), Times.Once);
        }

        [Fact]
        public async Task CreateAsync_NullDto_ThrowsArgumentNullException()
        {
            // Act
            Func<Task> act = async () => await _sut.CreateAsync(null);

            // Assert
            await act.ShouldThrowAsync<ArgumentNullException>();
        }

        [Theory]
        [InlineData("Test 🚀")]
        [InlineData("Test 你好")]
        [InlineData("Test <>?*")]
        public async Task CreateAsync_UnicodeInput_HandlesCorrectly(string name)
        {
            // Arrange
            var createDto = _fixture.Build<CreateEntityDto>()
                .With(x => x.Name, name)
                .Create();
            var entity = _fixture.Create<YourEntity>();
            _mockRepository.Setup(x => x.AddAsync(It.IsAny<YourEntity>()))
                .ReturnsAsync(entity);

            // Act
            var result = await _sut.CreateAsync(createDto);

            // Assert
            result.ShouldNotBeNull();
        }

        [Theory]
        [Trait("Category", TestCategories.Security)]
        [InlineData("'; DROP TABLE Entities; --")]
        [InlineData("1' OR '1'='1")]
        public async Task SearchAsync_SqlInjectionAttempt_DoesNotExecuteInjection(string searchTerm)
        {
            // Arrange
            _mockRepository.Setup(x => x.SearchAsync(It.IsAny<string>()))
                .ReturnsAsync(new List<YourEntity>());

            // Act
            var result = await _sut.SearchAsync(searchTerm);

            // Assert
            result.ShouldNotBeNull();
            _mockRepository.Verify(x => x.SearchAsync(searchTerm), Times.Once);
        }
    }
}
```

---

## Quick Reference Summary

1. **Always use** xUnit, Moq, AutoFixture, and Shouldly
2. **Follow AAA pattern** (Arrange-Act-Assert)
3. **Name tests** as `MethodName_Scenario_ExpectedResult`
4. **Test both** happy paths and unhappy paths
5. **Include security tests** (SQL injection, XSS, RBAC)
6. **Test input validation** (null, empty, unicode, boundaries)
7. **Aim for 80%+ code coverage**
8. **Keep tests independent** and fast
9. **Use AutoFixture** for test data generation
10. **Verify mock interactions** with Shouldly

---

This guide should be used as the primary reference when generating any unit tests in a .NET solution.
