# Stack vs Heap Allocation

## Core Difference

**Stack**: Fast, automatic memory for local variables and method calls
**Heap**: Flexible, managed memory for objects with longer lifetime

## Stack

**Characteristics**
- LIFO (Last In, First Out) structure
- Thread-specific
- Automatically managed (no GC)
- Limited size (~1 MB default)
- Fast allocation/deallocation

**What goes on stack:**
- Local value types (`int`, `bool`, `struct`)
- Method parameters
- Return addresses
- References to heap objects (pointer itself)

```csharp
void Method() {
    int x = 5;           // Stack: value
    string s = "hello";  // Stack: reference → Heap: "hello" object
}  // x and reference s are popped from stack
```

**Stack Frame**: Each method call creates a frame with local variables. When method returns, frame is removed.

## Heap

**Characteristics**
- Dynamic memory pool
- Shared across threads
- Managed by Garbage Collector
- Larger (~2 GB 32-bit, virtually unlimited 64-bit)
- Slower allocation (GC overhead)

**What goes on heap:**
- All reference types (`class`, `string`, `array`, `delegate`)
- Boxed value types
- Static fields

```csharp
class Person { public int Age; }

void Method() {
    Person p = new Person();  // Stack: reference 'p'
                               // Heap: Person object
}  // Reference 'p' removed from stack
   // Person object remains on heap until GC collects it
```

## Memory Layout Example

```csharp
void Example() {
    int a = 10;                    // Stack
    Person p = new Person();       // p on stack, object on heap
    p.Age = 30;                    // Modifies heap object
    int[] arr = new int[5];        // arr reference on stack, array on heap
}

class Person { public int Age; }   // Age is inside heap object
```

**Visualization:**
```
Stack (Thread 1)          Heap
+-------------------+     +-------------------------+
| a: 10             |     | Person object           |
| p: 0x12AB --------|---->|   Age: 30               |
| arr: 0x45CD ------|---->| int[5] array            |
+-------------------+     |   [0, 0, 0, 0, 0]       |
                          +-------------------------+
```

## Value Type vs Reference Type Allocation

**Value types:**
- On **stack** if local variable/parameter
- On **heap** if field of a class or boxed

```csharp
struct Point { public int X, Y; }

void Method() {
    Point p = new Point();  // Stack (entire struct)
}

class Container {
    public Point p;  // Heap (part of Container object)
}
```

**Reference types:**
- Always on **heap** (object data)
- Reference stored wherever variable is (stack or heap)

## Performance Implications

**Stack advantages:**
- Extremely fast (pointer increment)
- No GC pressure
- Automatic cleanup
- CPU cache-friendly (locality)

**Heap advantages:**
- Flexible size
- Survives method scope
- Shareable across methods/threads

## Stack Overflow

**Cause**: Too many nested method calls or large local allocations

```csharp
void Recursive(int n) {
    int large = 100;  // Each call adds to stack
    if (n > 0) Recursive(n - 1);
}  // Deep recursion → StackOverflowException
```

**Solutions:**
- Use iteration instead of recursion
- Move large data to heap
- Increase stack size (rare)

## Modern C#: stackalloc & Span

**stackalloc**: Allocate arrays on stack (unsafe, no GC)
```csharp
Span<int> numbers = stackalloc int[100];  // Stack allocation, no heap
```

**Benefits**: Zero GC pressure, faster for hot paths

## When It Matters

- **High-frequency allocations**: Prefer stack (use structs)
- **Large objects**: Heap only option
- **Performance-critical code**: Stack + `Span<T>` / `stackalloc`
- **Thread-local data**: Stack is thread-safe by design
- **Object lifetime**: Stack for method scope, heap for longer

## Summary

| Aspect | Stack | Heap |
|--------|-------|------|
| Speed | Very fast | Slower |
| Size | Limited (~1 MB) | Large (GBs) |
| Lifetime | Method scope | Until GC collects |
| Management | Automatic | Garbage Collector |
| Data | Value types (local), references | Reference types, boxed values |
| Thread safety | Thread-specific | Shared (needs sync) |
