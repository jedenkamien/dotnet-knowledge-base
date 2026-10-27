# Polymorphism in .NET (C#)

## Core Concept

Polymorphism = "many forms" — the ability to treat objects of different types uniformly through a common interface, where the actual behavior is determined by the real type of the object.

**Two kinds in C#:**
- **Compile-time (static):** method overloading, operator overloading — resolved at compile time
- **Runtime (dynamic):** `virtual`/`override`, interfaces — resolved at runtime based on actual object type

---

## Heap, Stack, and the "Lens" Mental Model

When you write `Animal a = new Dog()`, two things happen:
- A `Dog` object is created **on the heap** — it has all Dog properties (`BreedType`, `NationalRegistrationId`, `TailLength`) and Dog behaviors (`Bark()`, `Fetch()`, `WagTail()`), plus the inherited Animal ones (`Age`, `Weight`, `Sleep()`, `Eat()`)
- A **reference variable** `a` lives on the stack, typed as `Animal`

The **object never changes**. What changes is the *lens* (reference type) through which you look at it:

```
Heap:           [ Dog object — has BreedType, NationalRegistrationId, Bark(), etc. ]
                          ↑           ↑           ↑
Stack refs:      a (Animal)     d (Dog)      i (IAnimal)   ← all point to the same object
```

Through an `Animal` lens you can only call `Age`, `Eat()`, `Speak()` — the Dog-specific members are still there in memory, just inaccessible from that reference. Cast back to `Dog` and they are visible again.

**Upcast (always safe):** `Dog` → `Animal` — you narrow the view, but everything is still there.  
**Downcast (may fail):** `Animal` → `Dog` — only safe if the actual heap object really IS a `Dog`.

```csharp
Bird b = new Bird();       // heap: Bird with WingSpan, Chirp(), Fly(), BuildNest()
Animal a = b;              // ✓ upcast — safe, uses only Animal members
Bird b2 = a as Bird;       // ✓ downcast — succeeds because the heap object IS a Bird
// b2.WingSpan is accessible again

Animal a2 = new Animal();
Dog d = a2 as Dog;         // null — the heap object is a plain Animal, has no Bark(), no NationalRegistrationId
```

---

## The Virtual Method Table (vtable)

Every class has a hidden **vtable** — an array of function pointers, one slot per virtual method. It is built at compile time and baked into the type. When you call a virtual method, the runtime looks up the actual object's vtable.

```
Class Animal's vtable:
  slot[0] → Speak → Animal::Speak()

Class Dog's vtable (inherits layout, overrides slot):
  slot[0] → Speak → Dog::Speak()    ← pointer replaced by override
```

`new` (method hiding) does **not** touch the vtable — the hidden method lives outside it as a separate entry. That is why calling through a base or interface reference never reaches the hidden method; those calls go through the vtable slots.

```
Class B's vtable (based on A):
  slot[0] → F1 → B::F1()    ← override replaced the pointer
  slot[1] → F2 → A::F2()    ← new did NOT touch this slot; B::F2() lives elsewhere
```

---

## virtual / override — True Polymorphism

```csharp
class Animal
{
    public virtual void Speak() => Console.WriteLine("...");
}

class Dog : Animal
{
    public override void Speak() => Console.WriteLine("Woof");
}

Animal a = new Dog();
a.Speak();  // Woof ← runtime decides based on actual type
```

**`virtual`** = base class declares overridable behavior (provides a default implementation).  
**`override`** = derived class replaces it. Lookup happens at **runtime** → true polymorphism.

---

## abstract — Forced Polymorphism

```csharp
abstract class Shape
{
    public abstract double Area();  // No implementation — MUST override
}

class Circle : Shape
{
    public override double Area() => Math.PI * r * r;
}
```

| | `abstract` | `virtual` |
|---|---|---|
| Has body | No | Yes |
| Derived class must implement | Yes | No (can inherit as-is) |
| Use when | Every subtype needs its own logic | Base has a sensible default |
| Class must be | `abstract` | Any class |

**Rule of thumb:**
- `abstract` — when each child **must** come up with its own implementation
- `virtual` — when some children are expected to just take from their parent

---

## new — Method Hiding (Breaks Polymorphism)

When a derived class defines a method with the same signature as a **non-virtual** base method (or intentionally hides a virtual one), it *hides* rather than overrides.

```csharp
class Base
{
    public void Method() => Console.WriteLine("Base");
}

class Derived : Base
{
    public new void Method() => Console.WriteLine("Derived");  // Hides Base.Method
}

Base obj = new Derived();
obj.Method();    // Base ← compile-time type decides, NOT runtime
```

