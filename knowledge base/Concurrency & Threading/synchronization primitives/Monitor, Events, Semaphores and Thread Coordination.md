# .NET Synchronization Primitives: Monitor, Events, Semaphores and Thread Coordination

## Overview

When multiple threads need to cooperate, there are two fundamentally different concerns:

1. **Mutual Exclusion**
   - Prevent multiple threads from accessing shared data simultaneously.
   - Typical tool: `lock` (`Monitor.Enter/Exit`).

2. **Coordination / Signaling**
   - Control the order in which threads execute.
   - Typical tools:
     - `Monitor.Wait/Pulse`
     - `AutoResetEvent`
     - `ManualResetEvent`
     - `SemaphoreSlim`

A common interview mistake is thinking that `lock` alone can coordinate execution order. It cannot.

---

# The 101010 Problem

## Problem Statement

Two concurrent workers execute:

- Worker A prints `1` three times
- Worker B prints `0` three times

Desired output:

```
101010
```

The challenge is not protecting shared data.

The challenge is ensuring that the threads take turns.

---

# Solution 1: Monitor + Wait/Pulse

## When to use

Good when execution depends on shared state.

This is the classic interview answer because it demonstrates understanding of condition synchronization.

## Complete Example

```csharp
using System;
using System.Threading;

class Program
{
    private static readonly object _lock = new();
    private static bool shouldPrintOne = true;

    static void Main()
    {
        var t1 = new Thread(PrintOnes);
        var t2 = new Thread(PrintZeros);

        t1.Start();
        t2.Start();

        t1.Join();
        t2.Join();
    }

    static void PrintOnes()
    {
        for (int i = 0; i < 3; i++)
        {
            lock (_lock)
            {
                while (!shouldPrintOne)
                    Monitor.Wait(_lock);

                Console.Write("1");

                shouldPrintOne = false;
                Monitor.Pulse(_lock);
            }
        }
    }

    static void PrintZeros()
    {
        for (int i = 0; i < 3; i++)
        {
            lock (_lock)
            {
                while (shouldPrintOne)
                    Monitor.Wait(_lock);

                Console.Write("0");

                shouldPrintOne = true;
                Monitor.Pulse(_lock);
            }
        }
    }
}
```

or 

```csharp
public void Print101010()
{
    var lockObj = new object();
    var onesTurn = true;

    void PrintOnes()
    {
        for (var i=0; i<3; i++)
        {
            lock (lockObj)
            {
                while (!onesTurn)
                    Monitor.Wait(lockObj);

                Console.Write("1");
                onesTurn = false;
                Monitor.Pulse(lockObj);
            }
        }
    }

    void PrintZeros()
    {
        for (var i=0; i<3; i++)
        {
            lock (lockObj)
            {
                while (onesTurn)
                    Monitor.Wait(lockObj);

                Console.Write("0");
                onesTurn = true;
                Monitor.Pulse(lockObj);
            }
        }
    }

    var t1 = new Thread(PrintOnes);
    var t2 = new Thread(PrintZeros);

    t1.Start();
    t2.Start();

    t1.Join();
    t2.Join();
}
```

or

```csharp
public void Print101010()
{
    var lockObj = new object();
    var onesTurn = true;

    void PrintOnes()
    {
        for (var i=0; i<3; i++)
        {
            lock (lockObj)
            {
                while (!onesTurn)
                    Monitor.Wait(lockObj);

                Console.Write("1");
                onesTurn = false;
                Monitor.Pulse(lockObj);
            }
        }
    }

    void PrintZeros()
    {
        for (var i=0; i<3; i++)
        {
            lock (lockObj)
            {
                while (onesTurn)
                    Monitor.Wait(lockObj);

                Console.Write("0");
                onesTurn = true;
                Monitor.Pulse(lockObj);
            }
        }
    }

    var t1 = Task.Run(PrintOnes);
    var t2 = Task.Run(PrintZeros);

    Task.WhenAll(t1, t2);
}
```

Output:

```
101010
```

---

## How Monitor.Wait Works

Inside:

```csharp
while (!condition)
    Monitor.Wait(lockObj);
```

`Monitor.Wait` performs three actions atomically:

1. Releases the lock
2. Suspends the current thread
3. Reacquires the lock before returning

This atomicity prevents race conditions.

---

## Why the While Loop?

Never write:

```csharp
if (!condition)
    Monitor.Wait(lockObj);
```

Always write:

