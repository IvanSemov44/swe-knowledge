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

## Choosing the Right Test Type

Ask one question before writing any test:

> **"What external systems does this behavior require to exist?"**

| Code | External systems needed | Test type |
|---|---|---|
| `Order.AddItem()` rejects wrong status | None — pure domain logic | Unit (no mocks) |
| `CreateOrderCommandHandler` | `IUnitOfWork` — mockable interface | Unit (Moq) |
| `POST /api/orders` → DB → response | Full HTTP stack + real database | Integration |
| Full user checkout flow | Everything | E2E (rare) |

**The rule:**
- If you can replace a dependency with a fake interface → **unit test with Moq**
- If you need real SQL behavior (constraints, migrations, transactions) → **integration test with TestContainers**

**In your e-commerce project:**
```
Order.AddItem()                    → Unit (pure domain logic, no mocks)
CreateOrderCommandHandler          → Unit (mock IUnitOfWork)
POST /api/orders → DB → response   → Integration (TestContainers)
Full checkout flow                 → E2E (rarely written)
```

---

## Unit Testing with xUnit + Moq

### Setup
```bash
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Moq
dotnet add package FluentAssertions
```

### AAA Pattern — Arrange / Act / Assert

Every test has exactly three sections separated by a blank line. This is non-negotiable.

```csharp
public class CreateOrderCommandHandlerTests
{
    private readonly Mock<IUnitOfWork> _uowMock = new();
    private readonly CreateOrderCommandHandler _handler;

    public CreateOrderCommandHandlerTests()
    {
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
        _uowMock.Setup(u => u.Products.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
                .ReturnsAsync((Product?)null);
        var command = new CreateOrderCommand(Guid.NewGuid(), Guid.NewGuid(), 1, someAddress);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeFalse();
        result.ErrorCode.Should().Be(ErrorCodes.ProductNotFound);
        _uowMock.Verify(u => u.CommitAsync(It.IsAny<CancellationToken>()), Times.Never);
    }
}
```

### [Fact] vs [Theory]

`[Fact]` = one test, one scenario.
`[Theory]` = one test, multiple data sets — xUnit runs it once per row.

Use `[Theory]` when the same rule must hold across multiple inputs, especially invalid/edge cases.

```csharp
// [InlineData] — for primitive inputs
[Theory]
[InlineData(0)]
[InlineData(-1)]
[InlineData(-999)]
public async Task Handle_WhenQuantityIsInvalid_ShouldReturnFailure(int quantity)
{
    // Arrange
    var command = new CreateOrderCommand(Guid.NewGuid(), Guid.NewGuid(), quantity, someAddress);

    // Act
    var result = await _handler.Handle(command, CancellationToken.None);

    // Assert
    result.IsSuccess.Should().BeFalse();
    result.ErrorCode.Should().Be(ErrorCodes.InvalidQuantity);
}

// [MemberData] — when inputs are complex or you want to vary the expected outcome too
public static IEnumerable<object[]> InvalidQuantities => new[]
{
    new object[] { 0,     ErrorCodes.InvalidQuantity },
    new object[] { -1,    ErrorCodes.InvalidQuantity },
    new object[] { 9999,  ErrorCodes.ExceedsStockLimit },
};

[Theory]
[MemberData(nameof(InvalidQuantities))]
public async Task Handle_WhenQuantityIsInvalid_ShouldReturnCorrectError(
    int quantity, string expectedError)
{
    // Arrange
    var command = new CreateOrderCommand(Guid.NewGuid(), Guid.NewGuid(), quantity, someAddress);

    // Act
    var result = await _handler.Handle(command, CancellationToken.None);

    // Assert
    result.IsSuccess.Should().BeFalse();
    result.ErrorCode.Should().Be(expectedError);
}
```

### xUnit Attributes Cheat Sheet

| Attribute | Use |
|---|---|
| `[Fact]` | Single scenario |
| `[Theory]` + `[InlineData]` | Multiple primitive inputs |
| `[Theory]` + `[MemberData]` | Multiple complex inputs or varied expected outcomes |
| `IClassFixture<T>` | Shared setup across all tests in one class (e.g., shared DB container) |
| `IAsyncLifetime` | Async setup/teardown via `InitializeAsync` / `DisposeAsync` |

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

**The rule:** Unit test your logic. Integration test your configuration.

| Code | Test type | Why |
|---|---|---|
| `Order.AddItem()`, `Order.Place()` | Unit | Pure domain logic, no deps |
| `Money.Add()`, value object equality | Unit | Pure logic |
| `CreateOrderCommandHandler` | Unit | Logic, deps are mockable |
| FluentValidation rules | Unit | Logic, fast |
| `ProductRepository.GetByIdAsync()` | None / Integration | Delegates entirely to EF Core |
| Global query filters (`!p.IsDeleted`) | Integration | Only activates against real DB |
| EF Core mappings / migrations | Integration | Requires real SQL engine |
| Controllers (thin) | None | No logic to test |
| DI registration | None | Framework concern |

**Never test:**
- Private methods directly — test through the public API that calls them
- Third-party library behavior (EF Core, MediatR internals)
- Auto-generated code

**Key insight on repositories:** If `GetByIdAsync` is `return await _context.Products.FindAsync(id)`, there is no logic — don't unit test it. BUT if you have a global query filter like `HasQueryFilter(p => !p.IsDeleted)`, that filter only activates against a real DB. A mocked context won't catch whether the filter is applied. That's integration test territory.

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

---

## Test Doubles — Mock vs Stub vs Spy vs Fake vs Dummy

These terms are often used interchangeably in conversation ("just mock it"), but they mean different things. Interviews test whether you know the distinctions.

