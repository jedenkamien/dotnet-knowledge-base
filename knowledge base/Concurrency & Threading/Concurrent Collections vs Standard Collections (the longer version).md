# .NET Concurrent Collections vs Standard Collections

## Overview

The .NET `System.Collections.Concurrent` namespace contains thread-safe collections that can safely be accessed by multiple threads without external locks in most scenarios.

A key idea is:

- Standard collections often throw exceptions when an operation is invalid.
- Concurrent collections usually expose `Try*` methods and return `false` when an operation cannot be performed.

This makes them much easier to use in highly concurrent code.

---

# ConcurrentQueue<T>

A thread-safe FIFO (First In, First Out) queue.

## Real-Life Example: Order Processing System

Incoming orders are placed into a queue and worker threads process them one by one.

### Creating and Filling the Queue

```csharp
var orders = new ConcurrentQueue<string>();

orders.Enqueue("Order #1001");
orders.Enqueue("Order #1002");
orders.Enqueue("Order #1003");
```

Current queue:

```text
Order #1001
Order #1002
Order #1003
```

### Peek at the Next Order

```csharp
if (orders.TryPeek(out var nextOrder))
{
    Console.WriteLine($"Next order: {nextOrder}");
}
```

Output:

```text
Next order: Order #1001
```

Queue remains unchanged.

### Process Orders

```csharp
while (orders.TryDequeue(out var order))
{
    Console.WriteLine($"Processing {order}");
}
```

Output:

```text
Processing Order #1001
Processing Order #1002
Processing Order #1003
```

### Queue Is Empty

```csharp
var success = orders.TryDequeue(out var order);

Console.WriteLine(success);
```

Output:

```text
False
```

No exception is thrown.

### Typical Production Scenario

```csharp
public async Task WorkerAsync()
{
    while (true)
    {
        if (orders.TryDequeue(out var order))
        {
            await ProcessOrder(order);
        }
    }
}
```

Many worker threads can execute this simultaneously.

Only one worker will receive a particular order.

---

# ConcurrentDictionary<TKey, TValue>

A thread-safe key/value store.

## Real-Life Example: Logged-In Users

Imagine a web application storing the number of requests performed by each user.

### Creating the Dictionary

```csharp
var userRequests =
    new ConcurrentDictionary<string, int>();
```

### Add Users

```csharp
userRequests.TryAdd("Anna", 10);
userRequests.TryAdd("John", 3);
userRequests.TryAdd("Kate", 7);
```

Content:

```text
Anna -> 10
John -> 3
Kate -> 7
```

### Add Existing User

```csharp
var added = userRequests.TryAdd("Anna", 20);

Console.WriteLine(added);
```

Output:

```text
False
```

Anna already exists.

---

### Read Value

```csharp
if (userRequests.TryGetValue("John", out var requests))
{
    Console.WriteLine(requests);
}
```

Output:

```text
3
```

---

### Increment User Counter

Suppose John sends another request.

```csharp
userRequests.AddOrUpdate(
    "John",
    1,
    (_, oldValue) => oldValue + 1);
```

Content:

```text
Anna -> 10
John -> 4
Kate -> 7
```

This is atomic and safe even if multiple threads update John simultaneously.

---

### Add User If Missing

```csharp
var requests = userRequests.GetOrAdd(
    "Mark",
    0);
```

Content:

```text
Anna -> 10
John -> 4
Kate -> 7
Mark -> 0
```

---

### Conditional Update

Suppose we expect John's current value to be 4.

```csharp
var updated =
    userRequests.TryUpdate(
        "John",
        5,
        4);
```

Meaning:

```text
Change 4 -> 5
Only if current value is still 4
```

Output:

```text
True
```

If another thread already changed it to 8:

```text
False
```

---

### Remove User

```csharp
userRequests.TryRemove(
    "Kate",
    out var removedValue);
```

Output:

```text
removedValue = 7
```

Current content:

```text
Anna -> 10
John -> 5
Mark -> 0
```

---

### Indexer

```csharp
userRequests["John"] = 100;
```

Content:

```text
Anna -> 10
John -> 100
Mark -> 0
```

Unlike `TryAdd`, this overwrites an existing value.

---

# ConcurrentBag<T>

A thread-safe unordered collection.

The key concept:

```text
I have items.
I don't care about ordering.
```

## Real-Life Example: Pool of Available Database Connections

Imagine a system maintaining free connections that may be reused.

### Create Bag

```csharp
var freeConnections =
    new ConcurrentBag<string>();
```

### Add Connections