```csharp
while (!condition)
    Monitor.Wait(lockObj);
```

Reason:

- Another thread may change the state before you reacquire the lock.
- Multiple waiting threads may be awakened by `PulseAll`.

The condition must always be rechecked.

---

# Solution 2: AutoResetEvent

## When to use

Best when the problem is simply:

> "Wake the other thread."

or:

> "It's your turn now."

The intent is often clearer than a Monitor-based solution.

## Complete Example

```csharp
using System;
using System.Threading;

class Program
{
    private static readonly AutoResetEvent oneTurn =
        new AutoResetEvent(true);

    private static readonly AutoResetEvent zeroTurn =
        new AutoResetEvent(false);

    static void Main()
    {
        var t1 = new Thread(PrintOnes);
        var t2 = new Thread(PrintZeros);

        t1.Start();
        t2.Start();

        t1.Join();
        t2.Join();
    }

    static void PrintOnes()
    {
        for (int i = 0; i < 3; i++)
        {
            oneTurn.WaitOne();

            Console.Write("1");

            zeroTurn.Set();
        }
    }

    static void PrintZeros()
    {
        for (int i = 0; i < 3; i++)
        {
            zeroTurn.WaitOne();

            Console.Write("0");

            oneTurn.Set();
        }
    }
}
```

or:

```csharp
using System;
using System.Threading;

public class MyPrintingClass
{
    public void Print101010()
    {
        using var onesTurn = new AutoResetEvent(true);
        using var zerosTurn = new AutoResetEvent(false);

        var t1 = new Thread(() => Print3Times("1", onesTurn, zerosTurn));
        var t2 = new Thread(() => Print3Times("0", zerosTurn, onesTurn));

        t1.Start();
        t2.Start();

        t1.Join();
        t2.Join();
    }

    private void Print3Times(string whatToPrint, AutoResetEvent gateToConsume, AutoResetEvent gateToRelease)
    {
        for (var i = 0; i < 3; i++)
        {
            gateToConsume.WaitOne();
            Console.Write(whatToPrint);
            gateToRelease.Set();
        }
    }
}
```

or:

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

public class MyPrintingClass
{
    public async Task Print101010()
    {
        using var onesTurn = new AutoResetEvent(true);
        using var zerosTurn = new AutoResetEvent(false);

        var t1 = Task.Run(() => Print3Times("1", onesTurn, zerosTurn));
        var t2 = Task.Run(() => Print3Times("0", zerosTurn, onesTurn));

        await Task.WhenAll(t1, t2);
    }

    private void Print3Times(string whatToPrint, AutoResetEvent gateToConsume, AutoResetEvent gateToRelease)
    {
        for (var i = 0; i < 3; i++)
        {
            gateToConsume.WaitOne();
            Console.Write(whatToPrint);
            gateToRelease.Set();
        }
    }
}
```

Output:

```
101010
```

---

## AutoResetEvent Mental Model

Think of an AutoResetEvent as a traffic light.

### Signaled State

```
GREEN
```

One waiting thread may proceed.

### Non-Signaled State

```
RED
```

Threads wait.

### Set()

Turns the light green.

### WaitOne()

Waits until the light becomes green, then automatically returns it to red.

This automatic reset is where the name comes from.

---

# Solution 3: SemaphoreSlim

## When to use

When access should be limited to a certain number of concurrent participants.

Can also be used for turn-taking.

Especially useful in async applications.

## Complete Example

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    private static readonly SemaphoreSlim oneTurn =
        new SemaphoreSlim(1, 1);

    private static readonly SemaphoreSlim zeroTurn =
        new SemaphoreSlim(0, 1);

    static async Task Main()
    {
        var task1 = PrintOnes();
        var task2 = PrintZeros();

        await Task.WhenAll(task1, task2);
    }

    static async Task PrintOnes()
    {
        for (int i = 0; i < 3; i++)
        {
            await oneTurn.WaitAsync();

            Console.Write("1");

            zeroTurn.Release();
        }
    }

    static async Task PrintZeros()
    {
        for (int i = 0; i < 3; i++)
        {
            await zeroTurn.WaitAsync();

            Console.Write("0");

            oneTurn.Release();
        }
    }
}
```

or:

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

public class MyPrintingClass
{
    public async Task Print101010()
    {
        using var onesTurn = new SemaphoreSlim(1, 1);
        using var zerosTurn = new SemaphoreSlim(0, 1);

        var t1 = Print3Times(onesTurn, zerosTurn, "1");
        var t2 = Print3Times(zerosTurn, onesTurn, "0");

        await Task.WhenAll(t1, t2);
    }

