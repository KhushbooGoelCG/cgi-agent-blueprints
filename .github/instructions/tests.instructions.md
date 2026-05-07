---
applyTo: ["tests/Banking.Application.Tests/**", "tests/Banking.Api.Tests/**"]
---

# Banking Test Projects — Copilot Instructions

## Project Overview

This document provides guidelines for test projects in the Banking solution:

- **Banking.Application.Tests** — Unit tests for services and repositories
- **Banking.Api.Tests** — Unit tests for API controllers

Both projects target **.NET 10** and use **xUnit** as the testing framework with **Moq** for mocking.

---

## Solution Structure

```
tests/
├── Banking.Api.Tests/              # Unit tests for Banking.Api
│   └── Controllers/                # Controller tests
│       └── [Entity]ControllerTests.cs
├── Banking.Application.Tests/      # Unit tests for Banking.Application
│   ├── Services/                   # Service tests
│   │   └── [Entity]ServiceTests.cs
│   └── Repositories/               # Repository tests (if needed)
│       └── [Entity]RepositoryTests.cs
```

---

## Testing Framework & Tools

- **Test Framework**: xUnit 2.9+
- **Mocking Framework**: Moq 4.20+
- **Coverage Tool**: coverlet.collector
- **Test Runner**: Microsoft.NET.Test.Sdk 17.14+

---

## General Testing Principles

### 1. Test Class Naming

Test classes must be named after the class under test with a `Tests` suffix:
- Service test: `ContactServiceTests`
- Controller test: `ContactControllerTests`
- Repository test: `ContactRepositoryTests`

### 2. Test Method Naming

Use the pattern: `MethodName_Scenario_ExpectedBehavior`

**Examples:**
```csharp
GetAllAsync_ReturnsMappedDtos()
GetByIdAsync_ReturnsNullWhenNotFound()
Create_ReturnsUnprocessableEntity_WhenValidationFails()
```

### 3. AAA Pattern (Arrange-Act-Assert)

Always structure tests using the AAA pattern with clear section comments:

```csharp
[Fact]
public async Task MethodName_Scenario_ExpectedBehavior()
{
    // Arrange
    // ... setup mocks and test data

    // Act
    // ... call the method under test

    // Assert
    // ... verify expected behavior
}
```

### 4. Async Testing

- All test methods testing async code must be `async Task`
- Use `ReturnsAsync()` for mock setups that return `Task<T>`
- Use `await` when calling async methods

---

## Banking.Application.Tests Guidelines

### Service Test Structure

```csharp
namespace Banking.Application.Tests.Services;

public class [Entity]ServiceTests
{
    private readonly Mock<I[Entity]Repository> _mockRepository;
    private readonly [Entity]Service _service;

    public [Entity]ServiceTests()
    {
        _mockRepository = new Mock<I[Entity]Repository>();
        _service = new [Entity]Service(_mockRepository.Object);
    }

    // Test methods here
}
```

### Key Testing Patterns for Services

**1. Test GetAllAsync**
```csharp
[Fact]
public async Task GetAllAsync_ReturnsMappedDtos()
{
    // Arrange
    var mockEntities = new List<[Entity]>
    {
        new [Entity] { Id = 1, Name = "Test1" },
        new [Entity] { Id = 2, Name = "Test2" }
    };
    _mockRepository.Setup(r => r.GetAllAsync()).ReturnsAsync(mockEntities);

    // Act
    var result = await _service.GetAllAsync();

    // Assert
    Assert.NotNull(result);
    var list = result.ToList();
    Assert.Equal(2, list.Count);
    Assert.Equal("Test1", list[0].Name);
}
```

**2. Test GetByIdAsync - Not Found**
```csharp
[Fact]
public async Task GetByIdAsync_ReturnsNullWhenNotFound()
{
    // Arrange
    _mockRepository.Setup(r => r.GetByIdAsync(999)).ReturnsAsync(([Entity])null!);

    // Act
    var result = await _service.GetByIdAsync(999);

    // Assert
    Assert.Null(result);
}
```

**3. Test GetByIdAsync - Found**
```csharp
[Fact]
public async Task GetByIdAsync_ReturnsMappedDtoWhenFound()
{
    // Arrange
    var mockEntity = new [Entity] { Id = 1, Name = "Test" };
    _mockRepository.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(mockEntity);

    // Act
    var result = await _service.GetByIdAsync(1);

    // Assert
    Assert.NotNull(result);
    Assert.Equal(1, result.Id);
    Assert.Equal("Test", result.Name);
}
```

**4. Test CreateAsync**
```csharp
[Fact]
public async Task CreateAsync_MapsToEntityAddsAndReturnsMappedDto()
{
    // Arrange
    var dto = new [Entity]DTO { Name = "Test" };
    var createdEntity = new [Entity] { Id = 1, Name = "Test" };
    
    _mockRepository
        .Setup(r => r.AddAsync(It.IsAny<[Entity]>()))
        .ReturnsAsync(createdEntity);

    // Act
    var result = await _service.CreateAsync(dto);

    // Assert
    Assert.NotNull(result);
    Assert.Equal(1, result.Id);
    Assert.Equal("Test", result.Name);
    
    _mockRepository.Verify(
        r => r.AddAsync(It.Is<[Entity]>(e => e.Name == "Test")), 
        Times.Once);
}
```

