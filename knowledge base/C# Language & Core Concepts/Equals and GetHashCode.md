# C# `Equals()` and `GetHashCode()` for Custom Classes

## Overview

`Equals()` and `GetHashCode()` determine how objects are compared for equality and how they behave in hash-based collections such as:

- `Dictionary<TKey, TValue>`
- `HashSet<T>`
- `ConcurrentDictionary<TKey, TValue>`

When overriding one of these methods, you should almost always override the other to keep their behavior consistent.

---

## Default Behavior for Classes

Given:

```csharp
public class Document
{
    public int Id { get; set; }
    public string Header { get; set; }
}
```

Without overriding `Equals()` or `GetHashCode()`:

```csharp
var d1 = new Document { Id = 1, Header = "A" };
var d2 = new Document { Id = 1, Header = "A" };
```

```csharp
d1.Equals(d2); // false
d1 == d2;      // false
```

For ordinary reference types:

- Equality is based on **reference identity**
- Two instances with identical property values are still considered different objects
- The default `GetHashCode()` is based on the object instance identity, not on property values

The hash code is not derived from the object's fields unless you explicitly override the method.

---

## Equality Contract

A fundamental rule:

```csharp
a.Equals(b) == true
```

must imply:

```csharp
a.GetHashCode() == b.GetHashCode()
```

The reverse is **not** required:

```csharp
a.GetHashCode() == b.GetHashCode()
```

does not imply:

```csharp
a.Equals(b)
```

Collisions are allowed.

---

## Modern Implementation

If equality should depend on `Id` and `Code`:

```csharp
public override bool Equals(object? obj)
{
    return obj is MyClass other
           && Id == other.Id
           && Code == other.Code;
}

public override int GetHashCode()
{
    return HashCode.Combine(Id, Code);
}
```

This is the recommended modern approach.

---

## Why Not Simply Use Addition?

A naive implementation:

```csharp
return Id + Code.GetHashCode();
```

works but may create many collisions.

Example:

```text
10 + 1000 = 1010
20 +  990 = 1010
30 +  980 = 1010
```

Different values can easily produce identical results.

A good hash function aims for:

- Determinism
- Good distribution
- Low collision rate

not uniqueness.

---

## Traditional Prime-Based Hashing

Historically, code often looked like:

```csharp
public override int GetHashCode()
{
    unchecked
    {
        int hash = 17;
        hash = hash * 23 + Id.GetHashCode();
        hash = hash * 23 + (Code?.GetHashCode() ?? 0);
        return hash;
    }
}
```

### Why 17 and 23?

The exact values are not important.

What matters is the pattern:

```csharp
hash = hash * PRIME + valueHash;
```

Common primes include:

```text
23
31
37
397
```

Multiplication helps spread bits and reduce collisions.

### Why `unchecked`?

Hash calculations intentionally allow integer overflow:

```csharp
hash = hash * 23;
```

The value wraps around instead of throwing an exception.

This behavior is expected and useful in hashing algorithms.

---

## What Does `HashCode.Combine()` Do?

`HashCode.Combine()`:

1. Obtains hashes of individual fields.
2. Mixes their bits using modern hash-combining algorithms.
3. Produces a well-distributed 32-bit result.

Conceptually, it behaves roughly like:

```csharp
hash = Seed;

hash = Mix(hash, field1Hash);
hash = Mix(hash, field2Hash);
hash = Mix(hash, field3Hash);

return Finalize(hash);
```

where `Mix()` performs operations such as:

- XOR
- Bit rotations
- Multiplication by carefully chosen constants
- Additions

The actual implementation is considerably more sophisticated than the old prime-based approach.

Preferred modern usage:

```csharp
return HashCode.Combine(Id, Code);
```

---

## How Many Values Can `HashCode.Combine()` Accept?

Built-in overloads exist for up to 8 values:

```csharp
HashCode.Combine(a);
HashCode.Combine(a, b);
...
HashCode.Combine(a, b, c, d, e, f, g, h);
```

For more values:

```csharp
public override int GetHashCode()
{
    var hash = new HashCode();

    hash.Add(Id);
    hash.Add(Code);
    hash.Add(Name);
    hash.Add(Address);

    return hash.ToHashCode();
}
```

---

## Which Fields Should Participate?

Only fields used by `Equals()` should participate in `GetHashCode()`.

Bad:

```csharp
Equals => Id only
GetHashCode => Id + Name + Age
```

Good:

