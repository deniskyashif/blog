---
title: "Gödel's System T in TypeScript"
date: 2019-06-03T21:31:07+03:00
draft: false
tags: ["lambda-calculus", "functional-programming", "type-systems", "typescript", "javascript"]
summary: "Experimenting with a more restrictive type system which ensures that the programs always terminate."
description: "Experimenting with a more restrictive type system which ensures that the programs always terminate."
images: 
- "/"
useMath: true
---

Recently, I've been reading Bove and Dybjer’s paper on ["Dependent Types at Work"](http://www.cse.chalmers.se/~peterd/papers/DependentTypesAtWork.pdf) where Godel's System T is briefly described. It is a type system based on the simply typed lambda calculus and includes **booleans** and **natural numbers**. The unusual thing about it is that it allows us to perform **only** [primitive recursion](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=2ahUKEwiBqeT41M_iAhUK4aQKHeBYDowQFjACegQICxAG&url=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FPrimitive_recursive_function&usg=AOvVaw293_OtMzAukv1lsqcq0V4H), which considerably limits the amount of possible programs we can write but on the other hand guarantees that these programs **always terminate**. This means that in **System T** we can express only the set of **total computable functions**.

### Why do we only have primitive recursion?

In untyped lambda calculus we can define fixed point combinators which allow us to simulate recursion. So if System T is based lambda calculus, how come we can't have non-primitive recursion? The reason is the type system. Let's recall from the [article "On Recursive Functions"]("/on-recursive-functions") - and the \\(\omega\\) combinator:

\\[\omega := \lambda x.xx\\]

What would the type of this term be? Let's assume the second \\(x\\) in \\(xx\\) be of type \\(\alpha\\). That means the first \\(x\\) should also be of type \\(\alpha\\). Here we arrive at a contradiction because the first \\(x\\) is a function so it should have a type \\(\alpha \to \beta\\) for some \\(\beta\\). Both terms should have the same type, hence the contradiction. Every fixed point combinator involves some kind of self application, therefore, it cannot be expressed in this  type system thus making our language less powerful but on the other hand - more predictable.

## Building Blocks

System T includes predefined constants for `true`, `false` and `zero`, as well as the `Succ`, `Cases` and `Rec` combinators, which represent the successor function, if-then-else, and primitive recursion respectively. Having this in our arsenal, we can now build some abstractions.  
In the paper, the authors give definitions of the primitives and some operators in Agda and leave some additional tasks to the reader. So I tried to implement this system in TypeScript, which turned out to be a fun exercise.

## Primitives

* `Bool`: this is quite trivial as we can use TypeScript's `boolean` type.

* `Nat`: natural numbers. This is a tricky as the definition for `Nat` in System T is: `Nat: zero | Succ`. So it can be either zero or the successor function. To give an intuition - `Zero == 0`, `Succ(Zero) == 1`, `Succ(Succ(Zero)) == 2` etc. For the sake of simplicity I decided to use TypeScript's `number` type but only for the representating the numbers. We're not allowed to use their built-in properties, like arithmetic operations, because we're going to construct them from the ground up using the predefined primitives.

* `Succ: Nat -> Nat` we define as:

```ts
const Succ = (x: number): number => x + 1;
```

* `Cases<T>: Bool -> T -> T -> T` - think of it as a generic conditional (if-then-else). The polymorphism in System T can be achieved using TypeScript's generics.

```ts
function Cases<T>(cond: boolean, a: T, b: T): T {
    return cond ? a : b;
}
```

It's easy to see that `Cases<number>(true, 1, 2) == 1` and `Cases<number>(false, 1, 2) == 2`.

* `Rec<T>: Nat -> T -> (Nat -> T -> T) -> T` - this is called the **Godel's Recursor**.

```ts
function Rec<T>(sn: number, s: T, t: (z: number, w: T) => T): T {
    return IsZero(sn) ? s : t(sn - 1, Rec(sn - 1, s, t));
}
```

It might seem confusing at first but its reduction is straightforward:

```
Rec 0 s t -> s
Rec sn s t -> t n (R n s t)
```

`Rec` is a polymorphic higher-order function that takes three arguments (or 4 if we count the type). `sn` is the natural number on which we perform the recursion, think of `sn` as _the successor of n_. `s` is the element returned from the base, whereas `t` is the function called on each recursive step. 

## Arithmetic Operators

This is where it gets interesting. We construct **addition** the following way:

```ts
function add(x: number, y: number): number {
    return Rec<number>(x, y, (z, w) => Succ(w));
}
```

Seems weird? Let's see what happens when we invoke `add(2, 2)`:

```
t := λz.λw.Succ w

add 2 2
Rec 2 2 t
t 1 (Rec 1 2 t)
t 1 (t 0 (Rec 0 2 t))
t 1 (t 0 2)
t 1 3
4
```

