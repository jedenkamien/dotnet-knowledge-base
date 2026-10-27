# ref, out, in — Parameters & Pass-by-Value Semantics

## Overview
C# passes arguments **by value by default**.  
The `ref`, `out`, and `in` keywords enable **pass-by-reference** for specific scenarios.

---

## Default Behavior (Pass-by-Value)

### Value Types (Structs, Primitives)
A **copy of the value** is passed.

```csharp
void Increment(int x) { x++; }

int a = 5;
Increment(a);  // a is still 5 (copy was modified)
```

---

### Reference Types (Classes)
A **copy of the reference** is passed (both point to the same object).

```csharp
void Modify(Person p) { 
    p.Age = 30;              // ✅ Modifies original object (same heap location)
}

void Reassign(Person p) { 
    p = new Person();        // ❌ Does NOT affect original reference
}                            //    (only local copy is reassigned)

Person original = new Person();
Modify(original);    // original.Age is now 30
Reassign(original);  // original still points to old object
```

**Key insight:**  
You receive a **copy of the address**. Changing the address on the copy doesn't affect the original address, but dereferencing to modify the object affects the shared object on the heap.

---

## Keyword Modifiers

### `ref` — Bidirectional Reference

- Caller **must initialize** before passing
- Callee can **read and modify**
- Changes are **visible to the caller**

#### With Value Types

```csharp
void Swap(ref int a, ref int b) {
    int temp = a; 
    a = b; 
    b = temp;
}

int x = 1, y = 2;
Swap(ref x, ref y);  // x=2, y=1 (original values modified)
```

#### With Reference Types

```csharp
void Replace(ref Person p) {
    p = new Person { Name = "New" };  // ✅ Reassigns original reference
}

Person original = new Person { Name = "Old" };
Replace(ref original);  

// original now points to NEW object with Name="New"
```

#### Difference from Default

- **Without `ref`:**
  - Can modify object contents
  - Cannot reassign the reference variable

- **With `ref`:**
  - Can modify object contents ✅
  - Can reassign the reference variable ✅

---

### `out` — Output-Only Reference

- Caller **does not need to initialize**
- Callee **must assign before returning**
- Used for **multiple return values**

```csharp
bool TryParse(string s, out int result) {
    result = 0;  // Must assign
    return int.TryParse(s, out result);
}
```

#### Modern Usage (Inline Declaration)

```csharp
if (int.TryParse("123", out var number)) {
    Console.WriteLine(number);
}
```

---

### `in` — Read-Only Reference (C# 7.2+)

- Prevents copying **large structs**
- Callee **cannot modify**
- Used for **performance optimization**

```csharp
void Process(in LargeStruct data) {
    // data is read-only reference
    Console.WriteLine(data.Value);

    // data.Value = 10;  // ❌ Compiler error
}
```

---

## Comparison Table

| Scenario              | Value Type Behavior                  | Reference Type Behavior                                      |
|----------------------|-------------------------------------|---------------------------------------------------------------|
| Default (by value)   | Copy passed, original unchanged     | Reference copied, can modify object, cannot reassign reference |
| With `ref`           | Original modified directly          | Can modify object AND reassign reference variable             |
| With `out`           | Must assign in method               | Must assign in method (can reassign reference)                |
| With `in`            | Read-only, no copy                  | Read-only reference                                           |

---

## When to Use

- **`ref` with value types:**  
  Swap operations, mutating structs without copy overhead  

- **`ref` with reference types:**  
  Rare—when you need to reassign the reference itself  

- **`out`:**  
  Multiple return values (e.g., `TryParse`, `TryGetValue`)  

- **`in`:**  
  Passing large structs without copy overhead  

- **Default:**  
  Most scenarios—simpler and safer  

---

## Real-World Use Cases for `ref` with Reference Types

While uncommon, these scenarios justify using `ref`:

### Lazy Initialization

```csharp
void EnsureInitialized(ref ExpensiveResource resource) {
    if (resource == null) {
        resource = new ExpensiveResource();
    }
}
```

---

### Double Buffering / Swap Patterns

```csharp
void SwapBuffers(ref Texture front, ref Texture back) {
    var temp = front;
    front = back;
    back = temp;
}
```

---

### Factory with Fallback

```csharp
void GetOrCreateConnection(ref DbConnection conn, string fallback) {
    if (conn == null || !conn.IsAlive) {
        conn = new DbConnection(fallback);
    }
}
```

---

### Deserializers

```csharp
bool TryDeserialize(string json, ref User user) {
    var parsed = JsonSerializer.Deserialize<User>(json);
    if (parsed != null) {
        user = parsed;
        return true;
    }
    return false;
}
```

---

## Modern Alternatives

- **Tuples for multiple returns**  
  ```csharp
  (int, bool) Parse(string s)
  ```

- **Pattern matching with `out`**  
  ```csharp
  if (int.TryParse(s, out var num))
  ```

- **Nullable reference types**  
  Instead of reassigning with null checks

---

## Best Practices

✅ Prefer immutability and return values over `ref` parameters  
✅ Use `out` for well-known patterns like `TryParse`  
✅ Use `in` for large structs (> 16 bytes) passed frequently  

❌ Avoid `ref` with reference types unless truly necessary  
❌ Don't use `ref` to return multiple values — use tuples instead  
