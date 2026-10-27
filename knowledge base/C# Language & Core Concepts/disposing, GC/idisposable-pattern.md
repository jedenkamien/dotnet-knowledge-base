# IDisposable Pattern & Resource Cleanup

## Purpose
Provides deterministic cleanup of **unmanaged resources** (files, database connections, sockets) that GC doesn't automatically handle.

## Basic Pattern

```csharp
public class FileHandler : IDisposable {
    private FileStream? fileStream;
    private bool disposed = false;

    public FileHandler(string path) {
        fileStream = new FileStream(path, FileMode.Open);
    }

    public void Dispose() {
        Dispose(true);
        GC.SuppressFinalize(this);  // Skip finalizer
    }

    protected virtual void Dispose(bool disposing) {
        if (disposed) return;
        
        if (disposing) {
            // Free managed resources
            fileStream?.Dispose();
        }
        
        // Free unmanaged resources (if any)
        // e.g., CloseHandle(handle);
        
        disposed = true;
    }

    ~FileHandler() {  // Finalizer (optional)
        Dispose(false);
    }
}
```

## Usage with `using`

**Traditional**
```csharp
using (var handler = new FileHandler("data.txt")) {
    // Use handler
}  // Dispose() called automatically
```

**Modern (C# 8+)**
```csharp
using var handler = new FileHandler("data.txt");
// Use handler
// Dispose() called at end of scope
```

**Manual (not recommended)**
```csharp
var handler = new FileHandler("data.txt");
try {
    // Use handler
} finally {
    handler?.Dispose();
}
```

## What Needs Disposal

✅ **Dispose when using:**
- File streams (`FileStream`, `StreamReader`)
- Network resources (`HttpClient`, `Socket`)
- Database connections (`SqlConnection`, `DbContext`)
- Graphics resources (`Bitmap`, `Brush`)
- Synchronization primitives (`Semaphore`, `Mutex`)
- Any type implementing `IDisposable`

❌ **Don't dispose:**
- Managed-only objects (GC handles them)
- Shared/static instances
- Objects you don't own

## IAsyncDisposable (C# 8+)

For asynchronous cleanup:

```csharp
public class AsyncResource : IAsyncDisposable {
    public async ValueTask DisposeAsync() {
        await FlushAsync();
        Dispose();
    }
}

await using var resource = new AsyncResource();
// DisposeAsync() called at end of scope
```

## Common Patterns

### 1. Chaining Disposal
```csharp
class Container : IDisposable {
    private IDisposable child;
    
    public void Dispose() {
        child?.Dispose();
    }
}
```

### 2. Disposed Object Check
```csharp
public void DoWork() {
    if (disposed)
        throw new ObjectDisposedException(nameof(FileHandler));
    // Safe to use resources
}
```

### 3. Multiple using Statements
```csharp
using var stream = new FileStream("data.txt", FileMode.Open);
using var reader = new StreamReader(stream);
// Both disposed in reverse order
```

## Dispose vs Finalize

| Aspect | Dispose() | Finalizer (~Class) |
|--------|-----------|-------------------|
| When | Explicit call | Before GC collects |
| Speed | Immediate | Delayed (GC dependent) |
| Managed resources | Yes | No |
| Unmanaged resources | Yes | Yes |
| Recommended | Always | Rarely (only if unmanaged) |

**Finalize overhead**: Objects with finalizers promoted to Gen1, delayed collection.

### When Finalizers Run (Unpredictable Timing)

Finalizers execute **during garbage collection**, not immediately when an object becomes unreachable:

```csharp
var handler = new FileHandler("data.txt");
handler.ReadData();
// handler is unreachable here, but file stays OPEN
// Finalizer might run seconds/minutes later during GC
// Blocks other processes from accessing the file
```

**Gen0 → Gen1 Promotion**: Objects with finalizers skip reclamation and get promoted to Gen1 until their finalizer runs. This causes:
- Longer object lifetime in memory
- More expensive Gen1 collections
- Performance penalty for every finalized object

### Decision: Keep or Skip the Finalizer?

**Remove finalizer (~FileHandler) if:**
- Only managed resources (GC handles cleanup automatically)
- Strict enforcement of `using` statements in your codebase
- Performance is critical

**Keep finalizer if:**
- It's a public API where consumers might forget `Dispose()`
- Prevents silent resource leaks in untrusted code
- Safety net worth the Gen1 promotion cost

**Example**: `FileHandler` keeps it (defensive), `SimpleHandler` removes it (disciplined codebase).

## Best Practices

✅ **Do:**
- Always dispose `IDisposable` objects
- Use `using` statements
- Call `Dispose()` on base class in overrides
- Make `Dispose()` idempotent (safe to call multiple times)

❌ **Don't:**
- Rely on finalizers unless necessary
- Throw exceptions from `Dispose()`
- Access other managed objects in finalizer
- Forget `GC.SuppressFinalize()` in `Dispose()`

## Modern Simplification

If **only managed resources**, skip finalizer:

```csharp
public class SimpleHandler : IDisposable {
    private FileStream? stream;

    public void Dispose() {
        stream?.Dispose();
        stream = null;
    }
}
```

## Real-World Example

```csharp
public class DatabaseService : IDisposable {
    private SqlConnection connection;
    private bool disposed;

    public DatabaseService(string connectionString) {
        connection = new SqlConnection(connectionString);
        connection.Open();
    }

    public void Query(string sql) {
        if (disposed) 
            throw new ObjectDisposedException(nameof(DatabaseService));
        // Execute query
    }

    public void Dispose() {
        if (disposed) return;
        connection?.Close();
        connection?.Dispose();
        disposed = true;
    }
}

// Usage
using var db = new DatabaseService("connection string");
db.Query("SELECT * FROM Users");
// Connection closed and disposed automatically
```

## Summary

`IDisposable` = deterministic cleanup for resources GC doesn't manage. Always use `using` statements to ensure disposal, even with exceptions.
