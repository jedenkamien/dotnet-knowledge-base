# Using Statements

## Purpose
Ensures `IDisposable` objects are properly disposed, even if exceptions occur. Syntactic sugar for try-finally blocks.

## Traditional Syntax (C# 1.0+)

```csharp
using (var stream = new FileStream("data.txt", FileMode.Open)) {
    // Use stream
}  // stream.Dispose() called automatically
```

**Equivalent to:**
```csharp
FileStream stream = new FileStream("data.txt", FileMode.Open);
try {
    // Use stream
} finally {
    stream?.Dispose();
}
```

## Using Declaration (C# 8+)

```csharp
using var stream = new FileStream("data.txt", FileMode.Open);
// Use stream
// No braces needed
// stream.Dispose() called at end of enclosing scope (method/block)
```

**Benefits:**
- Less nesting
- Cleaner for multiple disposables
- Disposal at scope end (not immediately)

## Multiple Using Statements

**Nested (traditional):**
```csharp
using (var stream = new FileStream("data.txt", FileMode.Open)) {
    using (var reader = new StreamReader(stream)) {
        string content = reader.ReadToEnd();
    }  // reader disposed
}  // stream disposed
```

**Stacked (traditional):**
```csharp
using (var stream = new FileStream("data.txt", FileMode.Open))
using (var reader = new StreamReader(stream)) {
    string content = reader.ReadToEnd();
}  // Both disposed in reverse order
```

**Modern (C# 8+):**
```csharp
using var stream = new FileStream("data.txt", FileMode.Open);
using var reader = new StreamReader(stream);
string content = reader.ReadToEnd();
// Both disposed at method end (reverse order)
```

## Async Using (C# 8+)

For `IAsyncDisposable` types:

```csharp
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync();
// Use connection
// DisposeAsync() called at end of scope
```

## Pattern Matching with Using (C# 8+)

```csharp
using var stream = File.Exists(path) 
    ? new FileStream(path, FileMode.Open) 
    : null;

if (stream != null) {
    // Use stream
}
```

## Common Scenarios

### File Operations
```csharp
using var writer = new StreamWriter("output.txt");
writer.WriteLine("Hello, World!");
// File automatically closed and flushed
```

### Database Connections
```csharp
using var connection = new SqlConnection(connectionString);
using var command = new SqlCommand(sql, connection);
connection.Open();
var result = command.ExecuteScalar();
// Connection closed automatically
```

### HTTP Requests
```csharp
using var client = new HttpClient();
var response = await client.GetAsync("https://api.example.com");
var content = await response.Content.ReadAsStringAsync();
// Client disposed
```

### Transaction Management
```csharp
using var transaction = dbContext.Database.BeginTransaction();
try {
    dbContext.SaveChanges();
    transaction.Commit();
} catch {
    transaction.Rollback();
    throw;
}
```

## Null Handling

**Using with null** (safe):
```csharp
Stream? stream = GetStreamOrNull();
using (stream) {
    // If stream is null, no Dispose() call, no exception
}
```

## Using with Structs (C# 8+)

Any type with a `Dispose()` method works (not just `IDisposable`):

```csharp
ref struct FileHandle {  // Stack-only struct
    public void Dispose() { /* cleanup */ }
}

using var handle = new FileHandle();
// Dispose() called without IDisposable interface
```

## Best Practices

✅ **Do:**
- Always use `using` for `IDisposable` objects
- Prefer `using` declarations for cleaner code
- Use `await using` for `IAsyncDisposable`

❌ **Don't:**
- Manually call `Dispose()` when `using` will do
- Return `using` variables from methods (disposed too early)
- Use `using` with shared/static objects

## Edge Cases

**Invalid: Can't return using variable**
```csharp
using var stream = new FileStream("data.txt", FileMode.Open);
return stream;  // BAD: disposed at end of method before return
```

**Valid: Return without using**
```csharp
var stream = new FileStream("data.txt", FileMode.Open);
return stream;  // Caller responsible for disposal
```

## Choosing Syntax

| Scenario | Syntax | Reason |
|----------|--------|--------|
| Single resource, short scope | Traditional `using ()` | Clear disposal point |
| Multiple resources | Modern `using var` | Less nesting |
| Need precise disposal timing | Traditional `using ()` | Dispose at brace |
| Return resource | Neither | Caller disposes |

## Real-World Example

```csharp
public async Task<string> ReadFileAsync(string path) {
    using var stream = new FileStream(path, FileMode.Open);
    using var reader = new StreamReader(stream);
    return await reader.ReadToEndAsync();
    // Both disposed after return
}
```

## Summary

`using` ensures cleanup of resources implementing `IDisposable`. Modern C# 8+ syntax (`using var`) reduces nesting and improves readability. Always use `using` instead of manual `Dispose()` calls.
