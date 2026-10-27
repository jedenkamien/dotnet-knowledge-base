# Asynchronous Programming – Theory

## The Problem
Synchronous I/O blocks a thread while waiting for the OS / network / disk. Threads are expensive (memory + context switches), so blocking thousands of them does not scale.

## The Idea
Release the thread back to the pool while waiting; resume work on **any available thread** when the I/O completes (via OS completion ports / IOCP on Windows, epoll on Linux).

## Concurrency vs Parallelism
- **Concurrency**: multiple operations *in progress* (interleaved). Useful for I/O.
- **Parallelism**: multiple operations *executing simultaneously* on different cores. Useful for CPU-bound work.
- `async/await` is primarily about concurrency, not parallelism.

## How async/await Works (under the hood)
The compiler rewrites an `async` method into a **state machine**:
1. Code before `await` runs synchronously.
2. At `await`, if the awaited Task is incomplete, a continuation is registered and control returns to the caller.
3. When the Task completes, the continuation is scheduled (via `SynchronizationContext` or `TaskScheduler`).
4. State machine resumes from where it left off.

## Synchronization Context
- **UI apps (WinForms/WPF)**: continuations resume on the UI thread.
- **ASP.NET Classic**: resumes on a request context.
- **ASP.NET Core / Console**: no context — continuations run on thread pool.
- **`ConfigureAwait(false)`**: don't capture context. Use in libraries to avoid deadlocks.

## CPU-bound vs I/O-bound

| Type | Tool |
|------|------|
| I/O (HTTP, DB, file) | `async/await` over native async APIs |
| CPU (computation) | `Task.Run`, `Parallel.For`, PLINQ |
| Mixed | async at boundaries, parallel inside |

## Key Anti-Patterns
- **`async void`** outside of event handlers — unobservable exceptions.
- **`.Result` / `.Wait()`** — deadlock under sync contexts; wastes threads.
- **`Task.Run` over async I/O** — adds a hop, no scalability gain.
- **Mixing sync and async** ("sync over async" or "async over sync").

## Cancellation & Timeouts
Pass `CancellationToken` through the call chain. Compose with `CancellationTokenSource.CreateLinkedTokenSource` and `cts.CancelAfter(timeout)`.

## Mental Model
> A `Task` is a *promise of a future result*. `await` says "pause here, free this thread, wake me when the promise is fulfilled." It is cooperative, not preemptive.
