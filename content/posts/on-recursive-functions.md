---
title: "On Recursive Functions"
date: 2019-05-03T15:23:00-07:00
draft: true
tags: ["lambda-calculus", "functional-programming", "y-combinator", "javascript"]
useMath: true
summary: "The Y-Combinator in JavaScript."
images: 
- "/images/posts/2019-05-03-on-recursion/yf.png"
description: "How to define a recursive function in a language which doesn't support recursion using the Y-combinator."
---

In this article we'll explore one of the most fascinating concepts of computer science namely the **Y-combinator** using which we can simulate recursion in a language that doesn't support it.  

We're going to use use the _Factorial_ function as an example. _Factorial_ gives us the product of an integer and all the integers below it. For example \\(4! = 4*3*2*1 = 24\\). In JavaScript, we can implement it as follows:

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

What if, however, our language **does not** support recursion. That means we're not allowed to call `fact` within itself. Actually, let's make it even more challenging - our language supports **only** 
**function definition** and **function application**. No `goto` or any kind of looping constructs whatsoever. Also our functions are allowed to take exactly one argument; no more, no less. How would we solve this problem?

## The Lambda Calculus
We're going to construct our solution with the means of lambda calculus and will implement its equivalent in JavaScript.
https://www.i-programmer.info/programming/theory/4514-lambda-calculus-for-programmers.html
Lambda calculus is a minimalistic, [turing-complete](https://en.wikipedia.org/wiki/Turing_completeness) language, powerful enough to express any kind of computation that can be performed by a modern day computer language. A detailed description of lambda calculus is outside the scope of this article. Even if you are not familiar with it, I hope that the examples should be intuitive enough to understand. For those of you who want to dive deeper can check out this ["Introduction to Lambda Calculus"](http://www.nyu.edu/projects/barker/Lambda/barendregt.94.pdf).

In &lambda;-calculus we have the three fundamental rules:

* **Function definition (Abstraction)**: 
\\[\lambda x.M\\]
Where this \\(x\\) is an argument and \\(M\\) is the body of this anonymous function.  
* **Function application**: 
\\[(M N)\\]
Apply \\(N\\) as an arugmment to \\(M\\). We refer to  \\(M\\) and  \\(N\\) as lambda terms.

For example, let's define the _Identity_ function:  
\\[I \equiv \lambda x.x \\]
It returns the argument that it has been provided. To apply the function we can simply write  
\\[ I \space N \\] 
or  
\\[ (\lambda x.x)N  \\]
In &lambda;-calculus We refer to \\(I\\) as a [_combinator_](https://en.wikipedia.org/wiki/Lambda_calculus#Free_and_bound_variables). The _Identity_ combinator has a simple reduction sequence

\\[ (\lambda x.x)N  \\]
\\[ \to (\lambda x.x)[x \mapsto N] \\]
\\[ \to N \\]

We also say that the aforementioned &lambda;-term [**&beta;-reduces to**](https://en.wikipedia.org/wiki/Reduction_strategy_(lambda_calculus)) \\(N\\) or that \\(N\\) is the **normal form** of the term. It is called **reduction** because it gets rid of an application. On the second line of the reduction using \\([x \mapsto N]\\) we denote that \\(x\\) is being substituted with \\(N\\).

## The \\(\Omega\\)-combinator
Are there &lambda;-terms without a normal form? The answer is - yes and this is a crucial part for constructing our solution. Let's take a look at the following combinator:

\\[\omega := \lambda x.xx\\]

It takes a function and applies it to itself. We can express it in JavaScript the in following way:

```js
const w = x => x(x);
```

We can now define:

\\[ \Omega := \omega \omega \equiv (\lambda x.x x)(\lambda x.x x) \\]

```js
const W = w(w); // (x => x(x))(x => x(x))
```

So if we try reducing \\(\Omega\\) we'll end up in an infinite reduction sequence because the second \\(\omega\\) substitutes the \\(x\\) in the first term so after each reduction we'll get the same term over and over again. 

\\[ (\lambda x.x x)(\lambda x.x x) \\]
\\[ \to (\lambda x.x x)[x \mapsto (\lambda x.x x)] \\]
\\[ \to (\lambda x.x x)(\lambda x.x x) \\]
\\[ \to (\lambda x.x x)[x \mapsto (\lambda x.x x)] \to ... \\]

This construction is useful because it encodes an **infinite loop**.

### The Y-Combinator
So we're looking for a combinator \\(Y\\) that given an argument some function \\(F\\) would not only reproduces itself but also apply \\(F\\) on itself. We already saw the self-reproducing term \\(\Omega\\) so using it as our basis we can define:

\\[ \omega_F := \lambda x.F(x x) \\]

The difference here is that we **apply \\(x\\) to \\(x\\) and the result we apply to some function \\(F\\)**. We can now define \\(Y\\):

\\[ Y_F := \omega_F \omega_F \equiv (\lambda x.F(x x))(\lambda x.F(x x)) \\]

or in more general terms:

\\[ Y := \lambda f. (\lambda x.f(x x))(\lambda x.f(x x)) \\]

Let's implement the Y-Combinator in JavaScript:

```js
const Y = f => {
    const g = x => f(x(x));

    return g(g);
};
```

So if we apply a function \\(F\\) to the Y-Combinator, we're going to end up with the following reduction sequence:

\\[ Y F \\] 
\\[ \equiv (\lambda f. (\lambda x.f(x x))(\lambda x.f(x x)))F \\]
\\[ \to (\lambda f. (\lambda x.f(x x))(\lambda x.f(x x)))[f \mapsto F] \\]
\\[ (\lambda x.F(x x))(\lambda x.F(x x))\\]
\\[ \to  (\lambda x.F(x x))[x \mapsto \lambda x.F(x x)]\\]
\\[ \to  F((\lambda x.F(x x))(\lambda x.F(x x)))\\]
\\[ \to  F((\lambda x.F(x x))[x \mapsto \lambda x.F(x x)])\\]
\\[ \to  F(F((\lambda x.F(x x))(\lambda x.F(x x)))) \to ... \\]

We can see where this is going.

<img src="/images/posts/2019-05-03-on-recursion/matroshka-770x472.jpg" style="width: 450px" />

## The Fixed-Point Theorem
The **fixed point** of a function \\(F\\) is some value \\(x\\) such that \\(F(x) = x\\). That is when applied to a function, it returns the same value. Let's se the following example:

\\[ f(x) = x^2 - 3x + 4 \\]
\\[ f(2) = 2^2 - 3*2 + 4 = 2 \\]

In this case \\(2\\) is a fixed point of \\(f\\). The **fixed-point theorem** states that each function has at least one such value.

**Y** is indeed a fixed-point combinator meaning that when applied to an arbitrary function \\(F\\) we get the same result as applying \\(F\\) to the result of **Y** applied to \\(F\\).

\\[ Y F = F(Y F) \\]

## Factorial in &lambda;-calculus

Let's assume that we have already defined the following functions:

* `if: Bool x Exp x Exp -> Exp` - takes a boolean value and two expressions; if the value is true, returns the first expression, otherwise the second
* `mult: Int x Int -> Int` - returns the multiplication of two numbers
* `iszero: Int -> Bool` - returns `True` if a number is equal to 0, `False` otherwise
* `pred: Int -> Int` - returns the predecessor of a number by subtracting 1

So if &lambda;-calculus supported recursion, we'd write _Factorial_ in the following way:

\\[fact := \lambda n.(if \space (iszero \space n) \space 1 \space (mult \space n \space (fact \space (pred \space n)))) \\]

This is the same as the JavaScript definition that we made above. Remember though - in &lambda;-calculus we cannot make recursive calls. So the definition above is not valid. That's where we're going to apply the Y-combinator. For this we have to define our \\(F\\) by tweaking the above definition a little bit:

\\[ factStep := \lambda f.\lambda n. \space (iszero \space n) \space 1 \space n \space (mult \space (f \space (pred \space n))) \\]

So we've wrapped the _Factorial_ definition inside a function that takes a function \\(f\\) as an arugment and calls it in one of it's branches. In our case \\(f\\) is our **pseudo-recursive** function that is going to be passed and invoked over and over again until we reach the bottom of our pseudo-recursion. We can think of \\(factStep\\) to be the body of a loop to be iterated if are are to solve the the problem with with iteration. That's why it makes sense to pass it to itself.

In JavaScript it's look the following way:

```js
const factStep = nextStep /* f */ => {
    return n => {
        if (n === 0) {
            return 1;
        }
        return n * nextStep(n - 1);
    }
}
```

Now we are ready to define the _Factorial_ function using the Y-Combinator. Let's compute _Factorial_ of 2 by going through its reduction sequence. I know that the code below seems rather tedious but by going through it one line at a time I hope it should be easy to understand. I'd also suggest writing it down, as that helped me a grasp the concept when I was learning about it. 
I've put parentheses when the precedence of the operations is not obvious. The application in &lambda;-calculus is **left associative** so that the expression: \\(M \space N \space P \equiv ((M \space N) \space P) \\) means that \\(N\\) will be applied to \\(M\\) and \\(P\\) will be applied to the result of \\(M N\\).

\\[ Y \space factStep \space 2  \\]
\\[ \to factStep \space (Y \space factStep) \space 2 \\]
\\[ \equiv (\lambda f.\lambda n. \space (iszero \space n) \space 1 \space (mult \space n \space (f \space (pred \space n)))) \space (Y \space factStep) \space 2 \\]
\\[ \to (\lambda f.\lambda n. \space (iszero \space n) \space 1 \space (mult \space  n \space (f \space (pred \space n)))) \space [f \mapsto (Y \space factStep)] \space 2 \\]
\\[ \to \lambda n. \space (iszero \space n) \space 1 \space (mult \space n \space ((Y \space factStep) \space (pred \space n))) \space 2 \\]
\\[ \to \lambda n. \space (iszero \space n) \space 1 \space (mult \space n \space ((Y \space factStep) \space (pred \space n))) \space [ n\mapsto 2 ] \\]
\\[ \to (iszero \space 2) \space 1 \space (mult \space 2 \space ((Y \space factStep)\space (pred \space 2))) \\]
\\[ \to mult \space 2 \space ((Y \space factStep)\space 1) \\]
\\[ \to mult \space 2 \space ((factStep \space (Y \space factStep)) \space 1) \\]
\\[ \equiv mult \space 2 \space (((\lambda f.\lambda n. \space (iszero \space n) \space 1 \space (mult \space n \space (f \space (pred \space n)))) \space (Y \space factStep))
\space 1) \\]
\\[ \to mult \space 2 \space (((\lambda f.\lambda n. \space (iszero \space n) \space 1 \space (mult \space n \space (f \space (pred \space n)))) \space [f \mapsto (Y \space factStep)] \space) \space 1) \\]
\\[ \to mult \space 2 \space (\lambda n. \space (iszero \space n) \space 1 \space (mult \space n \space ((Y \space factStep) \space (pred \space n)))) \space 1 \\]
\\[ \to mult \space 2 \space (\lambda n. \space (iszero \space n) \space 1 \space (mult \space n \space ((Y \space factStep) \space (pred \space n)))) [n \mapsto 1] \\]
\\[ \to mult \space 2 \space ((iszero \space 1) \space 1 \space (mult \space 1 \space ((Y \space factStep) \space (pred \space 1)))) \\]
\\[ \to mult \space 2 \space (mult \space 1 \space ((Y \space factStep) \space 0)) \\]
\\[ ... \to mult \space 2 \space (mult \space 1 1)\\]
\\[ \to mult \space 2 \space 1\\]
\\[ \to 2 \\]

Essentially, by applying \\(factStep\\) to itself, we allow it to use itself in a recursive manner. However, \\(factStep\\) is only an intermediate value - it is not the recursive function itself, as it still needs a reference to itself to do the recursion. 

## Execution Strategy

Now we can put it all together:

```js
const fact = Y(factStep);
```

But not so fast. The code above will result in:

```
RangeError: Maximum call stack size exceeded
```

This is due to the fact that JavaScript and Lambda Calculus have different [models of evaluation](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book-Z-H-27.html#%_sec_4.2.1). In fact, JavaScript uses **applicative order** (call by value) evaluation which means that if we pass an expression to a function, it'll evaluate the expression and invoke the function with it's result. This causes the Y combinator to expand infinitely (I found this out the hard way). Let's recall:

\\[ Y \space factStep \space 2  \\]
\\[ \to factStep \space (Y \space factStep) \space 2 \\]

So instad of passing \\((Y \space factStep)\\) to \\(factStep\\) as it is, JavaScript would try to evaluate it first which would lead to an infinite expansion, therefore, causing stack overflow.

\\[ factStep \space (factStep \space ... \space (factStep \space (Y \space factStep))) \space 2 \\]

So the Y combinator is suited for languages with **normal order** (call by name) evaluation. In order to adapt it for an applicative order language we have to do a slight modification:

```diff
- λf.(λx.f (x x)) (λx.f (x x))
+ λf.(λx.f (x x) y) (λx.f (x x) y) 
```

```diff
function Y1(f) {
    const g = x => {
-       return f(x(x));
+       return y => f(x(x))(y);
    };
    return g(g);
}
```

We implement _laziness_ by wrapping the function call into another function.

## Conclusion
There're other fixed-point combnators like the Z-combinator which is suited for call-by-value languages:

\\[ Z := \lambda g.(\lambda x.g \space (\lambda v.((x \space x) \space v))) \space (\lambda x.g \space (\lambda v.((x \space x) \space v))) \\]

If you want to play some more with this stuff I'd recommend implementing the Z-combinator and writing down its reduction sequence. You can also try defining recursive functions with multiple arguments for example the [Ackermann function](https://en.wikipedia.org/wiki/Ackermann_function).



## Further Reading and References
* [Barendregt, Barendsen "Introduction to Lambda Calculus"](http://www.nyu.edu/projects/barker/Lambda/barendregt.94.pdf)
* [Fixed-point combinator](https://en.wikipedia.org/wiki/Fixed-point_combinator)
* Full code reference in [JavaScript](https://github.com/deniskyashif/fmi-lcpt/blob/master/src/fixed-point.js) and [Clojure](https://github.com/deniskyashif/fmi-lcpt/blob/master/src/fixed-point.clj)