Using it as a building block we can define **multiplication**:

```ts
function multiply(x: number, y: number): number {
    return Rec<number>(y, 0, (z, w) => add(x, w));
}
```

As well as **exponentiation**:

```ts
function exp(x: number, y: number): number {
    return Rec<number>(y, 1, (z, w) => multiply(x, w));
}
```

We can also define the **predecessor** function:

```ts
function pred(x: number): number {
    return Rec<number>(x, 0, (z, w) => z);
}
```

This is a bit different than what we've been defining so far and it might not be so obvious why it works. `t` is a function that takes two arguments and returns the first one. Let's walk through the reduction sequence of `pred(3)`:

```
t := λz.λw.z

pred 3
Rec 3 0 t
t 2 (Rec 2 0 t)
t 2 (t 1 (Rec 1 2 t))
t 2 (t 1 (t 0 (Rec 0 2 t)))
t 2 (t 1 (t 0 2))
t 2 (t 1 0)
t 2 1
2
```

**subtraction** is very similar to **addition**, we just have to replace `Succ` with `pred`:

```ts
function subtract(x: number, y: number): number {
    return Rec<number>(y, x, (z, w) => pred(w));
}
```

## Boolean Operators

Use the `Cases<T>` combinator with `true` and `false` constants to construct logical operators:

```ts
function not(x: boolean): boolean {
    return Cases<boolean>(x, false, true);
}

function and(x: boolean, y: boolean): boolean {
    return Cases<boolean>(x, y, false);
}

function or(x: boolean, y: boolean): boolean {
    return Cases<boolean>(x, true, y);
}
```

Now it's time to compare numbers. For that we're going to reuse some of the functions we implemented so far and give definition to `isZero: Nat -> Bool`.
```ts
const isZero = (x: number): boolean => {
    return Rec<boolean>(x, true, (z, w) => false);
}

function eq(x: number, y: number): boolean {
    return and(IsZero(subtract(x, y)), IsZero(subtract(y, x)));
}

function gt(x: number, y: number): boolean {
    return not(IsZero(subtract(x, y)));
}

function lt(x: number, y: number): boolean {
    return not(IsZero(subtract(y, x)));
}

function gte(x: number, y: number): boolean {
    return or(eq(x, y), gt(x, y));
}

function lte(x: number, y: number): boolean {
    return or(eq(x, y), lt(x, y));
}
```

## Beyond Primitive Recursion
Now to our last and most interesting example. In System T we can express the **total computable functions** using primitive recursion, but can we express the ones that are not primitive recursive?


### Ackermann

The [Ackermann function](http://mathworld.wolfram.com/AckermannFunction.html) is one of the earliest discovered examples of a total computable function that is **not primitive recursive**. All primitive recursive functions are total and computable, but the Ackermann function illustrates that not all total computable functions are primitive recursive.

<img src="/images/posts/2019-06-05-godel-t/ackermann.png" alt="Ackermann Definition" width="450" style="margin-left: 0" />

[Here](https://www.wolframalpha.com/input/?i=ackerman(2,3)) you can find some neat visual example of how it runs.

Let's try to implement it within our type system! We'll start by definig an operator for function composition:

```ts
type OneArityFn<T, K> = (x: T) => K;

function compose<T, K, V>(f: OneArityFn<K, V>, g: OneArityFn<T, K>)
    : OneArityFn<T, V> {
    return x => f(g(x));
}
```

Observe that `compose` is a higher order function. Now using the Godel's recursor, let's an iterator function:

```ts
function iterate<T>(f: OneArityFn<T, T>, n: number)
    : OneArityFn<T, T> {
    return Rec<OneArityFn<T, T>>(
        n,
        x => x,
        (z, acc) => compose(f, acc));
}
```

Simply put given a function `f` and a number `n`, `iterate` will invoke `f` on it's output `n` number of times. Think of it as composing it with itself. For example `iterate(f, 3)` will result in `x => f(f(f(x)))`. And now we're ready to define `ackermann`.

```ts
function ackermann(x: number): OneArityFn<number, number> {
    return Rec<OneArityFn<number, number>>(
        x,
        Succ,
        (z, acc) => y => iterate(acc, y)(acc(Succ(Zero))));
}

ackermann(1)(1); // => 3
```

## References

* ["Code Reference"](https://github.com/deniskyashif/fmi-lcpt/blob/master/src/godel.ts)
* ["Dependent Types at Work"](http://www.cse.chalmers.se/~peterd/papers/DependentTypesAtWork.pdf)
* ["Godel's System T in Agda"](http://gregorulm.com/godels-system-t-in-agda/)
* ["Godel's System T - Foundations of Programming Languages"](https://www.andrew.cmu.edu/course/15-312/recitations/rec3-notes.pdf)
