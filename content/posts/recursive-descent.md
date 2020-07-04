---
title: "Predictive Parsing with Recursive Descent"
date: 2020-06-28T08:38:51+03:00
draft: true
useMath: true
summary: "A simple and elegant way to implement parsers."
---

## Languages, Grammars, and Parsing

In computing we use formal languages - programming languages, query languages, markup languages, protocols, config formats, etc. Using them, we define what we want the computer to do.

Formal grammars, on the other hand, are used to define the languages themselves. For every formal language, we have a corresponding grammar (usually context-free) that defines it's structure. There are several notations used for writing grammars such as Backus-Naur Form (BNF) and Extended Backus-Naur Form (EBNF), but in this article, we'll use the syntax used in most textbooks.

Given a string and a grammar, **parsing** answers two questions:

- Does the string belong to the language of the grammar?
- What is the structure of the string with regards to the grammar?

The structure of the string relative to the grammar is denoted using a **parse tree**. If we can build a parse tree then the string is correct according to the grammar. Wel'll see some examples below.

In this article we'll learn how to define our own langages using grammars and parse their strings. We'll turn our attention to a special type of grammars called LL(1) and learn how to build corresponding parsers using the _recursive descent_ method.

## Grammar #1

Let's define the following grammar:

\\[ S \to aAe \\]
\\[ A \to bAd \mid c \\]

\\(S, A\\) are called nonterminals, \\(a,b,c,d,e\\) are the terminal symbols. \\(S\\) is the start symbol (axiom) of the grammar and above we have depicted the two production rules. This grammar describes strings that start with an "a", end with an "e", have a "c" in the middle which is surrounded by an equal number of "b"'s to the left and "d"'s to the right. Some of the strings that belong to the language of the grammar are "ace", "abcde", "abbbcddde", etc. We can examine the structure of these strings by building the corresponding parse trees.

<img src="/images/posts/rec/pt1all.png" width="300" />

Given a valid parse tree, we reconstruct the input string by traversing it depth-first and concatenating the values of the leaf nodes.

## Recursive Descent Parsing

Recursive descent parsers belong to the class of _top-down_ parsers, meaning, they construct the parse tree starting from the axiom of the grammar (the root of the tree) and make their way towards the leaves. The rules in the grammar are applied from left to right. They are fascinating due to their simplicity, elegance and can run in linear time for some types of grammars.  
This is a hands-on article, where we'll formally define some languages and build the corresponding parsers. For the examples we'll use JavaScript as a language of choice, but the concepts are universal and can be transferred to any other language.

A recursive descent parser is simply a set of functions for each nonterminal in the grammar. We begin parsing by executing the function corresponding to the axiom and terminate successfully if the whole input string has been processed. Suppose we have the following rule:

\\[ A \to X_0 X_1 \dots X_n \\]

where \\( X_i \\) is either a terminal or a nonterminal. A pseudocode for the corresponding function would look like this:

```txt
function A() {
    foreach Xi in X0...Xn
        if Xi is a nonterminal
            call the function for Xi()
        else if Xi is a terminal and Xi == current input symbol
            advance the to next input symbol
        else
            handle an errror (backtrack or throw)
}
```

## Implementing a Parser for Grammar #1

{{< highlight js "linenos=table" >}}
const parse = (() => {
    let tokens;
    let pos = 0;

    const peek = () => tokens[pos];
    const match = token => {
        if (peek() !== token)
            throw `Invalid token ${peek()}. Expected ${token}`;
        pos++;
    };
{{< / highlight >}}

We start by declaring an IIFE. The variable `tokens` is going to hold the input string, whereas `pos` indicates the current position of the input. We declare two auxiliary functions - `peek()` which will return the input token we're currently at and `match(token)` which compares a token to the one we're currently at, proceeds with one position forward if they match and throws an error otherwise.

{{< highlight js "linenos=table,linenostart=11" >}}
    function S() {
        match('a');
        A();
        match('e');
    }

    function A() {
        if (peek() === 'b') {
            match('b');
            A();
            match('d');
        } else {
            match('c');
        }
    }
{{< / highlight >}}

Now we're ready to implement the functions for the nonterminals. `S()` is trivial and directly follows the grammar rule. `A()` on the other hand has with two possible productions, so it has to make choose the correct one. To make that decision we use a lookahead by one symbol (line 18). This technique is called _predictive parsing_, but more on that later.

{{< highlight js "linenos=table,linenostart=26" >}}
    return input => {
        tokens = input;
        pos = 0;
        
        try {
            S();
            return true;
        } catch (err) {
            console.log(err);
            return false;
        }
    };
})();
{{< / highlight >}}

The parsing function takes an input string, resets the position and begins the parsing procedure by calling the function for the start symbol of the grammar. We use our simple parser as follows:

```js
parse('ace'); // true
parse('abbcdde'); // true
parse('abbcde'); // false
```

We're not quite there yet. Our `parse(input)` function is a recognizer. It can tell whether a string belongs to the language of the grammar, but it cannot infer the structure of the string. We'll deal with this in the next, less boring, example.

## LL(1) Grammars and Predictive Parsing

A nonterminal can have more than one production. If we expand one such that the string cannot be parsed, then we have to "backtrack" to the input's initial position at the time of expanding this nonterminal and try again with a different production which can be inefficient.

