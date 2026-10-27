# String Operations and Handling

## Key Characteristics

### Immutability
Every modification creates a new string — the original is never changed:
```csharp
string s = "Hello";
s += " World";  // New string allocated, old "Hello" discarded
```

This is a deliberate design choice, not a limitation. Immutability is an asset in many scenarios:
- **Dictionary keys** — hash is computed once on insertion; a mutable key could change its hash mid-flight, making it unfindable
- **Thread safety** — multiple threads can read the same string with zero locking
- **Interning / caching** — safe to share one instance across many references; no one can corrupt the shared copy
- **Security** — a file path or connection string passed to a security check cannot be mutated by another thread between the check and the use (prevents TOCTOU attacks)
- **Predictability** — methods like `Replace`, `Trim` return new strings; the original is never silently changed by a called method

### Reference Type That Behaves Like a Value Type
String is a reference type stored on the heap, but `==` compares **content**, not reference. This is just operator overloading — under the hood they are still references. The design rationale:

- **Too large to copy on every assignment** (like value types do) → reference type
```csharp
// If string were a value type, passing it to a method would copy every character.
// A 1MB string passed 100 times = 100MB of stack/copy overhead.
void Print(string s) { ... }  // Just passes a reference — no copy
```

- **Too dangerous to mutate** (shared references would corrupt each other) → immutable
```csharp
// If strings were mutable, this would be a disaster:
string a = "hello";
string b = a;         // Both point to same object
// b.MutateInPlace("world");  // Would silently change a too!
// Instead, immutability guarantees a stays "hello"
```

- **Too natural to compare by content** → `==` overloaded
```csharp
string x = "abc";
string y = "abc";      // Possibly a different object in memory
bool same = (x == y);  // true — you almost never want reference comparison for strings
// object.ReferenceEquals(x, y) would give the "raw" reference check if ever needed
```

## Common Operations

### Creation & Concatenation
```csharp
string s1 = "Hello" + " World";           // Simple concat
string s2 = string.Concat("A", "B", "C"); // Multiple strings
string s3 = string.Join(", ", array);     // With separator
string s4 = $"Name: {name}, Age: {age}";  // Interpolation (preferred)
```

### Manipulation
```csharp
s.Substring(0, 5);       // Extract portion (allocates new string)
s.Replace("old", "new"); // Replace text
s.Trim();                // Remove whitespace
s.ToLower() / ToUpper();
s.Split(',');            // To array
```

### Comparison
```csharp
s1 == s2                                           // Ordinal comparison (content equality)
s1.Equals(s2, StringComparison.OrdinalIgnoreCase)  // Case-insensitive
string.Compare(s1, s2)                             // Returns -1, 0, 1
```

## Performance Considerations

### StringBuilder for Loops

Concatenating strings in a loop with `+=` is O(n²) in **CPU copy work** (not just allocations):
- Each `+=` creates a new string by copying all existing characters plus the new ones
- Iteration k copies k characters → total copies = 1+2+...+n = n(n+1)/2 → O(n²)
- The old strings are short-lived and collected cheaply by Gen0 GC, but the copy work already happened

```csharp
// BAD: O(n²) character copy work
string result = "";
for (int i = 0; i < 1000; i++)
    result += i.ToString();

// GOOD: O(n) with single growable buffer
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++)
    sb.Append(i);
string result = sb.ToString();
```

**How StringBuilder works**: It maintains a single `char[]` buffer (like `List<T>`). When full, it doubles capacity and copies once — the doubling strategy gives amortized O(1) per append. `ToString()` at the end is one final O(n) copy to produce the immutable string.

> In very modern .NET, `StringBuilder` can use a linked list of chunks internally for very large strings, but the growable buffer is the standard mental model.

### String Interning
- Literal strings are interned at compile time (shared in memory pool)
- `string.Intern(s)` manually adds a runtime string to the pool
- Safe because strings are immutable — shared references can't corrupt each other
- Reduces memory for repeated strings (e.g. enum-like string constants)

### Span\<char\> / ReadOnlySpan\<char\> (Modern)

`Span<T>` is a **view into existing memory** — just a pointer + length. No copy, no heap allocation:

```csharp
string s = "Hello World";
ReadOnlySpan<char> span = s.AsSpan(6, 5);  // Points to "World" inside s — no allocation
// vs:
string sub = s.Substring(6, 5);            // Allocates a new string on the heap
```

Useful for high-frequency parsing/inspection of strings without GC pressure.

**Span\<T\> is a broader concept** — works over any contiguous memory:
- `T[]` arrays
- Stack-allocated memory (`stackalloc`)
- Unmanaged/native memory pointers

Related types:
| Type | Description |
|---|---|
| `Span<T>` | Read/write view, stack-only |
| `ReadOnlySpan<T>` | Read-only view, stack-only |
| `Memory<T>` | Like `Span<T>` but can be stored on the heap (e.g. in async methods) |
| `ArraySegment<T>` | Older predecessor, less efficient |

The theme: **slice and pass views of memory without copying** — matters for high-performance parsing, serialization, and networking.

## Special Cases

**Null vs Empty**
```csharp
string.IsNullOrEmpty(s)       // "" or null
string.IsNullOrWhiteSpace(s)  // "", null, or whitespace
```

**Verbatim strings**
```csharp
string path = @"C:\Users\Files\data.txt";  // No escaping needed
```

**Raw string literals (C# 11+)**
```csharp
string json = """
{
  "name": "value"
}
""";
```

## When It Matters

- Concatenation in loops → use `StringBuilder`
- Case-insensitive comparisons → specify `StringComparison`
- High-frequency string slicing → use `Span<char>`
- Memory-constrained apps → consider interning
- Shared mutable state / threading → immutability already protects you
