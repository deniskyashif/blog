---
title: "Closure of Operations in Domain Modeling"
date: 2025-12-18T15:18:43+02:00
useMath: true
draft: false
tags: ["compsci", "software-architecture", "domain-driven-design", "csharp"]
summary: "A design technique leading to a more predictable, composable, and maintainable code."
editLink: "https://github.com/deniskyashif/blog/blob/main/content/posts/2025-12-18-closure-of-operations.md"
aliases:
- /2025/12/18/closure-of-operations-in-computer-programming
- /2025/12/18/closure-of-operations-in-domain-modelling
---

In algebra, we say that a set is closed under an operation (or rule) if applying that operation to elements of the set never produces a result outside the set. For example, the set of integers \\( \mathbb{Z} \\) is closed under multiplication. Multiplying two integers would always produce an integer:

\\[
\forall a,b \in \mathbb{Z}, a * b \in \mathbb{Z}
\\]

However, the integers are not closed under division. Dividing one integer by another can produce a fractional value that does not belong to \\( \mathbb{Z} \\):

\\[
\exists a,b \in \mathbb{Z} \text{ such that } a \div b \notin \mathbb{Z}
\\]

For example:

\\[
\frac{1}{2} \notin \mathbb{Z}
\\]

When a set is closed under its allowed operations, repeatedly applying those operations—no matter how many times—never takes us outside the set. The system remains **self-contained and consistent**. We can multiply three, four, or even a hundred integers together, and the result will still be an integer.
Mathematicians value closed sets because they provide stability, structure, and control. Closure makes reasoning more predictable, proofs cleaner, and mathematical constructions far easier to work with.

## Closure of Operations

Closure of operations appears frequently in software design, even if we don’t always recognize it. When used intentionally, it tends to produce simpler and more elegant solutions.

For example, strings are closed under concatenation. Similarly, the set of all valid HTML documents is closed under DOM &rarr; DOM transformations: applying operations through the DOM API always produces another valid HTML document.

```js
document.body.appendChild(document.createElement("div"));
```

This is powerful because closure gives us guarantees. Those guarantees scale—from reasoning about tiny functions to designing whole systems by combining closed operations. 

### Value Objects 

This design approach is particularly useful when working with value objects—objects defined by their value rather than by identity. Value objects are typically designed to be closed under their core operations. For example, money can be modeled as a value object: the set of money values is closed under addition and subtraction.

```cs
public readonly record struct Money(decimal Amount, string Currency) 
{
    public Money Add(Money other) 
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add money in different currencies.");
        return new Money(Amount + other.Amount, Currency);
    }

    public Money Subtract(Money other) 
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot subtract money in different currencies.");
        return new Money(Amount - other.Amount, Currency);
    }
}
```

Usage

```cs
var wallet1 = new Money(50, "USD");
var wallet2 = new Money(30, "USD");
var total = wallet1.Add(wallet2);

Console.WriteLine(total); // prints: Money { Amount = 80, Currency = USD }
```

The operation never produces an invalid object as long as currencies match. With this design we also have a few more useful properties:

**Equality by Value**

```cs
var a = new Money(50, "USD");
var b = new Money(50, "USD");
Console.WriteLine(a == b); // prints: True
```

**Immutability**

The `Amount` and `Currency` can't change after creation so we have thread safety by default. Multiple parts of code can safely hold references to the same value object without risk of one part changing it for another.

**Encapsulation of Invariants**

The rules we enforce during object creation remain valid throughout the lifetime of the object. Our design does not allow for a Currency mismatch. We catch this in runtime. The Amount is always a decimal.


**Closure of Operations**

We can safely chain operations without worrying that we can lead us into an invalid state.

```cs
var sum = wallet1
    .Add(wallet2)
    .Add(new Money(20, "USD"));
```

### Entities

Closure of operations is natural for value objects but not typical for entities. Entities are defined by their identity and not by their value so two entities with identical data are not necessarily the same. Closure of operations on entities would mean that applying an operation to entities always produces another entity of the same type. This is rare and often undesireble. Let's take an example of a bank account.

```cs
public class Account
{
    // Constructors skipped for brevity
    public Guid Id { get; private set; }
    public Money Balance { get; private set; }

    public void Deposit(Money money)
    {
        Balance = Balance.Add(money);
    }
    
    public void Withdraw(Money money)
    {
       if (money.Amount > Balance.Amount) 
           throw new InvalidOperationException();
       Balance = Balance.Subtract(money);
    }
}
```

Typical entity operations are not closed, in the above case `Deposit`, and `Withdraw` mutate the state and do not produce a new `Account`. The `Account`, being an entity, has the following properties:

- Has an identity (`Id`).
- Has a lifecycle.
- Changes over time.
- Represents a process, not a value.
- Can introduce side effects.

We can force closure on the `Account` object but that would bring some undesireable behavior. Let's see an example.

```cs
public Account Deposit(Money money)
{
    return new Account(Id, Balance.Add(money));
}
```

```cs
var a1 = new Account(Guid.New(), new Money(100, "USD"));
var a2 = a1.Deposit(new Money(50, "USD"))
```

At this point, we can no longer tell which account instance represents the real entity. `a1` and `a2` share the same identity but have different states. When an entity is tracked by an ORM such as Entity Framework, creating additional instances breaks the change tracking and can lead to subtle, hard-to-diagnose bugs. This breaks the assumption that there is a single, authoritative instance for a given identity.

> Entities represent "things that happen over time", not things that we compute with.

The correct design is to compose entities from value objects that are closed under their operations. In this example, `Account` is the entity and `Money` is the value object. `Money` encapsulates the domain invariants and remains closed under addition, while `Account` governs the state and orchestrates its transitions.

### Closure using Abstract Types

An operation can be closed over an abstract type, even if the concrete arguments involved are value objects or entities that are not themselves closed. This is powerful because it allows a single closed set to encompass multiple concrete types.

A good example of this idea is the set of extension methods over collections in C# provided by `System.Linq`. The abstract type `IEnumerable<T>` is closed under many of its core operations, such as `Select()`, `Where()`, `OrderBy()`, `Union()`, and `Distinct()`. Because each of these operations returns another `IEnumerable<T>`, they can be freely composed, leading to elegant and expressive code:

```cs
var result = numbers
    .Where(n => n > 2)
    .Select(n => n * 10)
    .OrderBy(n => n);
```

`IEnumerable<T>`, however, is not closed over all of its operations. Such examples are `Count()`, `Any()`, `FirstOrDefault()`, etc. We can say here that we have a partial closure, however, it can still be useful and lead to a cleaner and a more maintainable design.

## Conclusion

Whenever possible, define operations whose return type matches the type of their arguments. If a method relies on the state of its object, consider that state as an implicit argument. This creates an opportunity to design the method so that it returns the same type as the object itself. Such operations are closed over the set of instances of that type, which promotes consistency and composability. This approach enhances the effectiveness of abstraction and encapsulation. By identifying closures in code, whether obvious or subtle, and making them explicit, we create designs that are more robust, elegant, and maintainable.