Without `new`, the compiler issues a warning. Adding `new` silences it and signals intentional hiding.

### override vs. new — side by side

```csharp
class A
{
    public virtual void F1()  { Console.WriteLine("A::F1()"); }
    public virtual void F2()  { Console.WriteLine("A::F2()"); }
}

class B : A
{
    public override void F1() { Console.WriteLine("B::F1()"); }  // polymorphic
    public new    void F2()   { Console.WriteLine("B::F2()"); }  // hidden
}

A a = new B();
a.F1();  // B::F1() ← override → runtime dispatch
a.F2();  // A::F2() ← new     → compile-time dispatch (ref type = A)

B b = (B)a;
b.F1();  // B::F1()
b.F2();  // B::F2() ← compile-time type is now B
```

**Key rule:** `new` resolution depends on the **reference type** (compile-time); `override` resolution depends on the **object type** (runtime).

### How many methods does a class with `new` actually have?

When `B` hides `A::F2()` with `new`, the `B` object effectively has **three** callable methods:
1. `F1` — overridden, replaces `A::F1()` in the vtable
2. `A::F2()` — inherited unchanged, still in the vtable slot
3. `B::F2()` — a separate, parallel method outside the vtable, only reachable when the reference type is `B`

### Interface contract and `new`

`B` does **not** re-implement interface `I`. It satisfies `I` through `A`'s implementations. `B::F2()` has nothing to do with the interface contract — `I.F2` is fulfilled by `A::F2()` even on a `B` object. That is why calling `F2()` through an `I` reference always lands on `A::F2()`.

### When to use `new`?
Rare. Useful when you intentionally want different behavior per reference type and do NOT want to participate in the base class's polymorphic contract. Considered a code smell in most scenarios.

---

## Interface Polymorphism — Implicit vs. Explicit Implementation

### Implicit (default)

```csharp
interface IFoo { void Method(); }

class MyClass : IFoo
{
    public void Method() { }  // Accessible as MyClass.Method() or IFoo.Method()
}
```

### Explicit

```csharp
class MyClass : IFoo
{
    void IFoo.Method() { }   // Only accessible via IFoo reference
}

var obj = new MyClass();
obj.Method();         // ❌ Compile error — not on public API

IFoo i = obj;
i.Method();           // ✓
```

### Both at once — resolves naming conflicts

```csharp
interface ILogger { void Write(string msg); }
interface IFile   { void Write(byte[] data); }

class MyClass : ILogger, IFile
{
    void ILogger.Write(string msg)  { }   // Each interface gets its own logic
    void IFile.Write(byte[] data)   { }
}
```

**Why explicit exists:** resolving name conflicts when implementing multiple interfaces with the same method signature, or hiding implementation details from the class's public API.

---

## Covariance & Contravariance (Generics)

### `List<Animal>` can hold mixed types — when created directly

```csharp
List<Animal> animals = new();
animals.Add(new Dog());   // ✓
animals.Add(new Cat());   // ✓
animals.Add(new Animal()); // ✓
```

Polymorphism works normally here — `List<Animal>` accepts any derived type. The issue below is different: it's about **assigning** a `List<Dog>` to a `List<Animal>` variable.

---

### `List<Dog>` is NOT assignable to `List<Animal>`

```csharp
List<Dog> dogs = new();
List<Animal> animals = dogs;  // ❌ Compile error — List<T> is invariant

animals.Add(new Cat());       // Would corrupt the dogs list — unsafe!
```

`List<T>` is **invariant** — the type parameter must match exactly, because you can both read and write.

**Why it's dangerous (memory model):**
`animals = dogs` would not create a new object. Both variables would point to the **same** `List<Dog>` on the heap — only the reference (stack) variable would differ. Adding a `Cat` via `animals` would corrupt the original `dogs` list. The compiler prevents this entirely at compile time.

---

### `IEnumerable<Dog>` IS assignable to `IEnumerable<Animal>`

```csharp
IEnumerable<Dog> dogs = new List<Dog>();
IEnumerable<Animal> animals = dogs;  // ✓ — IEnumerable<T> is covariant

// You can read:
foreach (var a in animals) { }

// But you cannot add — IEnumerable has no Add method
```

`IEnumerable<T>` is marked `out T` (covariant) — read-only, so there's no way to insert a wrong type.

---

### Variance explained

> **Important distinction:** Variance is about **generic type assignment** (can `List<Dog>` be used where `List<Animal>` is expected?), NOT about what elements a collection can hold. Adding a `Dog` to `List<Animal>` is just normal polymorphism — the `Add(Animal item)` method accepts any `Animal`, including derived types. That has nothing to do with variance.

