# Testing

## What it is
Automated verification that code works as expected. Tests catch regressions, document behavior, and enable confident refactoring.

## Why it matters
Without tests, every change is a risk. With good tests, you can refactor fearlessly, deploy confidently, and onboard new developers faster.

---

## The Test Pyramid

```
        /\
       /  \   E2E Tests (few, slow, brittle)
      /----\
     /      \ Integration Tests (some, moderate speed)
    /--------\
   /          \ Unit Tests (many, fast, isolated)
  /____________\
```

- **Unit tests:** Test a single class/function in isolation. Mock all dependencies. Fast (milliseconds).
- **Integration tests:** Test multiple components together (e.g., command handler + real database). Slower but more confidence.
- **E2E tests:** Test the whole system from the outside (HTTP request to DB and back).

---

## Unit Testing with xUnit + Moq

### Setup
```bash
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Moq
dotnet add package FluentAssertions
```

### Basic Test Structure (AAA Pattern)
```csharp
public class CreateOrderCommandHandlerTests
{
    private readonly Mock<IUnitOfWork> _uowMock;
    private readonly CreateOrderCommandHandler _handler;

    public CreateOrderCommandHandlerTests()
    {
        _uowMock = new Mock<IUnitOfWork>();
        _handler = new CreateOrderCommandHandler(_uowMock.Object);
    }

    [Fact]
    public async Task Handle_WhenProductExists_ShouldCreateOrder()
    {
        // Arrange
        var product = Product.Create("Blue Sneakers", Money.Of(89.99m, "USD"), stock: 10);
        var command = new CreateOrderCommand(
            UserId: Guid.NewGuid(),
            ProductId: product.Id,
            Quantity: 2,
            ShippingAddress: new Address("123 Main St", "Sofia", "BG"));

        _uowMock.Setup(u => u.Products.GetByIdAsync(product.Id, It.IsAny<CancellationToken>()))
                .ReturnsAsync(product);
        _uowMock.Setup(u => u.CommitAsync(It.IsAny<CancellationToken>()))
                .ReturnsAsync(1);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeEmpty();
        _uowMock.Verify(u => u.Orders.Add(It.IsAny<Order>()), Times.Once);
        _uowMock.Verify(u => u.CommitAsync(It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task Handle_WhenProductNotFound_ShouldReturnFailure()
    {
        // Arrange
        var command = new CreateOrderCommand(Guid.NewGuid(), Guid.NewGuid(), 1, someAddress);

        _uowMock.Setup(u => u.Products.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
                .ReturnsAsync((Product?)null);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeFalse();
        result.ErrorCode.Should().Be(ErrorCodes.ProductNotFound);
        _uowMock.Verify(u => u.CommitAsync(It.IsAny<CancellationToken>()), Times.Never);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    [InlineData(-100)]
    public async Task Handle_WhenQuantityInvalid_ShouldReturnFailure(int quantity)
    {
        // ... test edge cases
    }
}
```

### Testing Domain Entities (No Mocks Needed)
```csharp
public class OrderTests
{
    [Fact]
    public void AddItem_WhenOrderIsPending_ShouldAddItem()
    {
        var order = Order.Create(Guid.NewGuid(), someAddress);
        var product = Product.Create("Sneakers", Money.Of(89.99m, "USD"), stock: 10);

        var result = order.AddItem(product, quantity: 2);

        result.IsSuccess.Should().BeTrue();
        order.Items.Should().HaveCount(1);
        order.Items.First().Quantity.Should().Be(2);
    }

    [Fact]
    public void AddItem_WhenOrderIsNotPending_ShouldFail()
    {
        var order = Order.Create(Guid.NewGuid(), someAddress);
        order.Place(); // transition to Placed status

        var result = order.AddItem(someProduct, 1);

        result.IsSuccess.Should().BeFalse();
        result.ErrorCode.Should().Be(ErrorCodes.InvalidOrderStatus);
    }
}
```

---

## What to Test (and What Not To)

### Test:
- Domain entity business logic (invariants, state transitions)
- Command handlers (business workflow, error cases)
- Domain services
- Validators (FluentValidation)
- Mapping logic (if complex)

### Don't Test:
- Framework code (DI registration, EF Core itself)
- Simple CRUD with no business logic
- Third-party library behavior
- Auto-generated code
- Private methods directly (test through public API)

