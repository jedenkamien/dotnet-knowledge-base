# Concurrent Collections (Thread-Safe)

## Why
Standard collections (`List<T>`, `Dictionary<K,V>`) are **not thread-safe**. Concurrent reads + writes from multiple threads cause corruption or exceptions.

## ConcurrentDictionary<TKey,TValue>

The most widely used. Lock-free reads, fine-grained locking on writes (striped buckets).

```csharp
var cache = new ConcurrentDictionary<string, User>();

cache.TryAdd("alice", user);
cache.TryGetValue("alice", out var u);
cache.TryRemove("alice", out _);

// Atomic get-or-add
var user = cache.GetOrAdd("bob", key => repo.Load(key));

// Atomic update
cache.AddOrUpdate("bob",
    key => new User(key),                  // add
    (key, existing) => existing.WithHit()); // update
```

**Pitfall:** the factory in `GetOrAdd` may run more than once under contention; only the result of the winner is stored. If creation is expensive, wrap in `Lazy<T>`:

```csharp
var lazy = cache.GetOrAdd(key, k => new Lazy<User>(() => Load(k)));
return lazy.Value;
```

## Other Concurrent Types

| Type | Use case |
|------|----------|
| `ConcurrentQueue<T>` | FIFO producer/consumer |
| `ConcurrentStack<T>` | LIFO |
| `ConcurrentBag<T>` | Unordered, optimized when same thread adds + removes |
| `BlockingCollection<T>` | Bounded queue with blocking `Take`/`Add` |
| `Channel<T>` (newer) | Modern async producer/consumer, preferred over BlockingCollection |

## Channels (preferred for async)

```csharp
var channel = Channel.CreateBounded<WorkItem>(100);

// Producer
await channel.Writer.WriteAsync(item);

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync())
    Process(item);
```

## Alternatives

- **`ImmutableDictionary<K,V>`**: every modification returns a new instance — no locking needed, but writes cost more.
- **`lock` + `Dictionary`**: simpler when contention is low and operations are coarse.
- **`ReaderWriterLockSlim`**: many readers, few writers.

## Rule of Thumb

- Mostly reads, occasional writes → `ConcurrentDictionary`
- Producer/consumer pipeline → `Channel<T>`
- Snapshot/immutable view needed → `ImmutableDictionary`
- Single-writer → plain dict + `lock` is often fastest