**Invariant** — `List<Dog>` is **not** a subtype of `List<Animal>`, even though `Dog` is a subtype of `Animal`. The type argument must match exactly when assigning one generic variable to another. Required when both reading and writing (e.g. `List<T>`).

**Covariant** (`out`) — `IEnumerable<Dog>` IS a subtype of `IEnumerable<Animal>`. Can assign a more derived generic type where a base is expected. Safe because values are only *produced* (read). Example: `IEnumerable<Dog>` → `IEnumerable<Animal>`.

**Contravariant** (`in`) — can use a base generic type where a derived is expected. Safe because values are only *consumed* (written). Example: `Action<Animal>` → `Action<Dog>` (a handler accepting any `Animal` can handle a `Dog`-specific context).

| Variance | Keyword | Direction | Safe because |
|----------|---------|-----------|--------------|
| **Covariant** | `out` | `List<Derived>` → `List<Base>` allowed | Only produces values (read) |
| **Contravariant** | `in` | `List<Base>` → `List<Derived>` allowed | Only consumes values (write) |
| **Invariant** | (none) | Exact type argument required | Both reads and writes |

**Rule:** Read-only → covariant. Write-only → contravariant. Both → invariant.

---

## Classic Interview Trap (V-company-style)

```csharp
interface I { void F1(); void F2(); }

class A : I
{
    public virtual void F1() { Console.WriteLine("A::F1()"); }
    public virtual void F2() { Console.WriteLine("A::F2()"); }
}

class B : A
{
    public override void F1() { Console.WriteLine("B::F1()"); }
    public new    void F2()   { Console.WriteLine("B::F2()"); }
}

A a1 = new B();
a1.F1();  // B::F1()  — override, runtime dispatch
a1.F2();  // A::F2()  — new, compile-time dispatch (ref = A)

I x1 = a1;
x1.F1();  // B::F1()  — interface dispatch goes through A's vtable → override wins
x1.F2();  // A::F2()  — B::F2() is NOT part of A's vtable (was hidden with new)

B b1 = a1 as B;
b1.F1();  // B::F1()
b1.F2();  // B::F2()  — compile-time ref = B → hidden method visible

A a3 = new A();
I x2 = a3 as B;   // null — a3 is not a B
x2.F1();          // NullReferenceException!
```

**Insight:** `new` hides the method outside the vtable. Interface dispatch goes through the class's virtual method table (vtable), so the `new` version is invisible there — the original `A::F2()` is called even through an interface reference unless the compile-time type is `B`.

---

## Casting — `as`, `is`, and Direct Cast

### Upcast vs. Downcast

**Upcast** (`Dog` → `Animal`) is always safe — implicit, no cast needed. Every `Dog` IS-A `Animal`.  
**Downcast** (`Animal` → `Dog`) may fail — the heap object might not be a `Dog` at all.

```csharp
Dog dog = new Dog();
Animal a = dog;          // ✓ implicit upcast — always safe

Animal a2 = new Animal();
Dog d = (Dog)a2;         // ❌ InvalidCastException — no Bark(), no NationalRegistrationId on heap
```

If the downcast was allowed on a plain `Animal`, you would be accessing memory slots that don't exist — the object simply was never initialized as a `Dog`.

### `as` — Safe Cast (returns null on failure)

```csharp
Dog d = a as Dog;        // null if a's object is not a Dog — no exception
if (d != null) d.Bark(); // must check before use
```

### `is` — Type Check with Pattern Variable

```csharp
if (obj is Dog dog)
{
    dog.Bark();          // dog is already typed and non-null inside the block
}
```

### `as` vs. direct cast

| | `as` | `(T)obj` |
|---|---|---|
| On failure | returns `null` | throws `InvalidCastException` |
| Use when | unsure of runtime type | certain of type (e.g. after `is` check) |

### Are `is` / `as` code smells?

Not inherently. **Light, targeted use is fine.** The smell is a chain of type checks that should instead be handled by polymorphism:

```csharp
// ❌ Code smell — manual type dispatch
if (obj is Dog d) d.Bark();
else if (obj is Cat c) c.Meow();
else if (obj is Bird b) b.Chirp();

// ✓ Better — virtual/override does the dispatch
obj.MakeSound();  // each subclass has its own implementation
```

### Practical note

When a method receives a parameter typed as `A`, the compiler has no way to know whether the actual object is `A` or `B`. The programmer is responsible for checking:

```csharp
void Process(A a)
{
    B b = a as B;           // null if a's object isn't B — compiler won't warn
    if (b != null) { }      // your responsibility
}
```
