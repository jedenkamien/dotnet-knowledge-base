# Synchronization Primitives (.NET Interview Notes)

## Why Synchronization Exists

Threads within the same process share memory.

Problem:

```csharp
counter++;
```

This is not atomic.

Internally it is roughly:

```text
Read
Modify
Write
```

Example:

```text
counter = 5

Thread A reads 5
Thread B reads 5

Thread A writes 6
Thread B writes 6
```

Expected:

```text
7
```

Actual:

```text
6
```

This is called a **race condition**.

Synchronization primitives exist to coordinate access to shared mutable state.

---

## Process vs Thread

### Process

A process is an isolated operating-system unit.

Characteristics:

- Separate virtual memory space
- Own handles/resources
- Protected from memory access by other processes
- Usually heavier to create than a thread

Examples:

- Chrome
- Visual Studio
- SQL Server

Different processes can still compete for machine resources such as:

- CPU time
- RAM
- Disk I/O
- Network bandwidth

But they cannot directly read/write each other's memory.

### Thread

A thread is an execution path inside a process.

Characteristics:

- Shares process memory
- Shares heap
- Shares static fields
- Can access the same objects as other threads

Because threads share memory, synchronization becomes necessary.

---

# Synchronization Primitive Cheat Sheet

| Problem | Usually Use |
|----------|----------|
| Protect shared state | lock |
| Single atomic operation | Interlocked |
| Async synchronization | SemaphoreSlim |
| Many readers, few writers | ReaderWriterLockSlim |
| Cross-process synchronization | Mutex |
| Thread signaling | AutoResetEvent / ManualResetEvent |
| Extremely short waits | SpinLock (rare) |

---

# lock

The default synchronization primitive.

```csharp
private readonly object _lock = new();

lock (_lock)
{
    counter++;
}
```

### Important Concepts

The object itself is usually meaningless.

```csharp
private readonly object _lock = new();
```

is simply an identity token used by the CLR.

The lock protects a block of code, not a variable.

Correct:

```csharp
lock(_lock)
{
    counter++;
}
```

Wrong:

```csharp
counter++;
```

somewhere else without using the same lock.

Every code path touching the protected state must use the same lock object.

### Multiple Methods

```csharp
public void Increment()
{
    lock(_lock)
    {
        counter++;
    }
}

public void Reset()
{
    lock(_lock)
    {
        counter = 0;
    }
}
```

These methods block each other because they use the same lock.

### Waiting Behavior

Waiting threads:

- Do not burn CPU
- Are blocked by the OS/runtime
- Resume when the lock becomes available

### Reentrancy

`lock` is reentrant.

```csharp
lock(_lock)
{
    MethodB();
}

void MethodB()
{
    lock(_lock)
    {
        // allowed
    }
}
```

The same thread may acquire the same lock multiple times.

### Common Mistakes

```csharp
lock(this)
```

```csharp
lock("abc")
```

```csharp
lock(publicObject)
```

Because external code could also lock on them.

Use a private lock object instead.

---

# Monitor

`lock` is actually syntax sugar over `Monitor`.

Roughly:

```csharp
Monitor.Enter(_lock);

try
{
    // protected code
}
finally
{
    Monitor.Exit(_lock);
}
```

### TryEnter

Useful when you do not want to wait forever.

```csharp
if (Monitor.TryEnter(_lock, TimeSpan.FromSeconds(5)))
{
    try
    {
        // work
    }
    finally
    {
        Monitor.Exit(_lock);
    }
}
```

Important interview point:

The runtime repeatedly attempts to acquire the lock during the timeout period, not just once at the beginning and once at the end.

---

# Interlocked

Use when a single atomic operation is sufficient.

```csharp
Interlocked.Increment(ref counter);
```

```csharp
Interlocked.Decrement(ref counter);
```

```csharp
Interlocked.Exchange(ref value, newValue);
```

```csharp
Interlocked.CompareExchange(ref value, newValue, expected);
```

### Why Is It Atomic?

Normal increment:

```csharp
counter++;
```

needs:

```text
Read
Modify
Write
```

Interlocked uses special CPU instructions that perform the operation as one indivisible unit.

No other thread can observe an intermediate state.

### Interview Rule

> If one Interlocked call solves the problem, prefer Interlocked over lock.

It is usually faster and simpler.

---

# SemaphoreSlim

Think:

```text
lock            => capacity 1
SemaphoreSlim   => capacity N
```

Example:

