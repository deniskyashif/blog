---
title: "Closure of Operations in Computer Programming"
date: 2025-12-14T19:18:43+02:00
summary: "Using set theory for a better software design."
useMath: true
draft: true
---

## A Bit of Math

In Algebra we say that a set is closed under a operation (or a rule) if applying that operation to elements of the set never produces something outside the set. For example, the set of integers \\( \mathbb{Z} \\) is closed under multiplication. Multipling two integers would always produce an integer.

\\[
\forall a,b \in \mathbb{Z}, a * b \in \mathbb{Z}
\\]

Integers, however, are not closed under division, since dividing two integers can produce a fractional number that does not belong to the set of integers.

\\[
\exists a,b \in \mathbb{Z} \text{ such that } a \div b \notin \mathbb{Z}
\\]

For example:

\\[
\frac{1}{2} \notin \mathbb{Z}
\\]

Applying the allowed operations, no matter how many times, never takes us outside of the set which always leaves us in a consistent state - the set is **self-contained**. We can multiply three, four, or more integers and the result will still be an integer. Mathematicians love closed sets because they provide stability, structure, and control. They make reasoning, proofs, and constructions much easier.

## Closure of Operations

Closure of operatoins is extensively used in software design as well. It is all around us even though we don't always recognize it. Using it however leads to elegant solutions. 

For example, strings are closed under contactenation. Another one - the set of all valid HTML documents is closed under DOM &rarr; DOM transformations. We know that the DOM API would always produce valid HTML. 

```js
document.body.appendChild(document.createElement("div"));
```

This is powerful because closure gives us guarantees. Those guarantees scale—from reasoning about tiny functions to designing whole systems by combining closed operations. 

### Value Objects 

This design approach is especially useful when we deal with value objects - objects defined by their value and not an identity. They are typically designed to be closed under thir core oprations. Let's see an example - we can represent money as a value object. The set of money objects is closed under the operations of addition and subtraction.

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

The operation never produces an invalid object (as long as currencies match). With this design we also have a few more useful properties:

**Equality by Value**

```cs
var a = new Money(50, "USD");
var b = new Money(50, "USD");
Console.WriteLine(a == b); // prints: True
```

**Immutability**

The `Amount` and `Currency` can't change after creation so we have thread safety bu default. Multiple parts of code can safely hold references to the same value object without risk of one part changing it for another.

**Encapsulation of Invariants**

The rules we enforce during object creation remain valid throughout the lifetime of the object. Our design does not allow for a Currency mismatch. We catch this in runtime. The Amount is always a decimal (no invalid state).


**Closure of Operations**

We can safely chain operations without worrying that we can lead us into an invalid state.

```cs
var sum = wallet1
    .Add(wallet2)
    .Add(new Money(20, "USD"));
```

### Entities

### Abstract Types

Collections, generics, LINQ

### Partial Application

## Conclusion

Look for closures in your code. This leads to a better design. When possible.

> “Where it fits, define an operation whose return type is the same as the type of its argument(s). If the implementer has state that is used in the computation, then the implementer is effectively an argument of the operation, so the argument(s) and return value should be of the same type as the implementer. Such an operation is closed under the set of instances of that type. A closed operation provides a high-level interface without introducing any dependency on other concepts.”

> “The patterns presented in this chapter illustrate a general style of design and a way of thinking about design. Making software obvious, predictable, and communicative makes abstraction and encapsulation effective. Models can be factored so that objects are simple to use and understand yet still have rich, high-level interfaces.”
