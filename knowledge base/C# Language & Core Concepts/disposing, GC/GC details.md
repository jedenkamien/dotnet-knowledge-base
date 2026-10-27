# Garbage Collection in .NET

## Overview
Automatic memory management system that frees unreachable heap objects. GC manages **heap only**; stack is managed automatically through scope unwinding. Developers allocate; GC deallocates (for heap objects).

## Memory Layout: Stack vs. Heap

**Stack**
- Stores value types (int, double, bool, struct) and references to objects
- Also stores local variables, method parameters
- Size per thread: ~1-2 MB (limited)
- LIFO (Last In First Out) — freed in reverse order when scope exits
- Cleanup: **Instant** when scope ends (no GC involved)
- Speed: Very fast allocation/deallocation

**Heap**
- Stores reference types (classes, strings, arrays)
- Size: Dynamic, grows as needed
- Allows arbitrary allocation/deallocation
- Cleanup: Later, during GC cycle (non-deterministic)
- Speed: Allocation fast, but cleanup involves GC overhead

**Stack Local Variables Size**
- Value types occupy their type size (e.g., `int` = 4 bytes, `double` = 8 bytes)
- Reference types: 8 bytes on 64-bit (pointer to heap object)
- Large structs (~1 MB+) as locals can cause `StackOverflowException` if allocated frequently or recursively

## Key Concepts

**Generations**
- Gen 0: Short-lived objects (checked frequently, ~256 KB - few MB)
- Gen 1: Medium-lived objects
- Gen 2: Long-lived objects (checked infrequently)
- Larger generations trigger less often → efficiency
- **Gen 0 collection frequency**: Few times/sec in light apps; dozens-hundreds/sec in high-throughput apps

**Roots**
- Static fields, local variables (on stack), method parameters
- GC traces from roots to find reachable objects
- Unreachable objects = eligible for collection

**Stop-the-World**
- GC pauses all threads during collection
- Gen 0 collections: Fast, minimal latency
- Gen 2 collections: Can pause for hundreds of milliseconds or more
- Modern .NET has background/concurrent GC to reduce pauses (default in modern .NET)

**Object Lifetime**
- When a method exits, local references are removed from stack instantly
- Heap objects become unreachable (if no other references exist) but are **not deallocated immediately**
- Deallocation happens during next GC cycle that includes that generation (non-deterministic)

## IDisposable and Unmanaged Resources

**Why IDisposable?**
- File handles, database connections, sockets are OS-level resources (not managed by GC)
- GC only manages managed heap; unmanaged resources are invisible to GC
- If not closed explicitly, resources accumulate → OS limit exhaustion → app crash
- Appears as a "memory leak" but GC can't fix it

**Deterministic Cleanup Pattern**
- `using` statement: Syntactic sugar for `try-finally`; calls `Dispose()` **instantly** when scope exits
- `IDisposable`: Implement to clean up unmanaged resources
- Finalizers: Run later (when GC collects object); too late for OS resource limits

```csharp
// ❌ Bad - resource never closed; OS limit exhaustion
var reader = new StreamReader("file.txt");
var data = reader.ReadLine();

// ✅ Good - Dispose() called instantly when scope exits
using (var reader = new StreamReader("file.txt")) {
    var data = reader.ReadLine();
}
```

## Modes

- **Workstation GC** (default): Single-threaded collection, optimized for latency
- **Server GC**: Multi-threaded collection per processor, optimized for throughput
- **Concurrent/Background GC**: Runs alongside app threads (modern .NET default, reduces pause times)

## Performance Considerations

**Common Heap Issues**

1. **Memory Leaks** — Unintentional references prevent GC collection
   - Static collections holding references
   - Event handlers not unsubscribed
   - Growing memory over time despite objects going out of scope

2. **Excessive Allocations** — Each allocation has overhead
   - High allocation rate forces frequent Gen 0 collections
   - GC overhead and CPU consumption increases
   - Latency spikes during collection

3. **Heap Fragmentation** — Especially LOH (Large Object Heap)
   - Objects ≥85KB aren't compacted (LOH uses different strategy)
   - Holes left behind waste space
   - Can cause allocation failures despite available memory

4. **Gen 2 Pauses** — Long stop-the-world pauses
   - Gen 0: Fast, ~1-10ms
   - Gen 2: Can pause for 100s of milliseconds
   - Problematic for latency-sensitive apps (trading, gaming, real-time)

5. **Long-lived Objects** — Objects reaching Gen 2 get collected rarely
   - If actually dead, waste memory until Gen 2 collection
   - Consider design to keep Gen 2 mostly long-term survivors

**Mitigation Strategies**

- **Reduce allocations** — Minimize heap pressure in hot paths
- **Use `using` statements** or `IDisposable` for unmanaged resources
- **Pooling** — Reuse objects instead of allocating new ones
- **Avoid large structs as locals** — Use `ref` or `in` parameters instead
- **Stack allocation** (`stackalloc`) — For small temporary arrays in performance-critical code
- **Monitor Gen 2 collection frequency** — Indicates potential memory leaks

## Tuning

Set via `.csproj` or runtime config:
```xml
<ServerGarbageCollection>true</ServerGarbageCollection>
<RetainVMMemory>true</RetainVMMemory>
<TieredCompilation>true</TieredCompilation>
```

## When It Matters

- High-frequency trading, real-time systems
- Game loops, animation rendering (Gen 2 pauses = frame drops)
- Server handling thousands of requests (allocation rate, memory leaks)
- Long-running applications (memory leak detection)
- Latency-sensitive code (minimize collection pauses)

## Quick Reference: Common Questions

**Q: When does memory get freed for local value types?**
A: Instantly when scope exits (stack unwinding). No GC involved.

**Q: When does a local reference type get freed?**
A: Stack reference removed instantly, but heap object freed later during GC cycle. Non-deterministic.

**Q: Why use `using`?**
A: Calls `Dispose()` instantly (deterministic) for unmanaged resources. GC is too late.

**Q: Can large structs cause problems?**
A: Yes, stack overflow if allocated frequently or recursively. Use classes or `in`/`ref` parameters.

**Q: What's a memory leak in .NET?**
A: Unintentional references preventing GC collection (static collections, event handlers, circular refs). Not freed memory — held memory.

**Q: Why not use stack for everything?**
A: Stack is small (~1-2 MB), LIFO (rigid scope rules), and limited per thread. Heap is flexible but needs GC overhead.

**Q: Why does Gen 0 collect frequently?**
A: Because Gen 0 fills quickly (~256 KB - few MB). Frequent collections keep Gen 0 efficient but each collection has overhead.