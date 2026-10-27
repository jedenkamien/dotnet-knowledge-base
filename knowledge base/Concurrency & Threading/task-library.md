# Task Parallel Library (TPL)

## Purpose
Abstraction over threads representing asynchronous units of work. Replaces manual `Thread` management for most scenarios.

## Task vs Thread

| Aspect | `Thread` | `Task` |
|--------|----------|--------|
| Cost | OS thread (~1 MB stack) | Lightweight, pooled |
| Returns value | No (manual) | `Task<T>` natively |
| Composition | Hard | `WhenAll`, `ContinueWith` |
| Cancellation | Manual flag | `CancellationToken` |
| Scheduling | OS scheduler | `TaskScheduler` (default = ThreadPool) |

## Creating Tasks

```csharp
// CPU-bound work → Task.Run
Task<int> compute = Task.Run(() => HeavyCalculation());

// I/O-bound → already returns Task
Task<string> data = httpClient.GetStringAsync(url);

// Pre-completed
Task<int> ready = Task.FromResult(42);
Task failed = Task.FromException(new InvalidOperationException());
```

**Don't:** `Task.Factory.StartNew` for async lambdas (use `Task.Run`).
**Don't:** wrap inherently async I/O in `Task.Run` (no benefit, extra thread).

## Awaiting & Composition

```csharp
int result = await compute;
await Task.WhenAll(task1, task2, task3);    // All complete
Task first = await Task.WhenAny(task1, task2); // First to finish
await Task.Delay(1000);                      // Non-blocking sleep
```

## Cancellation

```csharp
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
await DoWorkAsync(cts.Token);

async Task DoWorkAsync(CancellationToken ct) {
    foreach (var item in items) {
        ct.ThrowIfCancellationRequested();
        await ProcessAsync(item, ct);
    }
}
```

## Common Pitfalls

- **`.Result` / `.Wait()`** → deadlocks in UI/ASP.NET sync contexts
- **`async void`** → exceptions cannot be caught by caller
- **Forgotten `await`** → fire-and-forget, exceptions lost
- **Task.Run on server code** → wastes thread pool threads

## TaskCompletionSource

Bridge non-Task APIs (events, callbacks) into the Task world:

```csharp
var tcs = new TaskCompletionSource<int>();
button.Click += (s, e) => tcs.SetResult(42);
int value = await tcs.Task;
```
