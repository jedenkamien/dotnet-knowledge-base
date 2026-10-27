# Async/Await Patterns

## Core Concepts

**async/await** enables non-blocking asynchronous programming with synchronous-looking code.

```csharp
async Task<string> FetchDataAsync() {
    HttpClient client = new HttpClient();
    string result = await client.GetStringAsync("https://api.example.com/data");
    return result;  // Compiler wraps in Task<string>
}
```

**What happens:**
1. `await` yields control back to caller
2. Thread is freed for other work — but **only** because there's a real async I/O operation underneath (OS handles it without occupying a thread)
3. When I/O completes, continues from await point
4. Returns `Task<T>` that completes with result

**Key insight — `await` only frees the thread for I/O-bound work:**
CPU-bound work has no such mechanism. The CPU core just grinds through it, keeping the thread busy the whole time. `await` on a synchronous method does NOT magically offload it.

## Key Rules

**Return Types**
- `Task<T>`: Returns value T
- `Task`: No return value (like void)
- `void`: Only for event handlers (cannot await)
- `ValueTask<T>`: Performance optimization (reduces allocations)

**Naming Convention**
- Async methods end with `Async` suffix: `GetDataAsync()`, `SaveAsync()`

**Exception Handling**
```csharp
try {
    await SomeOperationAsync();
} catch (HttpRequestException ex) {
    // Exceptions propagate naturally through await
}
```

## Synchronization Context

This concept explains a lot of async/await behavior.

**What it is:** A mechanism that controls where `await` continuations resume after an async operation completes. Captured at the `await` point.

**Default behavior depends on environment:**
| Environment | Sync context present? | Where continuation resumes |
|---|---|---|
| WinForms / WPF / MAUI | Yes (UI sync context) | UI thread |
| Classic ASP.NET | Yes | Original request thread |
| ASP.NET Core | No (deliberately removed) | Any thread pool thread |
| Console app | No | Any thread pool thread |
| Background services | No | Any thread pool thread |

**Why UI frameworks use it:** UI controls are not thread-safe. They can only be touched from the UI thread. The sync context ensures post-await code like `myLabel.Text = data` works without explicit marshalling.

**`ConfigureAwait(false)`** opts out of capturing the sync context — continuation runs on any thread pool thread:
```csharp
// Library code: don't capture synchronization context
await SomeOperationAsync().ConfigureAwait(false);

// UI code: default behavior (resumes on UI thread)
await SomeOperationAsync();
```

After `ConfigureAwait(false)`, touching UI controls will throw `InvalidOperationException: cross-thread operation not valid`. If needed, marshal back manually:
```csharp
var data = await FetchAsync().ConfigureAwait(false);
Dispatcher.Invoke(() => myLabel.Text = data);  // WPF
```

**Rule of thumb:** Use `ConfigureAwait(false)` in library/service code. Omit it in UI code.

## Common Patterns

### 1. Parallel Execution with WhenAll
```csharp
var task1 = FetchUser(1);
var task2 = FetchUser(2);
var task3 = FetchUser(3);
await Task.WhenAll(task1, task2, task3);  // Runs in parallel
var results = new[] { task1.Result, task2.Result, task3.Result };
// .Result is safe here — tasks are already completed
```

### 2. First Result with WhenAny
```csharp
var fastestTask = await Task.WhenAny(apiCall1, apiCall2, apiCall3);
var result = await fastestTask;  // Use whichever completes first
```

### 3. Timeout Pattern
```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
try {
    await LongRunningOperationAsync(cts.Token);
} catch (OperationCanceledException) {
    Console.WriteLine("Timeout!");
}
```

### 4. Sequential vs Parallel
```csharp
// Sequential (slower): total time = sum of all tasks
var result1 = await Task1Async();
var result2 = await Task2Async();

// Parallel (faster): total time = longest task
var t1 = Task1Async();
var t2 = Task2Async();
await Task.WhenAll(t1, t2);
```

### 5. CPU-bound work from UI thread
```csharp
// BAD: HeavyCalculation runs on UI thread even though method is async
async Task HeavyAsync() {
    HeavyCalculation();  // no await inside → UI thread does all the work
}
await HeavyAsync();  // UI still freezes!

// GOOD: Task.Run pushes work to thread pool, UI thread is free
var result = await Task.Run(() => HeavyCalculation());
```

