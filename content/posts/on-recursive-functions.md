---
title: "On Recursive Functions"
date: 2019-05-03T15:23:00-07:00
draft: true

tags: ["lambda-calculus", "functional-programming", "javascript"]
useMath: true
summary: "The Y-Combinator."
description: "How to define a recursive function in a language which doesn't support recursion using the Y-combinator."
---

This article assumes that the reader is familiar with the concept of [recursion](https://en.wikipedia.org/wiki/Recursion_(computer_science)) and has a basic understanding of the [Lambda calculus](https://en.wikipedia.org/wiki/Lambda_calculus) syntax.

A _Factorial_ of a number gives us the product of an integer and all the integers below it. For example \\(4! = 4*3*2*1 = 24\\). We can give the following recursive definition:

```js
const fact = n => {
    if (n === 0) {
        return 1;
    }
    return n * fact(n - 1);
}
```

So ```fact(4)``` will result in the following computation:

```
fact(4)
4 * fact(3)
4 * (3 * fact(2))
4 * (3 * (2 * fact(1)))
4 * (3 * (2 * (1 * fact(0))))
4 * (3 * (2 * (1 * 1)))
4 * (3 * (2 * 1))
4 * (3 * 2)
4 * 6
24
```

What if, however, our language **does not** support recursion. That means we're not allowed to call `fact` within itself. Actually, let's make it more challenging - our language supports **only** 
**function definition** and **function application**. No `goto` or any kind of looping constructs whatsoever. How would we solve this problem?

## The Lambda Calculus
This is a formal system for expressing computation introduced by [Alonzo Church](https://en.wikipedia.org/wiki/Alonzo_Church) in 1930's. It is a minimalistic, [turing-complete](https://en.wikipedia.org/wiki/Turing_completeness) language, powerful enough to express any kind of computation that can be expressed in a modern day computer language. A detailed description of lambda calculus is outside the scope of this article those of you who want to learn more about it can check out the [references](#further-reading-and-references) below. I decided to go with it due to it's concise syntax which would make it easier to describe and hopefully to understand the concepts of this article.

So we have the two fundamental concepts:

* **Function definition (Abstraction)**: 
\\[\lambda x.M\\]
Where this \\(x\\) is an argument and \\(M\\) is the body of this anonymous function.  
* **Function application**: 
\\[(M N)\\]
Apply \\(N\\) as an arugmment to \\(M\\). We refer to  \\(M\\) and  \\(N\\) as lambda terms.

For example, let's define the _Identity_ function:  
\\[I \equiv \lambda x.x \\]
It returns the argument it's been provided. To invoke the function we can simply write  
\\[ I N \\] 
or  
\\[ (\lambda x.x)N  \\]
which will result in \\(N\\). We also say that the aforementioned &lambda;-term [**reduces to**](https://en.wikipedia.org/wiki/Reduction_strategy_(lambda_calculus)) \\(N\\) or that \\(N\\) is the **normal form** of the term. We refer to \\(I\\) as a [_combinator_](https://en.wikipedia.org/wiki/Lambda_calculus#Free_and_bound_variables). The _Identity_ combinator has a very simple reduction sequence:

\\[ (\lambda x.x)N \to (\lambda x.x)[x \mapsto N] \to N \\]

## The \\(\Omega\\)-combinator
Are there &lambda;-terms without a normal form? The answer is - yes and this is a crucial part for constructing our solution. Let's take a look at the following combinator:

\\[\omega := \lambda x.xx\\]

It takes a function and applies it to itself. We can express it in JavaScript the following way:

```js
const w = x => x(x);
```

We can now define:

\\[ \Omega := \omega \omega \equiv (\lambda x.x x)(\lambda x.x x) \\]

```js
const W = w(w); // (x => x(x))(x => x(x))
```

So if we try reducing \\(\Omega\\) we'll end up in an infinite reduction sequence because the second \\(\omega\\) substitutes the \\(x\\) in the first term so after each reduction we'll get the same term over and over again. 

\\[ (\lambda x.x x)(\lambda x.x x) \to (\lambda x.x x)[x \mapsto (\lambda x.x x)] \to \\]
\\[ (\lambda x.x x)(\lambda x.x x) \to (\lambda x.x x)[x \mapsto (\lambda x.x x)] \to ... \\]

This construction is useful because it encodes an **infinite loop**.

## The Y-Combinator
So we're looking for a combinator \\(Y\\) that given an argument some function \\(F\\) would not only reproduces itself but also apply \\(F\\) on itself. We already saw the self-reproducing term \\(\Omega\\) so using it as our basis we can define:

\\[ \omega_F := \lambda x.F(x x) \\]

The difference here is that we **apply \\(x\\) to \\(x\\) and the result we apply to some function \\(F\\)**. We can now define \\(Y\\):

\\[ Y_F := \omega_F \omega_F \equiv (\lambda x.F(x x))(\lambda x.F(x x)) \\]

or in more general terms:

\\[ Y := \lambda f. (\lambda x.f(x x))(\lambda x.f(x x)) \\]

Let's implement the Y-Combinator in JavaScript:

```js
const Y = f => (x => f(x(x)))(x => f(x(x)));
```

We can as well write it in a more explicit way:

```js
const Y = f => {
    const fn = x => f(x(x));
    const arg = x => f(x(x));

    return fn(arg);
};
```

So if we apply a function \\(F\\) to the Y-Combinator, we're going to end up with the following reduction:

\\[ Y F \equiv (\lambda f. (\lambda x.f(x x))(\lambda x.f(x x)))F \to \\]
\\[ (\lambda f. (\lambda x.f(x x))(\lambda x.f(x x)))[f \mapsto F] \\]
\\[ (\lambda x.F(x x))(\lambda x.F(x x)) \to \\]
\\[ (\lambda x.F(x x))[x \mapsto \lambda x.F(x x)] \to \\]
\\[ F((\lambda x.F(x x))(\lambda x.F(x x))) \to \\]
\\[ F((\lambda x.F(x x))[x \mapsto \lambda x.F(x x)]) \to \\]
\\[ F(F((\lambda x.F(x x))(\lambda x.F(x x)))) \to ... \\]

## Factorial in &lambda;-calculus

Let's assume that we have already defined the following functions and they do exactly what their name suggests:

* `if: Bool x Exp x Exp -> Exp` - takes a boolean value and two expressions; if the 
value is true, returns the first expression, otherwise the second
* `mult: Int x Int -> Int` - returns the multiplication of two numbers
* `iszero: Int -> Bool` - returns `true` if a number is equal to 0, `false` otherwise
* `pred: Int -> Int` - returns the predecessor of a number by subtracting 1

So if &lambda;-calculus supported recursion, we'd write _Factorial_ in the following way:

\\[fact := \lambda n. \\]

## Further Reading and References
* [Barendregt, Barendsen "Introduction to Lambda Calculus"](http://www.nyu.edu/projects/barker/Lambda/barendregt.94.pdf)
* [Fixed-point combinator](https://en.wikipedia.org/wiki/Fixed-point_combinator)
* [Ken Shiriff's Blog, "The Y Combinator in Arc and Java"](http://www.righto.com/2009/03/y-combinator-in-arc-and-java.html)
* Full code reference in [JavaScript](https://github.com/deniskyashif/fmi-lambda-calc/blob/master/Y.js) and [Clojure](https://github.com/deniskyashif/fmi-lambda-calc/blob/master/Y.clj)
