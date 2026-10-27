# Encapsulation (.NET Interview Guide)

## Definition

Encapsulation is one of the four pillars of Object-Oriented Programming (OOP).

The core idea is:

> Hide implementation details and protect invariants by exposing a controlled public API.

Consumers of a class should know **what** it does, not **how** it does it.

---

# Simple Example

Bad:

```csharp
public class BankAccount
{
    public decimal Balance;
}
```

Anyone can do:

```csharp
account.Balance = -1000000;
```

The object cannot prevent invalid state.

Better:

```csharp
public class BankAccount
{
    public decimal Balance { get; private set; }

    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException();

        Balance += amount;
    }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException();

        if (amount > Balance)
            throw new InvalidOperationException();

        Balance -= amount;
    }
}
```

Now the object controls how its state changes.

---

# Strong Interview Definition

A senior-level answer could be:

> Encapsulation is the practice of hiding implementation details and protecting an object's internal state behind a well-defined public interface. The object itself is responsible for enforcing business rules and maintaining its invariants.

Keywords interviewers like:

- Information hiding
- Controlled access
- Separation of concerns
- Invariant protection
- Reduced coupling
- Maintainability

---

# Invariants

A very important interview concept.

An invariant is a condition that must always remain true.

Example:

For a bank account:

- Balance cannot be negative.
- Withdrawal cannot exceed current balance.

Without encapsulation:

```csharp
account.Balance = -500;
```

Invariant broken.

With encapsulation:

```csharp
account.Withdraw(500);
```

The object can validate the operation before changing state.

---

# Access Modifiers

Encapsulation is commonly implemented through access modifiers.

```csharp
private
protected
internal
public
private protected
protected internal
```

Example:

```csharp
public class User
{
    private string _passwordHash;

    public bool VerifyPassword(string password)
    {
        ...
    }
}
```

External consumers cannot access:

```csharp
user._passwordHash
```

which protects the internal implementation.

---

# Encapsulation Is Not Just Private Fields

A common interview trap.

Many developers think:

```csharp
private string _name;
public string Name { get; set; }
```

automatically implies good encapsulation.

Not really.

This still allows arbitrary state changes.

Example:

```csharp
customer.Name = "";
```

Better:

```csharp
public string Name { get; private set; }

public void ChangeName(string name)
{
    if (string.IsNullOrWhiteSpace(name))
        throw new ArgumentException();

    Name = name;
}
```

Here the object controls the transition.

---

# Behavior Over State

Senior engineers often expose behavior instead of setters.

Less preferred:

```csharp
order.Status = OrderStatus.Shipped;
```

Preferred:

```csharp
order.MarkAsShipped();
```

Implementation:

```csharp
public void MarkAsShipped()
{
    if (!PaymentReceived)
        throw new InvalidOperationException();

    Status = OrderStatus.Shipped;
}
```

This guarantees business rules are respected.

---

# Encapsulation Through Private Setters

Extremely common in modern .NET.

```csharp
public class Customer
{
    public string Email { get; private set; }

    public void ChangeEmail(string email)
    {
        Validate(email);
        Email = email;
    }
}
```

Benefits:

- Consumers can read state.
- Consumers cannot arbitrarily modify state.
- Validation is centralized.

---

# Mutable Collections: Important Interview Trap

Consider:

```csharp
public List<OrderItem> Items { get; } = new();
```

Looks encapsulated?

Not really.

Consumers can still do:

```csharp
order.Items.Clear();
order.Items.Add(newItem);
```

A better design:

```csharp
private readonly List<OrderItem> _items = new();

public IReadOnlyCollection<OrderItem> Items => _items;

public void AddItem(OrderItem item)
{
    _items.Add(item);
}
```

This is one of the most frequently discussed real-world encapsulation examples.

---

# Arrays: Another Common Trap

Bad:

```csharp
private byte[] _data;

public byte[] Data => _data;
```

Consumer:

```csharp
obj.Data[0] = 255;
```

Internal state was modified externally.

Better:

```csharp
public byte[] Data => _data.ToArray();
```

or

```csharp
public ReadOnlyMemory<byte> Data => _data;
```

---

# Encapsulation and Immutability

Immutable objects provide strong encapsulation.

Example:

```csharp
public record Money(decimal Amount, string Currency);
```

Usage:

```csharp
var money = new Money(100, "USD");
```

State cannot be accidentally modified after construction.

Advantages:

- Thread safety
- Easier debugging
- Less defensive code
- Fewer invalid states

---

# Encapsulation vs Abstraction

One of the most popular interview questions.

## Encapsulation