### Service Test Rules

1. **Mock the repository** — Services should only depend on repository interfaces
2. **Test DTO↔Entity mapping** — Verify that services correctly map between DTOs and entities
3. **Test business logic** — Verify any business rules implemented in the service
4. **Verify repository calls** — Use `Verify()` to ensure repositories are called correctly
5. **Don't test EF Core** — Don't test database operations; mock the repository instead

---

## Banking.Api.Tests Guidelines

### Controller Test Structure

```csharp
namespace Banking.Api.Tests.Controllers;

public class [Entity]sControllerTests
{
    private readonly Mock<I[Entity]Service> _mockService;
    private readonly Mock<ILogger<[Entity]sController>> _mockLogger;
    private readonly [Entity]sController _controller;

    public [Entity]sControllerTests()
    {
        _mockService = new Mock<I[Entity]Service>();
        _mockLogger = new Mock<ILogger<[Entity]sController>>();
        _controller = new [Entity]sController(_mockService.Object, _mockLogger.Object);
    }

    // Test methods here
}
```

### Key Testing Patterns for Controllers

**1. Test GetAll**
```csharp
[Fact]
public async Task GetAll_ReturnsOkResult_WithListOf[Entity]s()
{
    // Arrange
    var mockItems = new List<[Entity]DTO>
    {
        new [Entity]DTO { Id = 1, Name = "Test1" },
        new [Entity]DTO { Id = 2, Name = "Test2" }
    };
    _mockService.Setup(s => s.GetAllAsync()).ReturnsAsync(mockItems);

    // Act
    var result = await _controller.GetAll();

    // Assert
    var okResult = Assert.IsType<OkObjectResult>(result.Result);
    var returnValue = Assert.IsAssignableFrom<IEnumerable<[Entity]DTO>>(okResult.Value);
    Assert.Equal(2, returnValue.Count());
}
```

**2. Test GetById - Not Found**
```csharp
[Fact]
public async Task GetById_ReturnsNotFound_When[Entity]NotExists()
{
    // Arrange
    int id = 999;
    _mockService.Setup(s => s.GetByIdAsync(id)).ReturnsAsync(([Entity]DTO)null!);

    // Act
    var result = await _controller.GetById(id);

    // Assert
    Assert.IsType<NotFoundResult>(result.Result);
}
```

**3. Test GetById - Success**
```csharp
[Fact]
public async Task GetById_ReturnsOkResult_When[Entity]Exists()
{
    // Arrange
    int id = 1;
    var mockItem = new [Entity]DTO { Id = id, Name = "Test" };
    _mockService.Setup(s => s.GetByIdAsync(id)).ReturnsAsync(mockItem);

    // Act
    var result = await _controller.GetById(id);

    // Assert
    var okResult = Assert.IsType<OkObjectResult>(result.Result);
    var returnValue = Assert.IsType<[Entity]DTO>(okResult.Value);
    Assert.Equal(id, returnValue.Id);
}
```

**4. Test Create - Validation Failure**
```csharp
[Fact]
public async Task Create_ReturnsUnprocessableEntity_WhenValidationFails()
{
    // Arrange
    var dto = new [Entity]DTO(); // Missing required fields

    // Act
    var result = await _controller.Create(dto);

    // Assert
    var unprocessableResult = Assert.IsType<UnprocessableEntityObjectResult>(result.Result);
    Assert.NotNull(unprocessableResult.Value);
}
```

**5. Test Create - Success**
```csharp
[Fact]
public async Task Create_ReturnsCreatedAtAction_WhenValid()
{
    // Arrange
    var dto = new [Entity]DTO
    {
        // All required fields populated
    };

    var createdDto = new [Entity]DTO { Id = 1 };
    _mockService.Setup(s => s.CreateAsync(It.IsAny<[Entity]DTO>())).ReturnsAsync(createdDto);

    // Act
    var result = await _controller.Create(dto);

    // Assert
    var createdResult = Assert.IsType<CreatedAtActionResult>(result.Result);
    Assert.Equal(nameof([Entity]sController.GetById), createdResult.ActionName);
    Assert.Equal(1, createdResult.RouteValues?["id"]);
    var returnValue = Assert.IsType<[Entity]DTO>(createdResult.Value);
    Assert.Equal(1, returnValue.Id);
}
```

### Controller Test Rules

1. **Mock the service** — Controllers should only depend on service interfaces
2. **Test HTTP concerns** — Focus on status codes, response types, and routing
3. **Test validation** — Verify controller validation logic returns appropriate errors
4. **Don't test business logic** — Controllers shouldn't have business logic; mock service responses
5. **Mock ILogger** — Always provide a mocked logger to avoid null reference exceptions
6. **Test ActionResult types** — Use `Assert.IsType<>` to verify exact result types