    private async Task Print3Times(SemaphoreSlim semaphoreToConsume, SemaphoreSlim semaphoreToRelease, string whatToPrint)
    {
        for(var i=0; i<3; i++)
        {
            await semaphoreToConsume.WaitAsync();
            Console.Write(whatToPrint);
            semaphoreToRelease.Release();
        }
    }
}
```

Output:

```
101010
```

---

# Understanding SemaphoreSlim

## Constructor

```csharp
new SemaphoreSlim(initialCount, maxCount)
```

Example:

```csharp
new SemaphoreSlim(3, 3)
```

Means:

```
3 available slots
maximum 3 slots
```

---

## WaitAsync()

When:

```csharp
await semaphore.WaitAsync();
```

happens:

- a slot is consumed
- count decreases by 1

If no slot is available:

- the caller waits

---

## Release()

```csharp
semaphore.Release();
```

Returns a slot to the semaphore.

---

# Monitor vs AutoResetEvent

## Monitor

Natural when waiting for shared state.

Example:

```csharp
while (queue.Count == 0)
    Monitor.Wait(_lock);
```

You are waiting for:

```
queue became non-empty
```

not merely a signal.

---

## AutoResetEvent

Natural when waiting for another thread.

Example:

```csharp
workerFinished.WaitOne();
```

You are waiting for:

```
somebody signaled me
```

---

# Producer-Consumer Example

A classical use case for `Monitor`.

Consumer:

```csharp
lock (_lock)
{
    while (_queue.Count == 0)
        Monitor.Wait(_lock);

    var item = _queue.Dequeue();
}
```

Producer:

```csharp
lock (_lock)
{
    _queue.Enqueue(item);

    Monitor.Pulse(_lock);
}
```

The important condition is not:

```
a signal occurred
```

The important condition is:

```
queue is no longer empty
```

This is exactly where `Monitor` feels natural.

---

# Multiple Conditions Under One Lock

Suppose threads wait for:

```
queue.Count > 0
databaseReady == true
cacheLoaded == true
```

With Monitor:

```csharp
lock (_lock)
{
    while (!databaseReady)
        Monitor.Wait(_lock);
}
```

State change:

```csharp
lock (_lock)
{
    databaseReady = true;
    Monitor.PulseAll(_lock);
}
```

Each thread wakes up and rechecks its own condition.

This approach scales better than juggling many different events.

---

# Pulse vs PulseAll

## Pulse

```csharp
Monitor.Pulse(lockObj);
```

Wakes one waiting thread.

Use when only one waiter should proceed.

---

## PulseAll

```csharp
Monitor.PulseAll(lockObj);
```

Wakes all waiting threads.

Each thread rechecks its condition and either:

- proceeds
- waits again

Useful when many different conditions may have changed.

---

# Events in This Context

When discussing synchronization, "events" means:

- `AutoResetEvent`
- `ManualResetEvent`
- `ManualResetEventSlim`

It does **not** mean C# language events:

```csharp
public event EventHandler SomethingHappened;
```

These are two completely different concepts.

---

# Practical Interview Rules

## Use lock

When you only need:

```
Mutual exclusion
```

---

## Use Monitor.Wait/Pulse

When you need:

```
Mutual exclusion
+
Waiting for a change in shared state
```

Examples:

- producer-consumer queues
- bounded buffers
- schedulers
- connection pools

---

## Use AutoResetEvent

When you need:

```
Simple thread-to-thread signaling
```

Examples:

- turn-taking
- "work completed"
- "wake the other thread"

---

## Use ManualResetEvent

When you need:

```
Open the gate for everyone
```

All waiting threads may continue once signaled.

---

## Use SemaphoreSlim

When you need:

```
N concurrent slots/resources
```

Examples:

- rate limiting
- connection pools
- throttling
- bounded concurrency

---

# Key Interview Takeaway

A strong answer is:

> `lock` provides mutual exclusion. `Monitor.Wait/Pulse` provides condition synchronization around shared state. `AutoResetEvent` provides explicit thread signaling. `SemaphoreSlim` controls access to a limited number of resources. For a simple `101010` turn-taking problem, `AutoResetEvent` is usually the clearest expression of intent, while `Monitor.Wait/Pulse` becomes more attractive as the synchronization logic grows around shared state and multiple conditions.