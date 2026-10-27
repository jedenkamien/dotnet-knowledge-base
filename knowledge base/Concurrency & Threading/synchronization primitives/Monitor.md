# Monitor (.NET) - Interview Overview

## 1. What is Monitor?

`Monitor` is the synchronization primitive behind `lock`.

```csharp
lock (_lock)
{
    DoWork();
}
```

is roughly equivalent to:

```csharp
Monitor.Enter(_lock);

try
{
    DoWork();
}
finally
{
    Monitor.Exit(_lock);
}
```

Use `lock` by default. Use `Monitor` when you need:

- `TryEnter`
- `Wait`
- `Pulse`
- `PulseAll`

---

## 2. Why Do We Need It?

`Monitor` provides **mutual exclusion**.

Without synchronization:

```csharp
counter++;
```

multiple threads may corrupt shared state.

With synchronization:

```csharp
lock (_lock)
{
    counter++;
}
```

only one thread may execute the critical section at a time.

---

## 3. How Does a Lock Work?

Typically:

```csharp
private readonly object _lock = new();
```

The object usually contains no data and serves only as a **lock identity**.

When a thread executes:

```csharp
Monitor.Enter(_lock);
```

it acquires the monitor associated with that object.

Important:

> The object is not protected. The code path is protected.

Locks are also **cooperative**:

```csharp
lock (_lock)
{
    Counter++;
}
```

is useless if some other code modifies `Counter` without taking the same lock.

---

## 4. Reentrancy

A thread may enter the same monitor multiple times.

```csharp
lock (_lock)
{
    lock (_lock)
    {
        Console.WriteLine("Hello");
    }
}
```

This does **not** deadlock.

Internally Monitor tracks:

```
Owner thread
Recursion count
```

Example:

```
Enter -> count = 1
Enter -> count = 2
Exit  -> count = 1
Exit  -> count = 0
```

The lock is released only when the count reaches zero.

### Why is this useful?

Without reentrancy:

```csharp
public void A()
{
    lock (_lock)
    {
        B();
    }
}

public void B()
{
    lock (_lock)
    {
        // work
    }
}
```

would deadlock because `A` already owns the lock.

Reentrancy allows:

- synchronized methods calling each other
- recursion
- more composable OO designs

---

## 5. Enter, Exit and TryEnter

Manual locking:

```csharp
Monitor.Enter(_lock);

try
{
    DoWork();
}
finally
{
    Monitor.Exit(_lock);
}
```

Always release locks in `finally`.

### TryEnter

Normal lock acquisition waits indefinitely.

```csharp
if (Monitor.TryEnter(_lock))
{
    try
    {
        DoWork();
    }
    finally
    {
        Monitor.Exit(_lock);
    }
}
```

Timeout version:

```csharp
Monitor.TryEnter(
    _lock,
    TimeSpan.FromSeconds(5));
```

If the timeout expires, it returns `false`.

### Interview Note

`TryEnter(timeout)` does not busy-loop for 5 seconds.

Think:

```
Wait for lock
or
Wait for timeout
```

The thread is blocked, not consuming CPU.

---

## 6. Wait, Pulse and PulseAll

`Monitor` is more than a mutex.

It also supports **condition synchronization**, allowing threads to wait for state changes.

### Wait

```csharp
lock (_lock)
{
    while (queue.Count == 0)
    {
        Monitor.Wait(_lock);
    }

    return queue.Dequeue();
}
```

`Wait` atomically:

```
1. Releases the lock
2. Blocks the thread
```

This allows other threads to acquire the lock and modify state.

### Pulse

```csharp
lock (_lock)
{
    queue.Enqueue(item);
    Monitor.Pulse(_lock);
}
```

Meaning:

```
Wake one waiting thread
```

### PulseAll

```csharp
Monitor.PulseAll(_lock);
```

Meaning:

```
Wake all waiting threads
```

### Which waiter wakes up?

Do not rely on any ordering.

The CLR does not guarantee:

- FIFO
- fairness
- oldest waiter wins

A pulsed thread must still reacquire the lock before continuing.

---

## 7. Why Use `while` Instead of `if`?

Wrong:

```csharp
if (queue.Count == 0)
{
    Monitor.Wait(_lock);
}
```

Correct:

```csharp
while (queue.Count == 0)
{
    Monitor.Wait(_lock);
}
```

Reason:

```
Thread wakes
↓
Another thread consumes item
↓
Condition becomes false again
```

Always recheck the condition after waking.

A good rule:

> `Monitor.Wait` almost always belongs inside a `while` loop.

---

## 8. Producer-Consumer Example

```csharp
private readonly object _lock = new();
private readonly Queue<int> _queue = new();

public void Produce(int item)
{
    lock (_lock)
    {
        _queue.Enqueue(item);
        Monitor.Pulse(_lock);
    }
}

public int Consume()
{
    lock (_lock)
    {
        while (_queue.Count == 0)
        {
            Monitor.Wait(_lock);
        }

        return _queue.Dequeue();
    }
}
```

This is the classic use case for `Wait` and `Pulse`.

---

## 9. Blocking Behaviour

| Operation | Blocks thread? | Consumes CPU while waiting? |
|-----------|---------------|-----------------------------|
| `Enter` | Yes | No |
| `TryEnter` | Yes | No |
| `Wait` | Yes | No |
| `SpinLock` | No | Yes |

`Monitor` typically parks the thread and lets the OS schedule something else.

---

## 10. Monitor vs lock

| Feature | lock | Monitor |
|----------|----------|----------|
| Mutual exclusion | ✅ | ✅ |
| Simpler syntax | ✅ | ❌ |
| TryEnter | ❌ | ✅ |
| Wait/Pulse | ❌ | ✅ |
| Reentrancy | ✅ | ✅ |

`lock` is usually preferred because it is safer and easier to read.

---

## 11. Monitor vs SemaphoreSlim

### Monitor

```
Capacity = 1
Synchronous
Thread-affine
```

```csharp
lock (_lock)
{
    DoWork();
}
```

### SemaphoreSlim

```
Capacity = N
Supports async
Not limited to single ownership
```

```csharp
await semaphore.WaitAsync();

try
{
    await DoWorkAsync();
}
finally
{
    semaphore.Release();
}
```

```
public class OrderProcessor
{
    private readonly SemaphoreSlim _gate = new(10);
     
    public async Task ProcessOrdersAsync(IEnumerable<Order> orders)
    {
        var tasks = orders.Select(ProcessSingleOrderAsync);
         
        await Task.WhenAll(tasks);
    }
     
    private async Task ProcessSingleOrderAsync(Order order)
    {
        await _gate.WaitAsync();
         
        try
        {
            await CallExternalApiAsync(order);
        }
        finally
        {
            _gate.Release();
        }
    }
}
```

Use `SemaphoreSlim` when you need async coordination or more than one concurrent entrant.

---

# Interview Cheat Sheet

Remember these seven points:

1. `lock` is syntax sugar over `Monitor.Enter/Exit`.
2. `Monitor` provides mutual exclusion.
3. Locks are cooperative; all participating code must use the same lock.
4. `Monitor` is reentrant.
5. `TryEnter` supports timeout/non-blocking acquisition.
6. `Wait` releases the lock and blocks the thread.
7. `Pulse` wakes one waiter, `PulseAll` wakes all waiters.

### Senior-level one-liner

> Monitor provides both mutual exclusion (`Enter/Exit`) and condition synchronization (`Wait/Pulse`). This allows threads not only to protect shared state but also to coordinate state changes such as producer-consumer workflows.