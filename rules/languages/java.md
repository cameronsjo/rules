---
globs: ["**/*.java", "**/pom.xml", "**/build.gradle", "**/build.gradle.kts"]
alwaysApply: false
---

# Java Standards

**Runtime:** Java 21 LTS | **Framework:** Spring Boot 3.3+ | **Build:** Gradle 8.x (Kotlin DSL)

## Patterns

```java
// Records for immutable data (16+)
public record User(String id, String name, String email, Instant createdAt) {
    public User {
        Objects.requireNonNull(id, "id must not be null");
    }
}

// Pattern matching with switch (21+)
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "positive: " + i;
        case String s -> "string: " + s;
        case null -> "null value";
        default -> "unknown";
    };
}

// Virtual threads for I/O (21+)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var futures = users.stream()
        .map(user -> executor.submit(() -> processUser(user)))
        .toList();
}

// Structured logging
logger.info("Fetching user: userId={}", userId);
```

## Anti-patterns

```java
// ❌ Mutable POJO, string concat logging, platform threads for I/O
public class User { private String name; public void setName(String n) { name = n; } }
logger.info("Processing " + user.getName());
ExecutorService executor = Executors.newFixedThreadPool(100);

// ✅ Immutable record, parameterized logging, virtual threads
public record User(String name) {}
logger.info("Processing user: name={}", user.name());
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```