Protecting internal state and implementation details.

Example:

```csharp
account.Deposit(100);
```

You don't know how balance is stored.

## Abstraction

Hiding unnecessary complexity.

Example:

```csharp
ILogger logger;
```

You don't care whether logging goes to:

- Console
- File
- Database
- Application Insights
- OpenTelemetry

### Short Answer

> Encapsulation protects state. Abstraction hides complexity.

---

# Encapsulation vs Information Hiding

Sometimes interviewers distinguish them.

Information hiding:

- A design principle.
- Hide internal implementation details.

Encapsulation:

- The mechanism used to achieve it.
- Implemented via classes, access modifiers, properties, methods, etc.

Most developers use the terms interchangeably.

Mentioning the distinction can earn bonus points.

---

# Encapsulation in Domain-Driven Design (DDD)

Junior-style model:

```csharp
public class Order
{
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
}
```

Rich domain model:

```csharp
public class Order
{
    public decimal Total { get; private set; }
    public OrderStatus Status { get; private set; }

    public void Cancel()
    {
        if (Status == OrderStatus.Shipped)
            throw new InvalidOperationException();

        Status = OrderStatus.Cancelled;
    }
}
```

The entity itself owns and enforces the rules.

---

# Encapsulation and Thread Safety

These are related but different concepts.

This class is encapsulated:

```csharp
public class Counter
{
    private int _value;

    public void Increment()
    {
        _value++;
    }
}
```

But not thread-safe.

Multiple threads can race.

Safer implementation:

```csharp
public void Increment()
{
    Interlocked.Increment(ref _value);
}
```

Interview point:

> Encapsulation does not automatically imply thread safety.

---

# Reflection Can Break Encapsulation

Interesting senior-level topic.

Even private members can be accessed via reflection:

```csharp
typeof(User)
    .GetField("_secret", BindingFlags.NonPublic | BindingFlags.Instance);
```

Therefore:

> Encapsulation is a design principle, not a security boundary.

Many senior interviewers appreciate this observation.

---

# Serialization Can Break Encapsulation

JSON serializers can populate members directly.

Example:

```csharp
public class Customer
{
    public string Email { get; private set; }
}
```

Serializer frameworks may bypass some of the intended construction paths.

Therefore important validation should often live inside:

- Constructors
- Factory methods
- Domain methods

rather than only in controllers or API endpoints.

---

# Entity Framework Edge Cases

ORMs may require special accommodations.

Example:

```csharp
private Order()
{
}
```

or

```csharp
public string Name { get; private set; }
```

Entity Framework can still materialize such entities.

Real-world systems occasionally relax strict encapsulation to satisfy infrastructure requirements.

---

# Encapsulation with init

Modern C# introduced init-only setters.

```csharp
public class User
{
    public string Name { get; init; }
}
```

Usage:

```csharp
var user = new User
{
    Name = "John"
};
```

After construction:

```csharp
user.Name = "Bob"; // compile error
```

This is often a good compromise between convenience and encapsulation.

---

# Interview Questions You May Get

## What problem does encapsulation solve?

Protecting object state and preventing invalid state transitions.

## Is a public property with get/set properly encapsulated?

Not necessarily. It exposes unrestricted modification.

## Why return IReadOnlyCollection instead of List?

To prevent external code from modifying internal state.

## Why is exposing arrays dangerous?

Arrays are mutable reference types.

## Does encapsulation improve maintainability?

Yes. Internal implementation can change without affecting consumers.

## Does encapsulation guarantee thread safety?

No.

## Can reflection violate encapsulation?

Yes.

## What is the relationship between encapsulation and immutability?

Immutability can be viewed as a very strong form of encapsulation because state cannot change after creation.

---

# Common Mistakes

## Mistake 1

```csharp
public List<Order> Orders { get; }
```

Exposes mutable state.

---

## Mistake 2

```csharp
public string Name { get; set; }
```

Allows arbitrary changes without validation.

---

## Mistake 3

Putting all validation in service layers.

Bad:

```csharp
if (amount > account.Balance)
{
    ...
}
```

Better:

```csharp
account.Withdraw(amount);
```

The object should protect itself.

---

# Senior-Level Summary

If asked in an interview:

> Encapsulation is the practice of protecting an object's internal state and enforcing invariants through a controlled public API. Good encapsulation is not simply making fields private; it is about controlling state transitions, avoiding exposure of mutable internals, exposing behavior instead of setters where appropriate, and keeping an object valid throughout its lifetime. In modern .NET, encapsulation is often combined with immutability, readonly collections, domain-driven design patterns, and carefully designed public interfaces.