# Record Keyword

## Overview
`record` (C# 9+) is a reference type optimized for **immutable data modeling** with value-based equality.

## Basic Syntax

**Positional record (concise)**
```csharp
record Person(string FirstName, string LastName, int Age);

// Equivalent to:
// - Auto-implemented properties
// - Constructor
// - Deconstruction
// - ToString(), Equals(), GetHashCode()
```

**Traditional syntax**
```csharp
record Person {
    public string FirstName { get; init; }
    public string LastName { get; init; }
    public int Age { get; init; }
}
```

## Key Features

### 1. Value-Based Equality
```csharp
var p1 = new Person("John", "Doe", 30);
var p2 = new Person("John", "Doe", 30);
Console.WriteLine(p1 == p2);  // True (compares values, not references)
```

### 2. Immutability with `init`
```csharp
var person = new Person("Jane", "Smith", 25);
// person.Age = 26;  // Compile error
```

### 3. Non-Destructive Mutation (`with`)
```csharp
var original = new Person("John", "Doe", 30);
var modified = original with { Age = 31 };  // Creates new instance
// original.Age is still 30
```

### 4. Built-in ToString()
```csharp
var person = new Person("John", "Doe", 30);
Console.WriteLine(person);  // Person { FirstName = John, LastName = Doe, Age = 30 }
```

### 5. Deconstruction
```csharp
var (firstName, lastName, age) = person;
```

## Record vs Class

| Feature | Record | Class |
|---------|--------|-------|
| Equality | By value | By reference (default) |
| Default mutability | Immutable (`init`) | Mutable |
| Inheritance | Supported | Supported |
| ToString() | Auto-generated | Manual override |
| Primary use | Data transfer objects (DTOs) | Complex behavior |

## Record Types

**Record class (default)**
```csharp
record Person(string Name);  // Reference type, heap-allocated
```

**Record struct (C# 10+)**
```csharp
record struct Point(int X, int Y);  // Value type, stack-allocated
```

## When to Use

✅ **Use record when:**
- DTOs, API models, configuration
- Immutable data structures
- Value semantics matter (equality by content)
- Need frequent cloning with modifications

❌ **Avoid record when:**
- Mutable state required
- Entity with identity (e.g., database entities)
- Complex behavior (use class)

## Real-World Examples

**API Response**
```csharp
record UserDto(int Id, string Username, string Email);
```

**Configuration**
```csharp
record AppSettings {
    public string ConnectionString { get; init; }
    public int Timeout { get; init; }
}
```

**Event sourcing**
```csharp
record OrderPlaced(Guid OrderId, DateTime Timestamp, decimal Amount);
```

## Advanced Features

**Inheritance**
```csharp
record Person(string Name);
record Employee(string Name, string Department) : Person(Name);
```

**Custom equality**
```csharp
record Person(string Name) {
    public virtual bool Equals(Person? other) {
        // Custom comparison logic
    }
}
```

## Summary

`record` = **data-focused** immutable reference type with value equality. Use for DTOs, messages, and simple data containers. For behavior-rich types, stick with classes.
