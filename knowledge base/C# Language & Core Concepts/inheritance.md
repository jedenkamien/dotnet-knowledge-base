# Inheritance (OOP) - Interview Guide for Experienced .NET Developers

Inheritance is one of the fundamental OOP pillars. It allows a derived class (child) to reuse and extend the behavior and data of a base class (parent). In C#, inheritance models an **"is-a" relationship**:

- `Dog` is an `Animal`
- `SavingsAccount` is an `Account`
- `PdfReportGenerator` is a `ReportGenerator`

The purpose is:
- code reuse,
- polymorphism,
- specialization of behavior,
- creation of extensible object hierarchies.

---

# Basic Syntax

```csharp

public class Animal
{
    public virtual void MakeSound()
    {
        Console.WriteLine("Some sound");
    }
}

public class Dog : Animal
{
    public override void MakeSound()
    {
        Console.WriteLine("Woof");
    }
}

```

Usage:

```csharp

Animal animal = new Dog();
animal.MakeSound(); // Woof

```

This is polymorphism through inheritance.

---

# What Does a Child Class Actually Get?

The child inherits:

- public members
- protected members
- virtual methods
- abstract methods (must implement)

It does NOT inherit:

- constructors
- private members
- static constructor behavior

Example:

```csharp

public class Vehicle
{
    private int _secret;
    protected int Mileage;
}

public class Car : Vehicle
{
    public void Test()
    {
        // Mileage accessible
        // _secret inaccessible
    }
}

```

Interviewers often ask:

**Why protected instead of private?**

Answer:

> Protected allows derived classes to access implementation details while still hiding them from external consumers.

---

# Constructor Order

One of the most common interview questions.

```csharp

public class Animal
{
    public Animal()
    {
        Console.WriteLine("Animal");
    }
}

public class Dog : Animal
{
    public Dog()
    {
        Console.WriteLine("Dog");
    }
}

new Dog();

```

Output:

```text
Animal
Dog
```

Rule:

> Base constructor always executes before derived constructor.

---

# Explicit Base Constructor Calls

```csharp

public class Animal
{
    public Animal(string name)
    {
    }
}

public class Dog : Animal
{
    public Dog(string name)
        : base(name)
    {
    }
}

```

Especially important when the base class has no parameterless constructor.

---

# Virtual Methods

Virtual methods can be overridden.

```csharp

public class Animal
{
    public virtual void Eat()
    {
    }
}

public class Dog : Animal
{
    public override void Eat()
    {
    }
}

```

Keywords:

- virtual
- override

Interview question:

**What happens if override is omitted?**

Compilation error.

---

# Method Hiding (new)

Many developers understand overriding but forget method hiding.

```csharp

public class Animal
{
    public void Speak()
    {
        Console.WriteLine("Animal");
    }
}

public class Dog : Animal
{
    public new void Speak()
    {
        Console.WriteLine("Dog");
    }
}

```

```csharp

Animal a = new Dog();
a.Speak();

```

Result:

```text
Animal
```

Because hiding is resolved using compile-time type.

This differs from overriding.

Very common interview topic.

---

# Overriding vs Hiding

Overriding:

```csharp

Animal a = new Dog();

```

Calls:

```text
Dog implementation
```

Method hiding:

```csharp

Animal a = new Dog();

```

Calls:

```text
Animal implementation
```

Interview answer:

> Override participates in runtime polymorphism. New only hides a member and uses compile-time binding.

---

# Calling Base Implementation

```csharp

public override void Save()
{
    base.Save();

    Logger.Log("Extra work");
}

```

Useful when extending, not replacing, behavior.

---

# Abstract Classes

Abstract classes cannot be instantiated directly.

```csharp

public abstract class Shape
{
    public abstract double Area();
}

public class Circle : Shape
{
    public override double Area()
    {
        return Math.PI * Radius * Radius;
    }

    public double Radius { get; set; }
}

```

Interview question:

**Why not use an interface?**

Answer:

Abstract class:
- can contain implementation
- can contain state
- can contain fields

Interface:
- defines contract
- supports multiple inheritance

---

# Sealed Classes

Prevent inheritance.

```csharp

public sealed class Logger
{
}

```

Often asked:

**Why seal a class?**

Possible reasons:

- protect invariants
- simplify maintenance
- avoid dangerous overrides
- allow JIT optimizations

---

# Sealed Methods

```csharp

public class Animal
{
    public virtual void Move()
    {
    }
}

public class Dog : Animal
{
    public sealed override void Move()
    {
    }
}

public class Labrador : Dog
{
    // cannot override Move
}

```

---

# Multiple Inheritance

C# does not support multiple class inheritance.

Invalid:

```csharp

class Dog : Animal, LivingThing
{
}

```

Reason:

To avoid complexity and ambiguity (diamond problem).

Instead:

```csharp

class Dog : Animal, IRunnable, IFeedable
{
}

```

Multiple interfaces are allowed.

---

# The Diamond Problem

Classic interview topic.

Imagine:

```text
     A
    / \
   B   C
    \ /
     D
```

If both B and C override a method from A:

Which implementation should D get?

Different languages solve this differently.

C# avoids the issue by disallowing multiple class inheritance.

---

# Liskov Substitution Principle (Very Important)

A derived class should be usable wherever its base class is expected.

Bad example:

```csharp

class Bird
{
    public virtual void Fly()
    {
    }
}

class Penguin : Bird
{
    public override void Fly()
    {
        throw new NotSupportedException();
    }
}

```

This violates LSP.

Interviewers love this example.

Better design:

```csharp

class Bird
{
}

interface IFlyingBird
{
    void Fly();
}

class Eagle : Bird, IFlyingBird
{
}

class Penguin : Bird
{
}

```

---

# Composition vs Inheritance

One of the most important senior-level topics.

Inheritance:

```text
Dog IS-A Animal
```

Composition:

```text
Car HAS-A Engine
```

Example:

```csharp

class Engine
{
}

class Car
{
    private readonly Engine _engine;
}

```

Modern design often prefers composition.

Reason:

Inheritance couples classes tightly.

Composition:
- more flexible,
- easier to test,
- easier to change.

Common interview answer:

> Favor composition over inheritance unless the relationship is truly "is-a".

---

# Constructor Calling Virtual Methods (Dangerous)

Huge interview favorite.

```csharp

public class Animal
{
    public Animal()
    {
        MakeSound();
    }

    public virtual void MakeSound()
    {
    }
}

public class Dog : Animal
{
    private string _name;

    public Dog()
    {
        _name = "Rex";
    }

    public override void MakeSound()
    {
        Console.WriteLine(_name.Length);
    }
}

```

Problem:

Base constructor executes first.

Derived state may not yet be initialized.

Can lead to:
- nulls,
- invalid state,
- exceptions.

Rule:

> Never call virtual members from constructors.

---

# Exceptions in Constructors

Related interview trap.

```csharp

var dog = new Dog();

```

If constructor throws:

- object is not created,
- caller receives exception,
- finalizer may still execute if partially allocated 