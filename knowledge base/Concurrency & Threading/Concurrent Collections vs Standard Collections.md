# .NET Concurrent Collections vs Standard Collections

## Overview

The .NET `System.Collections.Concurrent` namespace provides thread-safe collections designed for concurrent access from multiple threads without requiring external locking in common scenarios.

A key design principle is that concurrent collections generally avoid throwing exceptions for expected race conditions. Instead, they expose atomic "Try*" methods that return `bool` to indicate success or failure.

---

# ConcurrentQueue<T>

A thread-safe FIFO (First In, First Out) queue.

## Main Operations

### Enqueue

Adds an item to the end of the queue.

```csharp
queue.Enqueue(item);
```

Characteristics:

- Thread-safe
- Multiple threads may call simultaneously
- Does not fail because another thread modified the queue
- Does not have a `TryEnqueue` method

### TryDequeue

Attempts to remove the next item.

```csharp
if (queue.TryDequeue(out var item))
{
    // item obtained
}
```

Characteristics:

- Returns `true` if an item was removed
- Returns `false` if the queue is empty
- Does not throw because another thread removed an item first

### TryPeek

Attempts to inspect the next item without removing it.

```csharp
queue.TryPeek(out var item);
```

---

# ConcurrentDictionary<TKey, TValue>

A thread-safe key/value collection.

Unlike `Dictionary<TKey,TValue>`, it is designed around atomic operations.

## Main Operations

### TryAdd

```csharp
dictionary.TryAdd("Anna", 5);
```

Returns:

- `true` if added
- `false` if the key already exists

### TryGetValue

```csharp
dictionary.TryGetValue("Anna", out var value);
```

### TryUpdate

Atomically updates a value only if it currently equals an expected value.

```csharp
dictionary.TryUpdate(
    "Anna",
    newValue: 6,
    comparisonValue: 5);
```

Equivalent conceptually to Compare-And-Swap (CAS).

### TryRemove

```csharp
dictionary.TryRemove("Anna", out var value);
```

Returns:

- `true` if removed
- `false` if key not found

### AddOrUpdate

```csharp
dictionary.AddOrUpdate(
    key,
    1,
    (_, oldValue) => oldValue + 1);
```

Atomic insert-or-modify operation.

### GetOrAdd

```csharp
dictionary.GetOrAdd(
    key,
    _ => CreateValue());
```

Atomically gets an existing value or creates one.

## Indexer

```csharp
dictionary[key] = value;
```

Thread-safe.

Behavior:

- Adds if key does not exist
- Overwrites if key exists

No information is provided about which of those happened.

## Missing Methods

There are no simple:

```csharp
Add(...)
Remove(...)
Update(...)
```

methods intended for concurrent usage.

Instead, the API exposes atomic `Try*` and combined operations.

---

# ConcurrentBag<T>

A thread-safe unordered collection.

Think of it as a "bag of items" where insertion and retrieval order does not matter.

## Main Operations

### Add

```csharp
bag.Add(item);
```

Adds an item.

There is no `TryAdd`.

### TryTake

```csharp
bag.TryTake(out var item);
```

Attempts to remove an item.

Returns:

- `true` if successful
- `false` if empty

### TryPeek

```csharp
bag.TryPeek(out var item);
```

Inspects an item without removing it.

### Count

```csharp
var count = bag.Count;
```

Available, but should not be relied upon for subsequent actions in multithreaded code because the collection can change immediately afterward.

## Missing Methods

No support for removing a specific item, such as:

```csharp
Remove(item)
TryRemove(item)
```

Only "take some item" semantics exist.

## Typical Use Cases

- Object pools
- Work-stealing scenarios
- Collections where ordering is irrelevant

---

# ConcurrentStack<T>

A thread-safe LIFO (Last In, First Out) stack.

## Main Operations

### Push

```csharp
stack.Push(item);
```

### TryPop

```csharp
stack.TryPop(out var item);
```

### TryPeek

```csharp
stack.TryPeek(out var item);
```

Typical use case:

- DFS algorithms
- Stack-based schedulers
- LIFO processing

---

# Meaning of "Try" Methods

In concurrent collections, `Try*` does not mean:

> "Try and detect a concurrency conflict."

Instead, it means:

> "Attempt an operation that may legitimately fail."

Examples:

```csharp
int.TryParse(...)
dictionary.TryAdd(...)
queue.TryDequeue(...)
monitor.TryEnter(...)
```

Failure is expected and represented by `false`, not by an exception.

---

# Standard Queue<T>

`Queue<T>` is NOT thread-safe.

## Dequeue on Empty Queue

```csharp
queue.Dequeue();
```

Throws:

```text
InvalidOperationException
```

if the queue is empty.

## Race Condition Example

```csharp
if (queue.Count > 0)
{
    var item = queue.Dequeue();
}
```

Two threads may interleave:

1. Thread A checks `Count > 0`
2. Thread B removes the last item
3. Thread A calls `Dequeue()`
4. `InvalidOperationException` is thrown

The check and dequeue are not atomic.

---

# Collection Enumeration During Modification

Standard collections such as:

```csharp
Queue<T>
Dictionary<TKey,TValue>
List<T>
```

may throw:

```text
InvalidOperationException:
Collection was modified; enumeration operation may not execute.
```

when one thread enumerates while another modifies.

Concurrent collections are specifically designed to avoid these common multithreading issues.

---

# Choosing the Correct Concurrent Collection

| Requirement | Recommended Collection |
|------------|------------------------|
| FIFO processing | ConcurrentQueue<T> |
| LIFO processing | ConcurrentStack<T> |
| Key/value storage | ConcurrentDictionary<TKey,TValue> |
| Ordering does not matter | ConcurrentBag<T> |

General guideline:

- Use `ConcurrentQueue<T>` for work queues.
- Use `ConcurrentStack<T>` for stack semantics and DFS-like processing.
- Use `ConcurrentDictionary<TKey,TValue>` for shared state indexed by key.
- Use `ConcurrentBag<T>` when items are simply stored and consumed without any ordering requirements.