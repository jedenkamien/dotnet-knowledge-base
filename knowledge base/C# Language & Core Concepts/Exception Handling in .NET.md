# Exception Handling in .NET - Interview Overview

Exception handling is a mechanism for dealing with **unexpected runtime errors** while keeping the application in a valid state and providing meaningful information for troubleshooting.

A common interview theme is understanding not only *how* exceptions work, but also *when* they should and should not be used.

---

# Basic Exception Flow

A typical sequence:

```csharp
try
{
    DoWork();
}
catch (Exception ex)
{
    Log(ex);
}
finally
{
    Cleanup();
}
```

Execution:

1. Code enters `try`.
2. Exception occurs.
3. CLR starts stack unwinding.
4. Matching catch block is found.
5. `finally` executes.
6. Execution continues (unless rethrow occurs).

---

# Exception Hierarchy

Everything derives from:

```csharp
System.Exception
```

Examples:

- ArgumentException
    - ArgumentNullException
    - ArgumentOutOfRangeException
- InvalidOperationException
- TimeoutException
- IOException
- UnauthorizedAccessException

Interview question:

> Should you throw Exception directly?

Generally no.

Prefer the most specific exception available.

Good:

```csharp
throw new ArgumentNullException(nameof(customer));
```

Bad:

```csharp
throw new Exception("Customer is null");
```

---

# Throw vs Throw ex

Classic interview question.

Wrong:

```csharp
catch(Exception ex)
{
    Log(ex);
    throw ex;
}
```

Correct:

```csharp
catch(Exception ex)
{
    Log(ex);
    throw;
}
```

Why?

`throw ex` resets the stack trace.

`throw` preserves the original stack trace.

Interview answer:

> Use `throw;` when rethrowing because debugging information remains intact.

---

# Stack Unwinding

When an exception occurs:

```csharp
A()
 -> B()
     -> C()
         -> throw
```

CLR walks back:

- C
- B
- A

looking for an appropriate catch block.

All `finally` blocks encountered during this process execute.

---

# Finally Block

Runs regardless of:

- success
- exception
- return statement

Example:

```csharp
try
{
    return 5;
}
finally
{
    Console.WriteLine("Executed");
}
```

Output:

```csharp
Executed
```

Then method returns 5.

Interviewers love this one.

---

# Using Statement

Modern resource management relies on:

```csharp
using var stream = new FileStream(...);
```

Compiler roughly converts it into:

```csharp
try
{
}
finally
{
    stream?.Dispose();
}
```

Therefore resources are disposed even if exceptions occur.

---

# Creating Custom Exceptions

Possible:

```csharp
public class OrderNotFoundException : Exception
{
    public OrderNotFoundException(string id)
        : base($"Order {id} not found")
    {
    }
}
```

When useful:

- business libraries
- reusable frameworks
- domain-specific failure scenarios

When unnecessary:

- creating dozens of application-specific exceptions nobody handles differently

Interview tip:

> Most business applications need fewer custom exceptions than developers think.

---

# Exceptions vs Validation

Bad:

```csharp
try
{
    int age = int.Parse(input);
}
catch
{
}
```

Better:

```csharp
if (int.TryParse(input, out var age))
{
}
```

Exceptions are for:

- exceptional situations

Not for:

- normal control flow
- validation loops
- expected user mistakes

Interview answer:

> Exceptions are expensive and should not be used for ordinary branching logic.

---

# Exception Filters

Often forgotten.

```csharp
catch(HttpRequestException ex)
when (ex.StatusCode == HttpStatusCode.NotFound)
{
}
```

Benefits:

- condition checked before entering catch
- preserves original stack information
- cleaner than catch + if

Interviewers sometimes ask about C# features people rarely use.

---

# Inner Exceptions

Wrapping exceptions:

```csharp
catch(SqlException ex)
{
    throw new OrderRepositoryException(
        "Failed to load order",
        ex);
}
```

Original exception becomes:

```csharp
ex.InnerException
```

Purpose:

