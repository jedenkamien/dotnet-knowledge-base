# Threads and Tasks in .NET

## Creating concurrent work

### Thread

```csharp
var t = new Thread(() =>
{
    for (int i = 0; i < 3; i++)
    {
        Console.Write("0");
    }
});

t.Start();
t.Join();
```

### Task

```csharp
var task = Task.Run(() =>
{
    for (int i = 0; i < 3; i++)
    {
        Console.Write("0");
    }
});

await task;
```

The output order is generally **not deterministic** when multiple threads/tasks write concurrently.

---

# Process vs Thread

Interviewers often expect this distinction:

| Concept | Meaning |
|----------|----------|
| Process | Running application with its own memory space |
| Thread | Execution path inside a process |
| Task | Logical unit of work |

`Monitor`, `lock`, `SemaphoreSlim` etc. synchronize **threads within one process**.

To synchronize multiple processes, use:

- Named `Mutex`
- Named `Semaphore`
- Pipes
- Shared memory
- Other IPC mechanisms

---

# Thread Lifecycle

## new Thread(...)

```csharp
var t = new Thread(Work);
```

This:

- Creates a managed `Thread` object
- Stores the delegate to run
- Does **not** start execution

Think: "create thread object".

---

## Start()

```csharp
t.Start();
```

This:

- Makes the thread runnable
- Causes the CLR/OS to create and schedule an OS thread
- Returns immediately

Important:

```csharp
t.Start();
Console.WriteLine("Main");
```

does **not** guarantee whether `Main` or the thread's code runs first.

---

## Join()

```csharp
t.Join();
```

Blocks the calling thread until `t` finishes.

Example:

```csharp
var t = new Thread(() =>
{
    Thread.Sleep(2000);
});

t.Start();
t.Join();

Console.WriteLine("Done");
```

Output:

```
Done
```

appears only after the worker thread completes.

---

# Useful Thread Members

## Sleep

```csharp
Thread.Sleep(1000);
```

Blocks the current thread for one second.

---

## IsAlive

```csharp
bool running = t.IsAlive;
```

Indicates whether the thread is still executing.

---

## Interrupt

```csharp
t.Interrupt();
```

Interrupts a thread blocked in operations such as:

```csharp
Sleep()
Wait()
Join()
```

Rarely used in modern code.

---

# Tasks

Tasks are a higher-level abstraction than threads.

A useful interview sentence:

> A thread is a physical execution resource managed by the OS. A task is a logical unit of work managed by the Task Parallel Library.

---

## Task.Run

```csharp
var task = Task.Run(() =>
{
    Console.WriteLine("Hello");
});
```

Roughly:

```
Create Task
↓
Queue work to ThreadPool
↓
Return immediately
↓
ThreadPool thread executes work
```

Important:

```
Task != Thread
```

A task usually executes on a ThreadPool thread.

---

# Why Task.Run Is Preferred

```csharp
Task.Run(Work);
```

typically beats:

```csharp
var task = new Task(Work);
task.Start();
```

because it:

- Is shorter
- Creates and schedules in one step
- Uses sensible defaults
- Avoids forgetting `Start()`

---

# Creating a Task First and Starting Later

Sometimes you will see:

```csharp
var task = new Task(Work);

// later
task.Start();
```

This uses the `Created` state before execution begins.

Useful to know, but uncommon in production code.

---

# StartNew

```csharp
Task.Factory.StartNew(Work);
```

Older and more configurable than `Task.Run`.

Allows:

- Custom schedulers
- Creation options
- Additional control

Usually avoid it unless you specifically need that control.

---

# Waiting for Tasks

## Wait

```csharp
task.Wait();
```

Blocks the current thread.

Equivalent in spirit to:

```csharp
thread.Join();
```

---

## Result

```csharp
var value = task.Result;
```

Also blocks.

---

## await

```csharp
await task;
```

Preferred approach.

Key distinction:

| API | Blocks thread? |
|-------|-------|
| `Wait()` | Yes |
| `Result` | Yes |
| `await` | No |

This is one of the most common async interview questions.

---

# Can a Task Run on Different Threads?

Yes.

For example:

```csharp
await Task.Delay(1000);
```

Before `await`, the task may execute on one thread.

After `await`, continuation may execute on:

- The same thread
- A different ThreadPool thread
- The captured UI thread (if applicable)

A thread is tied to one OS execution path.

A task is not.

---

# Scheduling Is Not Deterministic

Consider:

```csharp
var task = Task.Run(() =>
{
    Console.WriteLine("Hello");
});

Console.WriteLine("World");
```

Possible outputs:

```
World
Hello
```

or

```
Hello
World
```

because `Task.Run` only queues the work.

The scheduler decides when execution actually begins.

---

# Thread vs Task Summary

| Thread | Task |
|----------|----------|
| OS execution thread | Logical unit of work |
| More expensive | Usually cheaper |
| Created with `new Thread()` | Usually created with `Task.Run()` |
| Dedicated thread | Usually uses ThreadPool |
| Wait using `Join()` | Wait using `await` |

## Interview-ready Summary

> `new Thread()` creates a thread object. `Start()` makes it runnable. `Join()` blocks until it finishes. `Task.Run()` creates a task and queues it to the ThreadPool. A Task is not a Thread; it represents work, not the physical execution resource. In modern .NET, Tasks and `async`/`await` are usually preferred over manually creating threads.