A predictive parser is a recursive descent parser that is able to determine in any case which production to use for a given nonterminal. The one we implemented for Grammar #1 is such. These parsers can be constructed for a class of grammars called LL(1).  The first "L" in the name stands for "scan from left to right", the second "L" means that we "expand the leftmost nonterminal" and the "1" indicates that we can look at most one token ahead at each step to make a parsing decision. LL(1) grammars are also unambiguous and don't have left recursive rules. Ambiguity means that for a given string we can have more than one valid parse tree. Left recursives rule have the form \\(A \to A\alpha \\) where \\(A\\) is a nonterminal and \\(\alpha\\) is an arbitrary sequence of terminals and nonterminals. Recursive descent parsers cannot deal with left recursion as they will end up in an infinite loop.

The LL(1) grammars are of great practical interest as they are powerful enough to cover most of the languages when it comes to computation, and their parsers run in linear time.

### FIRST and FOLLOW

Picking the right production during predictive parsing is not always as straightforward as it was for Grammar #1. That's why we define two utility functions that will make it easy for us to do so.

We define \\( FIRST(\alpha) \\), where \\( \alpha \\) is a body of a production, to be the set of terminals that the strings derived from \\( \alpha \\) begin with. If \\( \alpha \\) derives the empty string \\( \epsilon \\) then it is also in the set \\( FIRST(\alpha) \\).

Let's see an example. We have \\( A \to \alpha \mid \beta \\). The sets \\( FIRST(\alpha) \\) and \\( FIRST(\beta) \\) are disjoint. We have to choose one of the these productions so we look at the next token in in the input. If this token is in \\( FIRST(\alpha) \\) then we pick \\(A \to \alpha \\), if it is in \\( FIRST(\beta) \\) we pick \\(A \to \beta \\), otherwise the input cannot be parsed.

We define \\( FOLLOW(A) \\), where \\(A\\) is a nonterminal, to be the set of terminals that can appear immediately to the right of \\(A\\) for some derivation. For example during parsing we can end up with the following derivation \\( S \rightarrow^* \alpha A a \beta \\) for some \\(\alpha \\) and \\(\beta\\), \\(a\\) follows \\(A\\) so it is in the set. Let's see now how to compute these sets. 

\\(FIRST(X)\\) is constructed as follows:

1. If \\(X\\) is a terminal, then \\(FIRST(X) = \\{ X \\}\\).
2. If \\(X\\) is a nonterminal and \\( X \to Y_1Y_2 \dots Y_k\\) then \\( a \in FIRST(X) \\) if \\( a \in FIRST(Y_i) \\) for some \\(i = 1 \dots k \\) and \\( \epsilon \in FIRST(Y_{j}) \\) for all \\( 1 \leq j < i \\) which means that the production \\( Y_1 \dots Y_{i-1} \Rightarrow^* \epsilon \\) (derives the empty string). If the whole production \\( Y_1Y_2 \dots Y_k \Rightarrow^* \epsilon \\) then \\( \epsilon \in FIRST(X) \\).
3. If \\(X \to \epsilon \\) is a production then \\( \epsilon \in FIRST(X) \\).

Suppose we have some production body \\( X_1X_2 \dots X_n \\). We add to \\( FIRST(X_1X_2 \dots X_n) \\) all the non-\\(\epsilon\\) symbols from \\(FIRST(X_1)\\). If \\(\epsilon \in FIRST(X_1)\\) then we add the non-\\(\epsilon\\) symbols from \\(FIRST(X_2)\\) and so on. If \\(\epsilon \\) is in all \\(FIRST(X_i), i = 1 \dots n\\) then we add \\(\epsilon \\) to \\( FIRST(X_1X_2 \dots X_n) \\).

To compute \\(FOLLOW(A)\\) we apply these three rules until nothing can be added to the set:

1. Place $ in \\(FOLLOW(S)\\) where \\(S\\) is the start symbol of the grammar and $ is a special marker denoting the end of the input. $ is not a part of the grammar symbols.
2. For a production of type \\( A \to \alpha B \beta \\), everything in \\( FIRST(\beta) \\) except \\(\epsilon\\) is in \\(FOLLOW(B)\\).
3. If there is a production \\(A \to \alpha B \\) or \\(A \to \alpha B \beta \\), where \\(\epsilon \in FIRST(\beta)\\), then everything in \\(FOLLOW(A)\\) is in \\(FOLLOW(B)\\).




## Lexical Analysis

At first, our program sees a sequence of characters. The role of the lexical analyzer (tokenizer) is to group these characters into tokens and pass them to the parser. For example, the arithmetic expression "19+3=22" consists of the following tokens: ("19", Num), ("+", Op), ("3", Num), ("=", Eq), ("22", Num). Tokenizers can be either manually implemented, or generated using tools such as **lex** or **flex**. Those generators take a lexical specification in the form of regular expressions and output a tokenizer. The role of the tokenizer is to infer the tokens from a string based on this specification.

Depending on the case, the the lexing and parsing stages can be performed together so there is no need to explicitly implement a tokenizer.

## References and Further Reading

* Compilers: Principles, Techniques, and Tools (The Dragon Book), Chapter 4 Syntax Analysis