**Rule of thumb:** Test behavior, not implementation details. Tests should survive refactoring.

---

## Integration Testing with TestContainers

Run real infrastructure (SQL Server, Redis) in Docker containers during tests.

### Setup
```bash
dotnet add package Testcontainers.MsSql
dotnet add package Microsoft.AspNetCore.Mvc.Testing
```

### Web Application Factory
```csharp
public class IntegrationTestFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly MsSqlContainer _dbContainer = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Replace real DB with test container
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(opts =>
                opts.UseSqlServer(_dbContainer.GetConnectionString()));
        });
    }

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();
        // Run migrations
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public async Task DisposeAsync() => await _dbContainer.StopAsync();
}
```

### Integration Test
```csharp
public class OrdersIntegrationTests : IClassFixture<IntegrationTestFactory>
{
    private readonly HttpClient _client;

    public OrdersIntegrationTests(IntegrationTestFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task CreateOrder_WithValidData_ShouldReturn201()
    {
        // Arrange
        var request = new CreateOrderRequest { ProductId = seededProductId, Quantity = 2, ... };

        // Act
        var response = await _client.PostAsJsonAsync("/api/v1/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var body = await response.Content.ReadFromJsonAsync<ApiResponse<Guid>>();
        body!.Success.Should().BeTrue();
        body.Data.Should().NotBeEmpty();
    }
}
```

---

## Test Data Builders

Avoid repetitive test setup with builder pattern.

```csharp
public class OrderBuilder
{
    private Guid _userId = Guid.NewGuid();
    private Address _address = new Address("123 Main St", "Sofia", "BG");
    private List<(Product product, int quantity)> _items = new();

    public OrderBuilder WithUser(Guid userId) { _userId = userId; return this; }
    public OrderBuilder WithItem(Product product, int qty) { _items.Add((product, qty)); return this; }

    public Order Build()
    {
        var order = Order.Create(_userId, _address);
        foreach (var (product, qty) in _items)
            order.AddItem(product, qty);
        return order;
    }
}

// Usage
var order = new OrderBuilder()
    .WithUser(customerId)
    .WithItem(sneakers, 2)
    .Build();
```

---

## FluentAssertions Cheat Sheet

```csharp
// Booleans
result.IsSuccess.Should().BeTrue();
result.IsSuccess.Should().BeFalse();

// Strings
name.Should().Be("expected");
name.Should().StartWith("Ivan");
name.Should().Contain("mid");
name.Should().BeNullOrEmpty();

// Numbers
count.Should().Be(3);
price.Should().BeGreaterThan(0);
price.Should().BeInRange(1, 100);

// Collections
list.Should().HaveCount(3);
list.Should().ContainSingle();
list.Should().BeEmpty();
list.Should().Contain(item => item.Id == expectedId);
list.Should().BeEquivalentTo(expected); // deep equality

// Exceptions
action.Should().Throw<DomainException>()
      .WithMessage("*cannot be negative*");

await asyncAction.Should().ThrowAsync<InvalidOperationException>();

// Objects
order.Should().NotBeNull();
order.Should().BeEquivalentTo(expected, opts => opts.Excluding(o => o.CreatedAt));
```

---

## Common Interview Questions

1. What is the test pyramid and why does it matter?
2. What is the difference between unit and integration tests?
3. What is AAA (Arrange/Act/Assert)?
4. What should you mock in unit tests?
5. What is TestContainers and why is it better than in-memory databases for integration tests?
6. What is a test data builder?

---

## Common Mistakes

- Testing implementation details instead of behavior (tests break on every refactor)
- Over-mocking (mocking everything, including value objects)
- Using in-memory EF Core for integration tests (doesn't validate real SQL, missing constraints)
- Not resetting DB state between tests (flaky tests due to test pollution)
- Writing tests after the fact with no actual coverage of edge cases

---

## How It Connects

- Unit tests cover domain entities and command handlers
- Integration tests cover the full HTTP → handler → DB → response cycle
- TestContainers uses Docker — same containers as production deployment
- FluentAssertions + AAA = readable tests that document behavior
- Test data builders use the Builder pattern from GoF

---

## My Confidence Level
- `[b]` Unit testing with xUnit
- `[b]` Mocking with Moq
- `[~]` Integration testing with TestContainers
- `[~]` What to test and what not to test
- `[ ]` Frontend testing (React Testing Library)

## My Notes
<!-- Personal notes -->
