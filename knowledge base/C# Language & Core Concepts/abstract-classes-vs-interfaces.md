# Abstract Classes vs Interfaces

## Core Difference

| Feature | Abstract Class | Interface |
|---------|---------------|-----------|
| Members | Fields, constructors, methods (any access) | Methods, properties, events (public only) |
| Implementation | Can provide default implementation | Can provide default (C# 8+) |
| Inheritance | Single inheritance | Multiple implementation |
| Fields | Yes | No (only properties) |
| Constructors | Yes | No |
| Access modifiers | Any | Public (implicit) |

## Abstract Class

**Definition**: Partially implemented base class that cannot be instantiated

```csharp
abstract class Animal {
    protected string name;  // Field
    
    public Animal(string name) => this.name = name;  // Constructor
    
    public abstract void MakeSound();  // Must implement
    
    public void Sleep() => Console.WriteLine($"{name} sleeps");  // Shared logic
}

class Dog : Animal {
    public Dog(string name) : base(name) { }
    public override void MakeSound() => Console.WriteLine("Woof!");
}
```

**Use when:**
- Related classes share common **state** (fields)
- Need constructors to initialize shared data
- Want to provide **partial implementation**
- "Is-a" relationship (Dog **is an** Animal)

## Interface

**Definition**: Contract specifying what a type must do (no state)

```csharp
interface IFlyable {
    void Fly();
    int MaxAltitude { get; }
}

class Bird : Animal, IFlyable {  // Multiple interfaces OK
    public void Fly() => Console.WriteLine("Flying!");
    public int MaxAltitude => 10000;
}
```

**Use when:**
- Defining **capabilities** across unrelated types
- Multiple inheritance needed
- "Can-do" relationship (Bird **can** fly)
- Dependency injection (interface-based programming)

## Real-World Examples

**Abstract Class**: `Stream`, `DbContext`, `ControllerBase`
- Shared state (buffers, connections)
- Template method pattern

**Interface**: `IEnumerable<T>`, `IDisposable`, `IComparable<T>`
- Pure contracts
- Plugin architectures

## Modern C# Features

**Default interface methods (C# 8+)**
```csharp
interface ILogger {
    void Log(string message);
    void LogError(string error) => Log($"ERROR: {error}");  // Default implementation
}
```

**Static abstract members (C# 11+)**
```csharp
interface IParseable<T> {
    static abstract T Parse(string s);
}
```

## Choosing Between Them

```
                    Need multiple inheritance?
                          /          \
                        Yes           No
                         |             |
                    Interface    Need shared fields/state?
                                      /          \
                                    Yes           No
                                     |             |
                              Abstract Class   Either works
                                            (prefer interface for flexibility)
```

## Common Patterns

**Abstract class + Interface**
```csharp
interface IRepository<T> {
    T GetById(int id);
    void Save(T entity);
}

abstract class BaseRepository<T> : IRepository<T> {
    protected readonly DbContext context;  // Shared state
    public abstract T GetById(int id);
    public virtual void Save(T entity) { /* default logic */ }
}
```

## Key Insight

- **Abstract class** = blueprint with partial construction
- **Interface** = contract with no construction
- Not mutually exclusive — often used together!
