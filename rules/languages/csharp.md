---
paths:
  - "**/*.cs"
  - "**/*.csproj"
  - "**/Directory.Build.props"
  - "**/*.sln"
---

# C# Standards

- **Runtime**: .NET 10 LTS (Nov 2025, supported through Nov 2028) or .NET 9
- **C# Version**: 14 (single-file scripting) or 13 (Lock type, params collections)
- **Web**: ASP.NET Core Minimal APIs (preferred) or Controllers
- **Database**: EF Core 9+ or Dapper
- **Validation**: FluentValidation
- **Error Handling**: FluentResults (Result pattern)
- **Testing**: xUnit + NSubstitute + FluentAssertions
- **Observability**: OpenTelemetry + Serilog

## Core Requirements

- **MUST** use nullable reference types (`<Nullable>enable</Nullable>`)
- **MUST** use file-scoped namespaces
- **MUST** use global usings to reduce clutter
- **MUST** use `async/await` for I/O operations
- **MUST** use dependency injection
- **MUST** use `throw;` not `throw ex;` in catch blocks (preserves stack trace)
- **MUST** return `async Task` not `async void` (void crashes process on exception)
- **MUST NOT** use `dynamic` without justification
- **MUST NOT** use sync-over-async (`.Result`, `.Wait()`) - causes thread pool starvation
- **SHOULD** use records for DTOs and value objects (immutable, value equality)
- **SHOULD** use primary constructors (C# 12)
- **SHOULD** use Minimal APIs for microservices (faster than Controllers)
- **SHOULD** use `IAsyncEnumerable<T>` for streaming large datasets
- **SHOULD** use `Span<T>`/`Memory<T>` for parsing to avoid heap allocations
- **SHOULD** use AOT compilation for serverless/Lambda (eliminates cold starts)
- **SHOULD** start with modular monolith, not microservices (90% of the time)
- **SHOULD** use CQRS pattern - separate read/write operations for scalability

## Project File

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

## C# 13/14 Features

```csharp
// Primary constructors (C# 12+)
public class UserService(IUserRepository repository, ILogger<UserService> logger)
{
    public async Task<User?> GetUserAsync(string id)
    {
        logger.LogInformation("Getting user {UserId}", id);
        return await repository.FindAsync(id);
    }
}

// Collection expressions (C# 12+)
int[] numbers = [1, 2, 3, 4, 5];
List<string> names = ["Alice", "Bob"];
int[] combined = [..first, ..second, 99];

// System.Threading.Lock (C# 13) - better than object locking
private readonly Lock _lock = new();
lock (_lock) { /* thread-safe code */ }

// params with any collection type (C# 13)
void Log(params ReadOnlySpan<string> messages) { }

// Task.WhenEach (C# 13) - iterate as tasks complete
await foreach (var task in Task.WhenEach(tasks))
{
    var result = await task;
}

// Single-file scripting (C# 14)
// dotnet run app.cs - no .csproj needed
```

## Modern Patterns

```csharp
// Records for immutable data
public record User(string Id, string Email, UserRole Role);

public record CreateUserRequest(
    [Required] string Email,
    [Required] string Name
);

// Result pattern with FluentResults (no exceptions for expected failures)
// Install: dotnet add package FluentResults
using FluentResults;

public Result<User> GetUser(string id)
{
    var user = _db.Find(id);
    if (user is null)
        return Result.Fail<User>($"User {id} not found");
    return Result.Ok(user);
}

// Usage
var result = GetUser(id);
if (result.IsFailed) return Results.NotFound(result.Errors);
return Results.Ok(result.Value);

// Extension methods for fluent APIs
public static class ResultExtensions
{
    public static Result<U> Map<T, U>(this Result<T> result, Func<T, U> map) =>
        result switch
        {
            Result<T>.Success(var value) => new Result<U>.Success(map(value)),
            Result<T>.Failure(var error) => new Result<U>.Failure(error),
            _ => throw new InvalidOperationException()
        };
}

// Structured logging with Serilog
Log.Information("User {UserId} created with email {Email}", user.Id, user.Email);
```

## Minimal API Pattern

```csharp
var builder = WebApplication.CreateBuilder(args);

// Services
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddOpenTelemetry()
    .WithTracing(b => b.AddAspNetCoreInstrumentation());

var app = builder.Build();

// Endpoints
app.MapGet("/users/{id}", async (string id, IUserService service) =>
{
    var user = await service.GetUserAsync(id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});

app.MapPost("/users", async (CreateUserRequest request, IUserService service) =>
{
    var user = await service.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});

app.Run();
```

## Error Handling

```csharp
// Custom exceptions for domain errors
public class NotFoundException(string resource, string id)
    : Exception($"{resource} not found: {id}")
{
    public string Resource { get; } = resource;
    public string Id { get; } = id;
}

// Global exception handler
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;

        var (statusCode, message) = exception switch
        {
            NotFoundException => (404, exception.Message),
            ValidationException => (400, exception.Message),
            _ => (500, "An error occurred")
        };

        context.Response.StatusCode = statusCode;
        await context.Response.WriteAsJsonAsync(new { error = message });
    });
});
```

## Anti-patterns

```csharp
// ❌ Bad
public string Name { get; set; }     // Nullable without annotation
var data = (dynamic)obj;              // Dynamic typing
Task.Run(() => DoWork()).Wait();      // Sync over async

// ✅ Good
public required string Name { get; init; }
var data = obj as MyType ?? throw new InvalidCastException();
await DoWorkAsync();
```

## Testing

```csharp
public class UserServiceTests
{
    private readonly IUserRepository _repository = Substitute.For<IUserRepository>();
    private readonly UserService _sut;

    public UserServiceTests()
    {
        _sut = new UserService(_repository, NullLogger<UserService>.Instance);
    }

    [Fact]
    public async Task GetUser_WhenExists_ReturnsUser()
    {
        // Arrange
        var user = new User("123", "test@example.com", UserRole.User);
        _repository.FindAsync("123").Returns(user);

        // Act
        var result = await _sut.GetUserAsync("123");

        // Assert
        result.Should().BeEquivalentTo(user);
    }
}
```