```csharp
freeConnections.Add("Connection-1");
freeConnections.Add("Connection-2");
freeConnections.Add("Connection-3");
```

### Count

```csharp
Console.WriteLine(
    freeConnections.Count);
```

Output:

```text
3
```

---

### Peek

```csharp
if (freeConnections.TryPeek(out var connection))
{
    Console.WriteLine(connection);
}
```

Possible output:

```text
Connection-3
```

or:

```text
Connection-1
```

Ordering is not guaranteed.

---

### Take Connection

```csharp
if (freeConnections.TryTake(out var connection))
{
    Console.WriteLine(
        $"Using {connection}");
}
```

Possible output:

```text
Using Connection-2
```

---

### Return Connection to Pool

```csharp
freeConnections.Add(connection);
```

A common object-pool pattern.

---

### Empty the Bag

```csharp
while (freeConnections.TryTake(
    out var connection))
{
    Console.WriteLine(connection);
}
```

Possible output:

```text
Connection-2
Connection-1
Connection-3
```

No ordering guarantees exist.

---

### What You Cannot Do

There is no:

```csharp
freeConnections.Remove("Connection-2");
```

or

```csharp
freeConnections.TryRemove("Connection-2");
```

A bag allows you to take *some* item, not a specific item.

---

# ConcurrentStack<T>

A thread-safe stack.

LIFO (Last In, First Out).

## Real-Life Example: Browser Navigation History

### Create Stack

```csharp
var history =
    new ConcurrentStack<string>();
```

### Push Pages

```csharp
history.Push("/home");
history.Push("/products");
history.Push("/checkout");
```

Visual view:

```text
TOP
/checkout
/products
/home
```

---

### Peek

```csharp
history.TryPeek(out var page);
```

Result:

```text
/checkout
```

---

### Pop

```csharp
history.TryPop(out var page);
```

Result:

```text
/checkout
```

Remaining:

```text
TOP
/products
/home
```

---

### Pop Again

```csharp
history.TryPop(out var page);
```

Result:

```text
/products
```

Remaining:

```text
TOP
/home
```

---

### Empty Stack

```csharp
history.TryPop(out var page);
history.TryPop(out page);

var success =
    history.TryPop(out page);
```

Output:

```text
False
```

No exception is thrown.

---

# Standard Queue<T>

NOT thread-safe.

## Normal Usage

```csharp
var queue = new Queue<string>();

queue.Enqueue("A");
queue.Enqueue("B");

var item = queue.Dequeue();
```

Result:

```text
A
```

---

## Dequeue on Empty Queue

```csharp
var queue = new Queue<string>();

queue.Dequeue();
```

Throws:

```text
InvalidOperationException
```

---

## Multithreading Race Condition

Suppose:

```text
queue contains exactly one item
```

Thread A:

```csharp
if (queue.Count > 0)
{
    // context switch
}
```

Thread B:

```csharp
queue.Dequeue();
```

Thread A:

```csharp
queue.Dequeue();
```

Result:

```text
InvalidOperationException
```

Thread A checked the count earlier, but another thread removed the item before `Dequeue()` executed.

---

# Meaning of "Try"

The pattern appears everywhere in .NET:

```csharp
int.TryParse(...)
queue.TryDequeue(...)
dictionary.TryAdd(...)
dictionary.TryRemove(...)
stack.TryPop(...)
bag.TryTake(...)
```

The meaning is:

```text
This operation may legitimately fail.
Instead of throwing an exception,
return a bool indicating success.
```

Examples:

```csharp
if (queue.TryDequeue(out var item))
{
    Process(item);
}
```

```csharp
if (dictionary.TryRemove(id, out var value))
{
    Cleanup(value);
}
```

```csharp
if (stack.TryPop(out var task))
{
    Execute(task);
}
```

This pattern is especially valuable in concurrent code because races are expected and should not be treated as exceptional situations.

---

# Quick Selection Guide

Use:

```text
ConcurrentQueue<T>
```

when tasks must be processed in arrival order.

Example:

```text
Orders
Background jobs
Messages
```

Use:

```text
ConcurrentStack<T>
```

when most recently added items should be processed first.

Example:

```text
DFS algorithms
Undo operations
Navigation history
```

Use:

```text
ConcurrentDictionary<TKey,TValue>
```

when you need shared state indexed by a key.

Example:

```text
User sessions
Caches
Statistics
Configuration values
```

Use:

```text
ConcurrentBag<T>
```

when ordering is irrelevant.

Example:

```text
Connection pools
Object pools
Reusable resources
Work-stealing scenarios
```