```csharp
Equals => Id only
GetHashCode => Id
```

Equality and hashing must be based on the same logical identity.

---

## Built-in Equality of Classes vs Structs

A useful mental model:

| Type | Default Equality |
|--------|--------|
| Class | Reference equality |
| Struct | Value equality |

### Classes

By default, classes inherit `object.Equals()`:

```csharp
var a = new Document();
var b = new Document();

a.Equals(b); // false
```

because they are different objects.

### Structs

All structs derive from `ValueType`.

`ValueType.Equals()` performs a field-by-field comparison.

Example:

```csharp
public struct Point
{
    public int X;
    public int Y;
}
```

```csharp
var p1 = new Point { X = 1, Y = 2 };
var p2 = new Point { X = 1, Y = 2 };

p1.Equals(p2); // true
```

For performance reasons, custom structs often override `Equals()` and `GetHashCode()` anyway.

---

## `int.GetHashCode()`

For `Int32`, the hash code is essentially the value itself:

```csharp
5.GetHashCode();     // 5
123.GetHashCode();   // 123
(-7).GetHashCode();  // -7
```

Thus:

```csharp
return Id;
```

and

```csharp
return Id.GetHashCode();
```

are effectively equivalent.

The multiplication by primes is performed by the **combining algorithm**, not by `int.GetHashCode()`.

---

## `long.GetHashCode()`

A `long` contains 64 bits but must return a 32-bit hash.

Therefore the upper and lower 32-bit halves are mixed together.

Conceptually:

```csharp
(int)value ^ (int)(value >> 32)
```

Example:

```csharp
long value = (long)int.MaxValue + 1;
```

which equals:

```text
2147483648
```

Its hash is **not** zero.

The purpose is to ensure that both halves of the 64-bit value influence the final 32-bit hash.

A hash of zero occurs only when:

```text
upper32Bits == lower32Bits
```

because:

```csharp
x ^ x == 0
```

---

## Hashing Other Value Types

### `DateTime`

`DateTime` internally stores a 64-bit value representing ticks (plus some additional state).

Its hash is derived from that internal 64-bit representation.

Conceptually:

```csharp
hash = Hash64BitsInto32Bits(ticks);
```

similar in spirit to `long.GetHashCode()`.

### Custom Structs

Given:

```csharp
public struct Point
{
    public int X;
    public int Y;
}
```

a modern implementation would typically be:

```csharp
public override int GetHashCode()
{
    return HashCode.Combine(X, Y);
}
```

The resulting hash depends on both coordinates and produces a much better distribution than:

```csharp
X + Y
```

---

## `string.GetHashCode()`

Strings generate a hash derived from their characters:

```csharp
"ABC".GetHashCode();
```

Important:

- Equal strings always return the same hash within a process.
- The actual numeric value should not be persisted.
- Modern .NET may randomize string hashes across process executions.

---

## Why Are String Hashes Randomized?

This is primarily a security feature.

Without randomization, attackers could precompute many different strings that intentionally produce the same hash code.

If those strings are inserted into a server-side:

```csharp
Dictionary<string, TValue>
```

the dictionary's performance could degrade significantly because many entries would land in the same bucket.

To make this attack impractical, .NET includes a secret process-specific random seed when computing string hashes.

Conceptually:

```csharp
static int secret = Random.Shared.Next();

int GetHashCode(string text)
{
    int hash = secret;

    foreach (char c in text)
        hash = hash * 31 + c;

    return hash;
}
```

The real implementation is different and much more sophisticated, but the idea is similar.

Consequences:

```csharp
"Hello".GetHashCode()
```

may produce one value in Process A and a different value in Process B.

However, within a single process:

```csharp
"Hello".GetHashCode()
```

always returns the same result.

---

## Strings Are Reference Types With Value Equality

Many developers initially assume:

> Reference type ⇒ reference equality

This is only the default behavior inherited from `object`.

`string` overrides both methods.

Therefore:

```csharp
var s1 = new string("Hello".ToCharArray());
var s2 = new string("Hello".ToCharArray());

ReferenceEquals(s1, s2); // false
s1.Equals(s2);           // true
```

The objects are distinct, but the contents are identical.

Because equal objects must have equal hashes:

```csharp
s1.GetHashCode() == s2.GetHashCode();
```

is also true.

Conceptually, `string` behaves like:

