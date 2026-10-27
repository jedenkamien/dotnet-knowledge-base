# Classes vs Structures

## Core Difference

| Feature | Class | Struct |
|---------|-------|--------|
| Type | Reference type | Value type |
| Storage | Heap | Stack (if local) or inline |
| Default | `null` | Zero-initialized |
| Inheritance | Can inherit & be inherited | Cannot inherit (except interfaces) |
| Assignment | Copies reference | Copies entire value |

## Memory & Performance

**Class**
```csharp
class Point { public int X, Y; }
Point p1 = new Point { X = 1, Y = 2 };
Point p2 = p1;  // Both point to same object
p2.X = 10;      // p1.X is now 10
```
- Heap allocation → GC pressure
- Indirection overhead (pointer dereference)

**Struct**
```csharp
struct Point { public int X, Y; }
Point p1 = new Point { X = 1, Y = 2 };
Point p2 = p1;  // Copies entire struct
p2.X = 10;      // p1.X is still 1
```
- Stack allocation (faster)
- No GC overhead for local structs
- Copying can be expensive for large structs

## When to Use Struct

✅ **Use struct when:**
- Small data (≤16 bytes recommended)
- Logically represents a single value (e.g., `Point`, `Color`, `DateTime`)
- Immutable
- Rarely boxed
- Short-lived

**Examples from .NET:**
- `int`, `double`, `bool` (primitives)
- `DateTime`, `TimeSpan`, `Guid`
- `ValueTuple<T1, T2>`

## When to Use Class

✅ **Use class when:**
- Complex behavior or state
- Large data (>16 bytes)
- Needs inheritance
- Identity matters (two objects with same data are distinct)
- Long-lived

## Common Pitfalls

**Mutable structs** (avoid!)
```csharp
struct Counter {
    public int Value;
    public void Increment() => Value++; // Modifies copy, not original
}
Counter c = new Counter();
c.Increment();  // Works
someList[0].Increment();  // NO EFFECT (modifies temporary copy)
```

**Boxing overhead**
```csharp
object obj = myStruct;  // Boxing → heap allocation
```

## Modern Features

**Readonly struct (C# 7.2+)**
```csharp
readonly struct Point {
    public int X { get; }
    public int Y { get; }
    public Point(int x, int y) => (X, Y) = (x, y);
}
```

**Record struct (C# 10+)**
```csharp
record struct Point(int X, int Y);  // Immutable by default
```

## Rule of Thumb

- **Default to class** unless you have a specific reason
- Struct for **small, immutable, value-like** data
- Benchmark if performance is critical
