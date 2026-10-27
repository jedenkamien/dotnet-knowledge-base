# Parallel.ForEach and MaxDegreeOfParallelism

## Overview

`Parallel.ForEach` is a high-level API from the **Task Parallel Library (TPL)** used to process elements of a collection concurrently.

It is designed primarily for **CPU-bound workloads** where many independent items can be processed in parallel.

```csharp
Parallel.ForEach(items, item =>
{
    Process(item);
});
```

Internally, `Parallel.ForEach` uses:
- The .NET Thread Pool
- TPL task scheduling
- Work partitioning and load balancing

The runtime automatically distributes iterations across available worker threads.

---

## Basic Example

```csharp
Parallel.ForEach(
    Enumerable.Range(1, 10),
    item =>
    {
        Console.WriteLine(
            $"Item {item} on thread {Thread.CurrentThread.ManagedThreadId}");
    });
```

Characteristics:

- Iterations may execute simultaneously.
- Execution order is not guaranteed.
- Different iterations may run on the same or different threads.

---

## Limiting Concurrency with MaxDegreeOfParallelism

The number of concurrently executing iterations can be limited using `ParallelOptions`.

```csharp
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = 3
};

Parallel.ForEach(
    Enumerable.Range(1, 10),
    options,
    item =>
    {
        Thread.Sleep(2000);
        Console.WriteLine($"Processed {item}");
    });
```

### Meaning

```csharp
MaxDegreeOfParallelism = 3
```

means:

> At most 3 iterations may execute concurrently.

It does **not** mean:
- Exactly 3 threads are created.
- Exactly 3 threads are always active.
- One dedicated thread exists per iteration.

The runtime remains free to schedule work as it sees fit.

---

## Example Timeline

For 10 items:

```text
1 2 3 4 5 6 7 8 9 10
```

with:

```csharp
MaxDegreeOfParallelism = 3
```

a possible execution flow is:

```text
Time 0s:
1 2 3 running

Time 2s:
4 5 6 running

Time 4s:
7 8 9 running

Time 6s:
10 running

Time 8s:
finished
```

Only three iterations run simultaneously at any given moment.

---

## Parallel.ForEach vs Task.Run

### Parallel.ForEach

```csharp
Parallel.ForEach(items, item =>
{
    Process(item);
});
```

Purpose:

- Data parallelism
- Processing collections
- Built-in partitioning
- Built-in concurrency control

Advantages:

- Optimized for CPU-bound collection processing
- Minimal code
- Supports `MaxDegreeOfParallelism`

---

### Task.Run

```csharp
var tasks = items.Select(item =>
    Task.Run(() => Process(item)));

await Task.WhenAll(tasks);
```

Purpose:

- Scheduling individual units of work
- General task creation
- Greater flexibility

Characteristics:

- Every task is separately queued.
- The developer manages the task collection.
- No built-in throttling unless implemented manually.

---

## When to Use Which

### Prefer Parallel.ForEach

When:

- Processing large collections
- Performing CPU-bound computations
- Applying the same operation to many items

Examples:

- Image processing
- Data transformations
- Calculations
- Parsing large datasets

---

### Prefer Task.Run

When:

- Launching a few independent operations
- Wrapping synchronous work
- Integrating with task-based APIs

Examples:

```csharp
Task.Run(() => CalculateReport());
Task.Run(() => GeneratePdf());
Task.Run(() => CompressArchive());
```

---

## CPU-Bound vs I/O-Bound

### CPU-Bound

Examples:

- Compression
- Encryption
- Mathematical calculations
- Image rendering

Good fit:

```csharp
Parallel.ForEach(...)
```

because multiple CPU cores can be utilized efficiently.

---

### I/O-Bound

Examples:

- Database access
- HTTP calls
- File downloads

Prefer asynchronous APIs:

```csharp
await httpClient.GetAsync(...);
await dbContext.SaveChangesAsync();
```

rather than creating extra threads.

---

## Why Task.Run Is Often Discouraged in ASP.NET

Example:

```csharp
await Task.Run(() => CpuHeavyWork());
```

In ASP.NET, request handling already runs on a Thread Pool thread.

Calling `Task.Run` often means:

1. Current Thread Pool thread starts the request.
2. Work is delegated to another Thread Pool thread.
3. The original thread waits for completion.

This introduces unnecessary scheduling overhead.

For collection-oriented CPU-bound work, approaches such as:

```csharp
Parallel.ForEach(...)
```

or

```csharp
Parallel.ForEachAsync(...)
```

are typically more appropriate.

---

## Interview Notes

### What does MaxDegreeOfParallelism control?

The maximum number of concurrently executing iterations.

### Does it specify the number of threads?

No. It specifies concurrency, not thread count.

### Is iteration order guaranteed?

No.

### Is Parallel.ForEach part of TPL?

Yes.

### What is it optimized for?

CPU-bound processing of collections.

### What's the main difference from Task.Run?

- `Task.Run` schedules individual tasks.
- `Parallel.ForEach` parallelizes work across a collection.

### When should Parallel.ForEach usually be preferred?

When the same CPU-intensive operation must be performed on many independent items.

### Short Interview Answer

> `Parallel.ForEach` is a TPL construct designed for data parallelism. It automatically distributes collection items across Thread Pool threads and can limit concurrency via `MaxDegreeOfParallelism`. Unlike `Task.Run`, which schedules individual tasks, `Parallel.ForEach` is optimized for CPU-bound processing of large collections.