# Boxing & Unboxing

## Definition
- **Boxing**: Converting value type (int, bool, struct) → `object` (heap allocation)
- **Unboxing**: Converting `object` → original value type (back to stack)

## Example
```csharp
int x = 42;
object boxed = x;    // Boxing
int y = (int)boxed;  // Unboxing
```

## Modern Relevance

**Less critical** — Generics (`List<int>`) eliminated most boxing scenarios

**Still matters when:**
- Non-generic collections (`ArrayList`, `Hashtable`)
- Methods accepting `object` parameter
- String interpolation (`$"Value: {x}"`)
- Reflection/serialization
- Performance-critical code

**Takeaway**: Avoid unnecessary boxing in high-performance applications, but rarely an issue in typical modern C# code.