# Task.WhenAll – Practical Examples

## What It Does
Returns a single `Task` that completes when **all** supplied tasks complete. Runs them concurrently, not sequentially.

```csharp
await Task.WhenAll(task1, task2, task3);
```

## Sequential vs Parallel

```csharp
// BAD: ~3s total (sequential awaits)
var a = await GetAsync(1);
var b = await GetAsync(2);
var c = await GetAsync(3);

// GOOD: ~1s total (parallel)
var t1 = GetAsync(1);   // start
var t2 = GetAsync(2);
var t3 = GetAsync(3);
await Task.WhenAll(t1, t2, t3);
var (a, b, c) = (t1.Result, t2.Result, t3.Result);
```

## Collecting Results

```csharp
int[] ids = { 1, 2, 3, 4, 5 };
Task<User>[] tasks = ids.Select(id => repo.GetUserAsync(id)).ToArray();
User[] users = await Task.WhenAll(tasks);  // Returns array of results
```

## Fan-out / Fan-in

```csharp
async Task<Report> BuildReportAsync() {
    var sales   = LoadSalesAsync();
    var users   = LoadUsersAsync();
    var inventory = LoadInventoryAsync();

    await Task.WhenAll(sales, users, inventory);

    return new Report(sales.Result, users.Result, inventory.Result);
}
```

## Exception Behavior

- If multiple tasks throw, `WhenAll` aggregates them but `await` re-throws **only the first**.
- Use `task.Exception` or inspect each task to see all failures.

```csharp
var t = Task.WhenAll(t1, t2, t3);
try { await t; }
catch {
    foreach (var ex in t.Exception!.InnerExceptions)
        Log(ex);
}
```

## Throttling Concurrency

`WhenAll` with thousands of tasks can overwhelm downstreams. Use a `SemaphoreSlim`:

```csharp
var gate = new SemaphoreSlim(10); // max 10 in flight
var tasks = items.Select(async item => {
    await gate.WaitAsync();
    try   { return await ProcessAsync(item); }
    finally { gate.Release(); }
});
var results = await Task.WhenAll(tasks);
```

Or `Parallel.ForEachAsync` (.NET 6+) which has built-in `MaxDegreeOfParallelism`.

## Common Mistakes

- Awaiting inside a `foreach` instead of collecting tasks → serial execution
- Using `WhenAll` for ordered work that depends on previous result
- Forgetting that **all tasks start when called**, not when awaited