```csharp
var semaphore = new SemaphoreSlim(3);

await semaphore.WaitAsync();

try
{
    await ProcessAsync();
}
finally
{
    semaphore.Release();
}
```

---

## Conceptual Model

A semaphore maintains a permit count.

```text
SemaphoreSlim(3)

Available permits: 3
```

First caller:

```text
Available permits: 2
```

Second caller:

```text
Available permits: 1
```

Third caller:

```text
Available permits: 0
```

Fourth caller:

```text
Must wait
```

When:

```csharp
semaphore.Release();
```

happens:

```text
Available permits: 1
```

One waiting caller immediately consumes it:

```text
Available permits: 0
```

---

## Parking Lot Analogy

```text
SemaphoreSlim(3)

[ ] [ ] [ ]

Car A enters
[ ] [ ]

Car B enters
[ ]

Car C enters

No spaces left

Car D waits

Car A leaves

Car D immediately takes the free space
```

---

## What WaitAsync Actually Means

```csharp
await semaphore.WaitAsync();
```

means:

```text
Try to acquire a permit.

If available:
    Continue immediately.

If unavailable:
    Wait until somebody releases one.
```

It does **not** mean:

```text
Lock this code block
```

Only code using the same semaphore participates in this coordination.

---

## Wait vs WaitAsync

### Wait()

```csharp
semaphore.Wait();
```

- Blocks current thread
- Occupies thread while waiting

### WaitAsync()

```csharp
await semaphore.WaitAsync();
```

- Does not block a thread
- Suspends the task
- Continues later when a permit becomes available

This is why SemaphoreSlim fits naturally with async/await.

---

## Common Uses

- Limiting outgoing HTTP requests
- Limiting database connections
- Restricting expensive parallel work

---

## Interview Gotchas

Always release in finally:

```csharp
try
{
}
finally
{
    semaphore.Release();
}
```

Otherwise permits may be permanently lost.

---

# AutoResetEvent

Used for thread signaling.

Think:

```text
One signal -> one waiter released
```

---

## State

```csharp
var gate = new AutoResetEvent(false);
```

Meaning:

```text
false -> unsignaled -> closed
true  -> signaled   -> open
```

### Initially Closed

```csharp
var gate = new AutoResetEvent(false);

gate.WaitOne();
```

The thread blocks.

Later:

```csharp
gate.Set();
```

releases one waiting thread.

---

## Initially Open

```csharp
var gate = new AutoResetEvent(true);
```

The first call to:

```csharp
gate.WaitOne();
```

passes immediately.

Then the event automatically resets to closed.

Example:

```csharp
var gate = new AutoResetEvent(true);

Console.WriteLine("A");

gate.WaitOne();

Console.WriteLine("B");

gate.WaitOne(); // blocks
```

---

## Mental Model

```text
Closed

Set()

Open

One thread passes

Automatically closes
```

---

## Important Rule

Each call to:

```csharp
Set();
```

releases at most one waiting thread.

---

# ManualResetEvent

Same signaling concept.

Different behavior.

```csharp
evt.Set();
```

opens the gate and keeps it open.

It releases:

- all current waiting threads
- all future waiting threads

until:

```csharp
evt.Reset();
```

is executed.

### Mental Model

```text
AutoResetEvent
Open for one person

ManualResetEvent
Open and stay open
```

---

# ReaderWriterLockSlim

Optimized for read-heavy workloads.

### Reading

```csharp
rw.EnterReadLock();

try
{
    return cache[id];
}
finally
{
    rw.ExitReadLock();
}
```

### Writing

```csharp
rw.EnterWriteLock();

try
{
    cache[id] = value;
}
finally
{
    rw.ExitWriteLock();
}
```

### Behavior

Many readers can enter simultaneously.

```text
Reader A
Reader B
Reader C
```

all allowed.

Writer:

```text
Writer
```

gets exclusive access.

No readers or writers may execute concurrently with a writer.

### Good Candidates

- Caches
- Configuration data
- Reference data

### Interview Rule

Use only when reads dramatically outnumber writes.

For many applications a simple lock is preferable.

---

# Mutex

A mutex is similar to a lock but works across process boundaries.

### Example

Allow only a single application instance:

```text
Process A -> acquires mutex

Process B -> cannot acquire mutex
```

### Interview Summary

```text
lock  -> same process

Mutex -> multiple processes
```

Mutex is significantly heavier than lock.

---

# SpinLock