| Double | What it does | When to use |
|---|---|---|
| **Dummy** | Passed around but never used. Fills a required parameter. | Constructor parameters you don't care about in this test |
| **Stub** | Returns a pre-configured answer to calls. No verification. | When you need to control what a dependency returns |
| **Mock** | Verifies that specific calls were made with specific arguments. | When the test cares THAT something was called (not just what it returned) |
| **Spy** | A real object that records what was called. Verify after. | When you want real behavior but also want to assert calls |
| **Fake** | A working implementation, simpler than production. | In-memory repository, in-memory queue |

```csharp
// STUB — control what the dependency returns, don't verify calls
_uowMock.Setup(u => u.Products.GetByIdAsync(id, It.IsAny<CancellationToken>()))
        .ReturnsAsync(product); // just return this, don't care about verification

// MOCK — verify the interaction happened
_uowMock.Verify(u => u.CommitAsync(It.IsAny<CancellationToken>()), Times.Once);
// Fails the test if CommitAsync was not called exactly once

// FAKE — in-memory implementation used instead of real DB
public class InMemoryOrderRepository : IOrderRepository
{
    private readonly List<Order> _orders = new();
    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => Task.FromResult(_orders.FirstOrDefault(o => o.Id == id));
    public void Add(Order order) => _orders.Add(order);
}

// DUMMY — don't care about this parameter
var dummyCancellationToken = CancellationToken.None; // not relevant to this test

// SPY (with Moq — CallBase + verify)
var spy = new Mock<IOrderRepository> { CallBase = true };
// ... use spy ...
spy.Verify(r => r.Add(It.IsAny<Order>()), Times.Once);
```

**Key rule:** Mock behaviors you own, don't mock external libraries directly.
**Key rule:** Verify behavior (MOCK) only when the call itself IS the outcome. If the outcome is the return value, use a STUB.

---

## React Testing Library

Test React components the way a user would interact with them — by querying the DOM, not component internals.

### Setup

```bash
npm install --save-dev @testing-library/react @testing-library/user-event @testing-library/jest-dom
```

### Core Principles

```typescript
// WRONG — testing implementation details
expect(component.state.isLoading).toBe(false);
expect(wrapper.find('Button').props().onClick).toBeDefined();

// RIGHT — testing what the user sees and interacts with
expect(screen.getByText('Add to Cart')).toBeInTheDocument();
await userEvent.click(screen.getByRole('button', { name: /add to cart/i }));
```

### Queries (priority order — use highest priority first)

```typescript
// 1. Role — most accessible query (tests a11y too)
screen.getByRole('button', { name: /submit/i });
screen.getByRole('heading', { name: /products/i });
screen.getByRole('textbox', { name: /email/i });

// 2. Label text — for form inputs
screen.getByLabelText(/email address/i);

// 3. Placeholder text
screen.getByPlaceholderText(/search products/i);

// 4. Text content
screen.getByText(/order confirmed/i);

// 5. Test ID — last resort (implementation detail)
screen.getByTestId('cart-count'); // avoid unless necessary
```

**getBy vs queryBy vs findBy:**
```typescript
getByRole('button')    // throws if not found — use when element must exist
queryByRole('button')  // returns null if not found — use to assert absence
findByRole('button')   // async, waits for element — use for async rendering
```

### Example: Testing a Form

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('calls onSubmit with email and password when form is submitted', async () => {
    const onSubmit = jest.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    // Arrange — fill the form
    await userEvent.type(
      screen.getByLabelText(/email/i),
      'ivan@example.com'
    );
    await userEvent.type(
      screen.getByLabelText(/password/i),
      'secret123'
    );

    // Act — submit
    await userEvent.click(screen.getByRole('button', { name: /sign in/i }));

    // Assert — check what the user sees
    expect(onSubmit).toHaveBeenCalledWith({
      email: 'ivan@example.com',
      password: 'secret123',
    });
  });

  it('shows validation error when email is empty', async () => {
    render(<LoginForm onSubmit={jest.fn()} />);
    await userEvent.click(screen.getByRole('button', { name: /sign in/i }));
    expect(screen.getByText(/email is required/i)).toBeInTheDocument();
  });
});
```

### Testing Async Components (RTK Query / loading states)

```typescript
import { renderWithProviders } from '../test-utils'; // custom render with Redux store
import { server } from '../mocks/server'; // MSW mock server
import { rest } from 'msw';

it('shows products after loading', async () => {
  renderWithProviders(<ProductList />);

  // Loading state
  expect(screen.getByRole('progressbar')).toBeInTheDocument();

  // Wait for products to appear
  await screen.findAllByRole('article'); // findBy = async wait
  expect(screen.getAllByRole('article')).toHaveLength(3);
});

it('shows error message when API fails', async () => {
  server.use(
    rest.get('/api/products', (req, res, ctx) =>
      res(ctx.status(500)))
  );

  renderWithProviders(<ProductList />);
  expect(await screen.findByText(/something went wrong/i)).toBeInTheDocument();
});
```

### What NOT to Test in React

- Component implementation details (state, props structure, method calls)
- Styling/CSS classes directly
- Third-party library internals (RTK Query, React Router internals)
- Snapshot tests that auto-update — they just confirm the last render, not behavior

---

## Common Interview Questions

1. What is the test pyramid and why does it matter?
2. What is the difference between unit and integration tests?
3. What is AAA (Arrange/Act/Assert)?
4. What should you mock in unit tests?
5. What is TestContainers and why is it better than in-memory databases for integration tests?
6. What is a test data builder?
7. What is the difference between a Mock and a Stub?
8. What is the difference between `getBy`, `queryBy`, and `findBy` in React Testing Library?
9. Why does React Testing Library encourage querying by role over test IDs?

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
