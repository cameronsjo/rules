---
globs: ["**/*.cs"]
alwaysApply: false
---

# C# Standards

**Runtime:** .NET 9 (or .NET 8 LTS) | **Language:** C# 12+ | **Web:** ASP.NET Core Minimal APIs

## Project Config

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

## Patterns

```csharp
// Primary constructors + file-scoped namespaces (C# 12)
namespace MyApp.Services;

public class UserService(IUserRepository repository, ILogger<UserService> logger)
{
    public async Task<User?> GetUserAsync(Ulid id, CancellationToken ct = default)
    {
        logger.LogInformation("Fetching user {UserId}", id);
        return await repository.FindByIdAsync(id, ct);
    }
}

// Records for immutable data
public sealed record User(Ulid Id, string Email, string Name, DateTimeOffset CreatedAt);

// Collection expressions (C# 12)
List<string> names = ["Alice", "Bob", "Charlie"];

// Pattern matching
public static string GetStatus(OrderStatus status) => status switch
{
    OrderStatus.Pending => "Processing",
    OrderStatus.Shipped => "On the way",
    _ => throw new UnreachableException()
};
```

## Anti-patterns

```csharp
// ❌ Blocking async, interpolated logging, catching Exception
var user = _repository.GetUserAsync(id).Result;  // Deadlock risk!
logger.LogInformation($"Getting user {id}");     // No structured logging
catch (Exception) { return null; }               // Swallows all errors

// ✅ Async all the way, structured logging, specific exceptions
var user = await _repository.GetUserAsync(id, ct);
logger.LogInformation("Getting user {UserId}", id);
catch (OperationCanceledException) { throw; }
```