**Note:** `Task.Run()` is mainly valuable from the UI thread. In background services or ASP.NET Core controllers, using `Task.Run()` for CPU-bound work adds scheduling overhead with no benefit — you're just moving work from one pool thread to another.

## Async Anti-Patterns

### ❌ Async void (except event handlers)
```csharp
async void ProcessData() { }  // Cannot await, exceptions crash app
```

**Why you can't `await` it:** `await` needs a `Task` object to track the operation. `async void` returns nothing — there's no handle to wait on.

**Why exceptions crash:** With `async Task`, unhandled exceptions are stored inside the `Task` and re-thrown when you `await` it. With `async void`, there's no `Task` to store the exception — it gets thrown directly on the synchronization context → unhandled exception → app crash.

### ❌ Blocking on async code
```csharp
var result = SomeAsync().Result;  // Deadlock risk
SomeAsync().Wait();               // Same issue
```

**Why `.Result` deadlocks in UI apps (step by step):**
1. UI thread calls `.Result` → blocks waiting for the Task to finish
2. The async method hits `await`, captures the UI sync context, and returns a Task
3. I/O finishes on OS/thread pool
4. The `await` continuation needs to resume on the UI thread (captured context)
5. UI thread is blocked at `.Result` → can't run the continuation → Task never completes → `.Result` waits forever

**When `.Result` is safe:**
- No sync context present (console apps, thread pool threads, ASP.NET Core)
- Task is already completed (e.g. after `await Task.WhenAll(...)`)
- Inside `Task.Run()` (thread pool thread, no UI sync context)

**Why `.Result` exists at all:** `Task` is a general-purpose object usable outside async/await. Legitimate uses: `Main()` before `async Main` was introduced, synchronous interop scenarios.

### ❌ Unnecessary async
```csharp
async Task<int> GetValue() {
    return 42;  // No await, don't use async
}
// Should be: Task<int> GetValue() => Task.FromResult(42);
```

### ❌ Async all the way up
```csharp
// If calling async, entire chain should be async
public void Controller() {
    Service().Wait();  // BAD
}
public async Task ControllerAsync() {
    await ServiceAsync();  // GOOD
}
```

## Cancellation

```csharp
async Task ProcessAsync(CancellationToken cancellationToken) {
    for (int i = 0; i < 100; i++) {
        cancellationToken.ThrowIfCancellationRequested();
        await Task.Delay(100, cancellationToken);
    }
}

var cts = new CancellationTokenSource();
var task = ProcessAsync(cts.Token);
cts.Cancel();  // Triggers cancellation
```

## ValueTask vs Task

**Task**: Always allocates on heap
**ValueTask**: Stack-allocated if result is already available (no allocation on cache hit)

```csharp
ValueTask<int> GetCachedValueAsync(int key) {
    if (cache.TryGetValue(key, out var value))
        return new ValueTask<int>(value);  // No allocation
    return new ValueTask<int>(FetchFromDbAsync(key));
}
```

Use `ValueTask<T>` for hot paths with high cache hit rates.

## I/O-bound vs CPU-bound

| | I/O-bound | CPU-bound |
|---|---|---|
| Examples | File, network, database | Image processing, calculations |
| Thread during operation | Freed — OS handles it | Busy the whole time |
| Correct approach | `await SomeAsync()` | `await Task.Run(() => HeavyWork())` |
| `Task.Run` needed? | No | Yes (from UI); No (from background) |

## Test Frameworks and Deadlocks

Deadlocks from `.Result` are not only a UI problem:
- **xUnit** — no sync context → safe
- **NUnit / MSTest** (older versions) — installs sync context on test threads → `.Result` can deadlock exactly like in a UI app

Always `await` in tests instead of blocking with `.Result`.

## Remember

- `await` != parallelism (use `Task.WhenAll` for that)
- `await` only yields the thread for **I/O-bound** work; CPU work still blocks
- Async is viral: once you go async, everything above must be async
- Don't mix blocking and async code
- ASP.NET Core has no sync context — `ConfigureAwait(false)` is redundant but harmless there