```csharp
public sealed class MyString
{
    private readonly char[] _chars;

    public override bool Equals(object? obj)
    {
        return obj is MyString other
            && _chars.SequenceEqual(other._chars);
    }

    public override int GetHashCode()
    {
        return HashCode.Combine(...all characters...);
    }
}
```

The actual implementation is far more optimized but follows the same idea.

---

## String Interning

For string literals:

```csharp
var s1 = "Hello";
var s2 = "Hello";
```

.NET usually stores only one instance:

```csharp
ReferenceEquals(s1, s2); // true
```

This optimization is called **string interning**.

However, string equality does not depend on interning.

Even distinct string instances containing the same characters are equal.

---

## What Happens in `Dictionary<string, TValue>`?

Consider:

```csharp
var s1 = new string("Hello".ToCharArray());
var s2 = new string("Hello".ToCharArray());

var dict = new Dictionary<string, int>();

dict.Add(s1, 1);
dict.Add(s2, 2);
```

The second call throws:

```text
ArgumentException:
An item with the same key has already been added.
```

Why?

### Step 1

Dictionary computes the hash of the key.

```csharp
s1.GetHashCode();
s2.GetHashCode();
```

produce the same value.

### Step 2

Dictionary finds the bucket.

### Step 3

Dictionary checks actual equality:

```csharp
s1.Equals(s2);
```

which returns:

```csharp
true
```

Therefore the dictionary concludes the key already exists.

---

## Why Dictionary Uses Both Hashes and Equals

Hash codes are only a fast filtering mechanism.

Different objects may legally share the same hash:

```csharp
a.GetHashCode() == b.GetHashCode()
```

while:

```csharp
a.Equals(b) == false
```

Therefore lookup is conceptually:

```text
1. Find bucket using hash code.
2. Inspect items in that bucket.
3. Use Equals() to determine actual equality.
```

Hash codes improve performance.

`Equals()` determines correctness.

---

## Mutable Fields and Hash-Based Collections

Consider:

```csharp
var doc = new Document
{
    Id = 1,
    Header = "A"
};

var set = new HashSet<Document>();

set.Add(doc);
```

Later:

```csharp
doc.Header = "B";
```

If `Header` participates in hashing and equality, the object may no longer be found correctly inside the `HashSet`.

This can lead to surprising behavior:

```csharp
set.Contains(doc);
```

may return `false`.

---

## Important Rule About Mutability

A common misconception:

> Fields participating in equality must always be immutable.

More precise rule:

> Fields participating in equality and hashing must not change while the object is being used in a hash-based collection.

Mutable equality is technically valid, but can be dangerous if the object is used as a key.

---

## Common Design Patterns

### Entity Equality

Very common in business applications and databases.

```csharp
public class Document
{
    public int Id { get; init; }

    public string Header { get; set; }
    public List<Position> Positions { get; set; }

    public override bool Equals(object? obj)
        => obj is Document d && Id == d.Id;

    public override int GetHashCode()
        => Id;
}
```

Identity is determined solely by a stable identifier.

Other properties may freely change.

### Value Object Equality

Common for concepts such as:

- Money
- Address
- Coordinates

Example:

```csharp
public record Money(decimal Amount, string Currency);
```

All value-defining fields participate in equality.

These types are typically immutable.

---

## Why Is `GetHashCode()` an `int` Instead of a `long`?

A 64-bit hash would reduce collision probability.

However:

- Hash codes are not unique identifiers.
- Collections rely on good distribution more than hash size.
- Existing .NET collections and APIs are built around `int`.
- Changing the return type would break the framework ecosystem.

Therefore:

```csharp
public virtual int GetHashCode()
```

remains the standard.

---

## Interview Takeaways

- `GetHashCode()` is for distribution, not uniqueness.
- Equal objects must have equal hash codes.
- Unequal objects may have identical hash codes.
- Always keep `Equals()` and `GetHashCode()` consistent.
- Only hash fields that participate in equality.
- `HashCode.Combine()` is the preferred modern implementation.
- `int.GetHashCode()` typically returns the integer itself.
- `long.GetHashCode()` mixes the upper and lower 32 bits.
- Most structs use value-based equality.
- Most classes use reference equality unless they override it.
- `string` is a reference type that implements value-based equality.
- Dictionary uses hash codes to find buckets and `Equals()` to confirm equality.
- Mutable equality is possible, but dangerous for objects used in `Dictionary` or `HashSet`.
- Entity types commonly use only an immutable identifier (e.g. `Id`) for equality.
- Value objects typically use all value-defining fields and are usually immutable.