- add business context
- retain technical root cause

Very common in enterprise systems.

---

# AggregateException

Appears with parallel operations.

```csharp
Task.WaitAll(task1, task2);
```

Can throw:

```csharp
AggregateException
```

containing multiple inner exceptions.

Example:

```csharp
catch(AggregateException ex)
{
    foreach(var inner in ex.InnerExceptions)
    {
    }
}
```

---

# Exceptions in async/await

Critical interview area.

```csharp
await service.GetDataAsync();
```

Exception inside async method:

- stored in Task
- rethrown when awaited

Example:

```csharp
try
{
    await GetDataAsync();
}
catch(Exception ex)
{
}
```

Works exactly as expected.

---

# What If You Forget await?

```csharp
try
{
    service.GetDataAsync();
}
catch
{
}
```

Catch does NOT catch the exception.

Because the method already returned a Task.

This is a very common production bug.

---

# Unobserved Task Exceptions

```csharp
Task.Run(() =>
{
    throw new Exception();
});
```

If nobody awaits or observes the task:

- exception remains inside Task
- may later trigger TaskScheduler.UnobservedTaskException

Modern .NET generally avoids crashing the process here, but it still indicates a bug.

Interview answer:

> Fire-and-forget tasks should be used carefully because exceptions can easily be lost.

---

# Constructor Exceptions

Since you asked about this recently, interviewers occasionally bring it up.

```csharp
var customer = new Customer();
```

If constructor throws:

```csharp
public Customer()
{
    throw new Exception();
}
```

Result:

- object creation fails
- reference is never assigned
- partially created object becomes eligible for GC

The caller receives the exception.

---

# Static Constructor Exceptions

Important edge case.

```csharp
static Customer()
{
    throw new Exception();
}
```

CLR wraps it into:

```csharp
TypeInitializationException
```

After failure:

- type becomes unusable
- future accesses fail again

Many experienced developers encounter this only once or twice in their careers.

---

# When NOT To Catch

Bad:

```csharp
catch(Exception)
{
}
```

This is called "swallowing exceptions."

Problems:

- hiding bugs
- impossible troubleshooting
- corrupted application state

If you catch, either:

- recover
- translate
- log and rethrow

Otherwise don't catch.

---

# Fatal Exceptions

Some exceptions generally should not be handled:

- StackOverflowException
- OutOfMemoryException (usually)
- AccessViolationException

Interview answer:

> Some failures indicate the process may already be in an invalid state, so recovery cannot be trusted.

---

# Best Practices

### Throw early

```csharp
ArgumentNullException.ThrowIfNull(customer);
```

### Catch late

Catch near an application boundary:

- API middleware
- message processor
- background worker
- UI entry point

### Use specific exceptions

Avoid generic Exception.

### Preserve stack traces

Use:

```csharp
throw;
```

not:

```csharp
throw ex;
```

### Never use exceptions for normal flow

Use validation APIs such as:

- TryParse
- TryGetValue

---

# High-Probability Interview Questions

### What is the difference between throw and throw ex?

Preserving vs resetting stack trace.

### Does finally execute after return?

Yes.

### Does finally execute during exception unwinding?

Yes.

### Can a constructor throw?

Yes. Object creation fails.

### What happens when an async method throws?

Exception is captured into Task and rethrown upon await.

### Why should exceptions not be used for control flow?

Performance cost and poor readability.

### What is AggregateException?

Container for multiple exceptions from parallel operations.

### What is an InnerException?

Original exception wrapped by another exception.

### What happens if you do not await a Task?

Exceptions may be missed and bypass local try/catch.

### When would you create a custom exception?

When callers need to distinguish a specific business or library failure from other failures.

---

# Senior-Level Insight

A strong senior answer is:

> Exceptions represent failures that the current code cannot reasonably handle. Most layers should not catch exceptions. Infrastructure and application boundaries should catch them, log them, translate them into appropriate responses (HTTP 500, gRPC status, message retry, etc.), and allow business code to remain focused on the happy path.