Instead of sleeping while waiting:

```text
Free yet?
Free yet?
Free yet?
Free yet?
```

the thread continuously checks.

Characteristics:

- Consumes CPU
- Useful only for extremely short waits
- Rare in business applications

### Interview Rule

If you think you need SpinLock, you probably do not.

---

# Deadlock

Classic example:

```text
Thread A

Lock A
Wait Lock B
```

```text
Thread B

Lock B
Wait Lock A
```

Result:

```text
Nobody progresses
```

### Prevention

Acquire locks in a consistent order.

Example:

Always:

```text
Lock A
then Lock B
```

Never sometimes:

```text
A -> B
```

and other times:

```text
B -> A
```

---

# volatile

`volatile` guarantees visibility.

It does **not** guarantee atomicity.

Example:

```csharp
private volatile bool stopRequested;
```

Worker:

```csharp
while (!stopRequested)
{
}
```

Another thread:

```csharp
stopRequested = true;
```

With volatile, the worker is forced to observe updates.

---

## What volatile Does NOT Solve

Still unsafe:

```csharp
volatileCounter++;
```

because increment is still:

```text
Read
Modify
Write
```

Use:

```csharp
Interlocked.Increment(...)
```

or

```csharp
lock
```

instead.

### Interview Summary

```text
volatile
    Visibility

Interlocked
    Atomicity

lock
    Atomicity + Larger Critical Sections
```

---

# CPU Usage While Waiting

| Primitive | CPU while waiting | Thread occupied |
|------------|------------|------------|
| lock | No | Yes |
| Monitor | No | Yes |
| SemaphoreSlim.Wait() | No | Yes |
| SemaphoreSlim.WaitAsync() | No | No |
| AutoResetEvent.WaitOne() | No | Yes |
| ManualResetEvent.WaitOne() | No | Yes |
| Mutex | No | Yes |
| SpinLock | Yes | Yes |

---

# Threads vs Tasks

Common interview topic around synchronization.

## Thread

```csharp
var thread = new Thread(Work);
thread.Start();
```

Characteristics:

- Actual operating-system thread
- Relatively expensive
- Dedicated execution path

Lifecycle:

```text
new Thread(...)
    Creates thread object

Start()
    OS thread begins execution

Join()
    Wait until thread ends
```

---

## Task

```csharp
Task.Run(Work);
```

A Task represents asynchronous work.

Usually executed on ThreadPool threads.

Characteristics:

- Higher-level abstraction
- Cheaper to create
- Better scalability

### Does Task.Run Immediately Create a Thread?

Not necessarily.

It queues work to the ThreadPool.

The ThreadPool decides:

- when to run it
- on which worker thread

### Ordering

```csharp
Task.Run(Task1);
Task.Run(Task2);
```

There is no guarantee Task1 starts before Task2.

There is also no guarantee they use different threads.

### Can One Task Use Multiple Threads?

Normally:

```text
No
```

A task executes on one thread at a time.

But after an await it may resume on a different thread.

---

## Why Task.Run Can Be Harmful on Server Code

Example:

```csharp
await Task.Run(() => DatabaseCall());
```

If `DatabaseCall()` is already asynchronous-capable, Task.Run simply occupies an extra ThreadPool thread.

This:

```text
Consumes worker threads
Reduces scalability
Adds overhead
```

without adding value.

---

# Common Interview Questions

### Why not use lock everywhere?

Because correctness is not the only concern.

Examples:

- Counter → Interlocked
- Read-heavy cache → ReaderWriterLockSlim
- Async coordination → SemaphoreSlim
- Cross-process coordination → Mutex

Still:

> Start with lock. Introduce more specialized primitives only when there is a proven reason.

---

### How do you print 101010 from two concurrent workers?

One worker prints:

```text
1
```

Another prints:

```text
0
```

Use signaling primitives (for example AutoResetEvents) to alternate execution:

```text
1 signals 0

0 signals 1

1 signals 0

...
```

This guarantees ordering despite concurrency.

---

# Senior-Level Takeaway

Before reaching for synchronization primitives, ask:

1. Can I eliminate shared mutable state?
2. Can I make the data immutable?
3. Can Interlocked solve it?
4. Can a simple lock solve it?
5. Only then consider SemaphoreSlim, ReaderWriterLockSlim, Mutex, SpinLock, etc.

Most production concurrency bugs come from unnecessary shared mutable state, not from choosing the wrong synchronization primitive.