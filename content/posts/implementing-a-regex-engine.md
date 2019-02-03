---
title: "Implementing a Regular Expression Engine"
date: 2019-01-28T22:04:55+02:00
draft: true
tags: ["regular-expression", "parser", "compiler", "finite-state-machine", "javascript"]
useMath: true
summary: "Using the Thompson construction algorithm."
---
    
Understanding and using regular expressions properly is a valuable skill when it comes to text processing. Due to their declarative yet idiomatic syntax, they have always been a source of confusion (even [anxiety](https://stackoverflow.com/questions/172303/is-there-a-regular-expression-to-detect-a-valid-regular-expression)) amongst software developers. In this article, we'll implement a simple and efficient regular expression engine. We'll define the syntax of our regular expressions, learn how to parse them and build our recognizer. First, we'll briefly cover some theoretical foundations.

## Finite Automata
In informal terms a **finite automation** (or **finite state machine**) an abstract machine that  has states and transitions between the states. It is always in one of its states and while it reads an input it switches from state to state. It has a start state and can have one or more end (accepting) states.

### Deterministic Finite Automata (DFA)

<img id="fig1.1" src="/images/posts/2019-02-20-regex/dfa.png" />
<p class="text-center"><small>Figure 1.1: Deterministic Finite Automation (DFA)</small></p>

In [Fig 1.1](#fig1.1) we have automation with four states; _q<sub>0</sub>_ is called the **start state** and _q<sub>3</sub>_ is the **end (accepting)** state. It recognizes all and only the strings of starting with _ab_, followed by an arbitrary number of _b's_ and ending with an _a_.

A processing of the string "abba" would look like:  

| Step | &nbsp;State&nbsp; | &nbsp;Input&nbsp; | &nbsp;New State&nbsp; |
|:----:|:-----------------:|:-----------------:|:---------------------:|
| 0    | q<sub>0</sub>     | a                 | q<sub>1</sub>         |
| 1    | q<sub>1</sub>     | b                 | q<sub>2</sub>         |
| 2    | q<sub>2</sub>     | b                 | q<sub>2</sub>         |
| 3    | q<sub>2</sub>     | a                 | q<sub>3</sub>         |
  
If we process the strings **"aba"**, **"abbba"** or **"abbbbba"** through our automation, we'll end up in the accepting state of _q<sub>3</sub>_. Our machine, however, won't recognize **"ab"** as it will end up in _q<sub>2</sub>_ which is not accepting. If at any point during processing, the machine has no state to follow for a given input symbol - it stops the execution and the string is not recognized. In our example, due to the fact that given input, we can end up in an exactly one state, we say that the machine is **deterministic** (DFA).

### Nondeterministic Finite Automata (NFA)
Suppose we have the following automation:

<img id="fig1.2" src="/images/posts/2019-02-20-regex/nfa.png" />
<p class="text-center"><small>Figure 1.2: Nondeterministic Finite Automation (NFA)</small></p>

We can see on [Fig 1.2](#fig1.2) that from _q<sub>1</sub>_ on input _b_ we can transition to two states - _q<sub>1</sub>_ and _q<sub>2</sub>_. In this case, we say that the machine is **nondeterministic** (NFA). It is easy to see that this machine is equivalent to the one in [Fig 1.1](#fig1.1), that is they recognize the same set of strings. Every NFA can be converted to its corresponding DFA, the proof and the conversion, however, are a subject for another article.

### &epsilon;-NFA

We represent an epsilon-NFA exactly as we do an NFA but with one exception. It includes transitions on the **empty string** - &epsilon;. That means from one state we are able to transition into another without reading an input symbol.

<img id="fig1.3" src="/images/posts/2019-02-20-regex/enfa.png" />
<p class="text-center"><small>Figure 1.3: epsilon-NFA</small></p>

On [Fig 1.3](#fig1.3) we can see that we have **&epsilon;-transition** from q<sub>2</sub> to q<sub>1</sub>. These transitions are usually denoted with the Greek letter &epsilon; (epsilon). This &epsilon;-NFA is equivalent to the NFA in [Fig 1.2](#fig1.2).

## From Regular Expressions to Finite Automata
The **set of strings** recognized by a finite automation _A_ is called **the language of _A_** denoted as _L(A)_. If a language can be recognized by **finite automation** then there's is a corresponding **regular expression** that describes the same language and vice versa ([Kleene's Theorem](http://www.cs.may.ie/staff/jpower/Courses/Previous/parsing/node6.html)). The regular expression, equivalent to the automation in [Fig 1.1](#fig1.1)  would be **abb*a**. In other words, regular expressions can be thought of as a **user-friendly alternative** to the NFAs for describing patterns in text.

### The Thompson's Construction
Algebras of all kinds start with some elementary expressions, usually constants and/or variables. Then they allow us to construct more expressions by applying a certain set of operations to these elementary expressions. Usually, some method of grouping operators with their operands such as parentheses is required as well.  
For instance, in the arithmetic algebra we start with constants such as integers and real numbers, we include variables and using arithmetic operators, such as _&plus;_, _&times;_, we build more complex expressions. The regular expressions are in a way no different. Using constants, variables, and operators as building blocks, they denote formal languages (sets of strings).

To convert a regular expression _R_ to an NFA we first need to parse _R_ into its constituent subexpressions. The rules for constructing an NFA can be split into two parts:  

1) **Base** rules for handling subexpressions with no operators.   
2) **Inductive** rules for constructing larger NFAs from the smaller NFA's by applying the operators.


#### Basis

<img id="fig2.1" src="/images/posts/2019-02-20-regex/e-nfa-impl.png" />
<p class="text-center"><small>Figure 2.1: NFA for the expression <strong>&epsilon;</strong></small></p>

On [Fig 2.1](#fig2.1) we see an NFA that recognizes the empty string &epsilon;. _i_ is the start state and _q_ is the accepting state.

<img id="fig2.2" src="/images/posts/2019-02-20-regex/symbol-nfa.png" />
<p class="text-center"><small>Figure 2.2: NFA for the expression <em>a</em></small></p>

On [Fig 2.2](#fig2.2) we construct an NFA from the symbol _a_. We treat each symbol of the input alphabet as a regular expression by itself. The language of this automation consists only of the string **"a"**.

#### Induction
Suppose we have the two regular expressions _S_ and _T_ and their NFAs _N(S)_ and _N(T)_ respectively:

a) **Union**: _R = S_&#8739;_T_

<img id="fig3.1" src="/images/posts/2019-02-20-regex/union.png" />
<p class="text-center"><small>Figure 3.1: Union of two NFAs</small></p>

We introduce a start state _i_ and add &epsilon;-trainsitions from it to the start states of _N(S)_ and _N(T)_. Next, we add transitions from the end states of _N(S)_ and _N(T)_ to the newly created _f_ state and mark them as not accepting. The resulting NFA will recognize strings that are either belong to _L(S)_ or _L(T)_.

b) **Concatenation**: _R = ST_

<img id="fig3.2" src="/images/posts/2019-02-20-regex/concat.png" />
<p class="text-center"><small>Figure 3.2: Concatenation of two NFAs</small></p>

We mark the accepting state of _N(S)_ as not accepting and add a transition from it to the start state of _N(T)_. Here _i_ denotes the start state of _N(S)_ and _f_ denotes the accepting state of _N(T)_. This would result in an NFA that recognizes all the string concatenations _vw_ where _v_ belongs to _L(S)_ and _w_ belongs to _L(T)_.

c) **Closure (Kleenee Star)**: _R = S*_

<img id="fig3.3" src="/images/posts/2019-02-20-regex/closure.png" />
<p class="text-center"><small>Figure 3.3: NFA for the closure of a regular expression.</small></p>

We introduce _i_ as start and _f_ as accepting state. We add &epsilon;-transitions: from _i_ to _f_, from _i_ to the start state of _N(S)_, then we connect the accepting state of _N(S)_ with _f_ and finally add a transition from the end state of _N(S)_ to its start state. We mark the end state of _N(S)_ as intermediate.

The _closure (*)_ operator has the highest precedence, followed by _union (&#8739;)_. The _concatenation_ is the operation with the lowest precedence.

#### Example
Let's go through an example case. We want to construct an NFA for **(a&#8739;b)*c**. The language of this expression are all the strings that have zero or more _a_'s or _b_'s and end with _c_. Just like in the arithmetic expressions, we use brackets to specify the operator precedence. We break the expression into its atomic subexpressions and build our way up. By the order of precedence we:

1) Construct _N(a)_: a NFA for _a_.

<img id="fig4.1" src="/images/posts/2019-02-20-regex/ex-nfa-a.png" />
<p class="text-center"><small>Figure 4.1: _N(a)_: NFA for _a_.</small></p>

2) Construct _N(b)_: a NFA for _b_.

<img id="fig4.2" src="/images/posts/2019-02-20-regex/ex-nfa-b.png" />
<p class="text-center"><small>Figure 4.2: _N(b)_: NFA for _b_.</small></p>

3) Apply _union_ on _N(a)_ and _N(b)_.

<img id="fig4.3" src="/images/posts/2019-02-20-regex/ex-union-na-nb.png" />
<p class="text-center"><small>Figure 4.3: Union of _N(a)_ and _N(b)_.</small></p>

4) Apply _closure_ on _N(a&#8739;b)_

<img id="fig4.4" src="/images/posts/2019-02-20-regex/ex-closure-nab.png" />
<p class="text-center"><small>Figure 4.4: Closure of _N(a&#8739;b)_.</small></p>

5) Apply _concatenation_ of _N((a&#8739;b)*)_ with _N\(c\)_

<img id="fig4.5" src="/images/posts/2019-02-20-regex/ex-concat-nabc.png" />
<p class="text-center"><small>Figure 4.5: NFA for the expression of _(a&#8739;b)*c_.</small></p>

### Parsing a regular expression
First, we need to preprocess the string by adding an explicit concatenation operator. We're going to use the dot (.) symbol, as described in the paper so for example, the expression **abc** would be converted to **a.b.c** and **(a&#8739;b)c** wound turn into **(a&#8739;b).c**  

The modern implementations use the dot character as "any" metacharacter. The modern implementations would also probably build the NFA during parsing instead of creating a postfix expression but doing it this way would let us understand the process more clearly.

There are several ways of parsing a regular expression. We'll follow through [Thompson's original paper](https://www.fing.edu.uy/inco/cursos/intropln/material/p419-thompson.pdf) and that is by converting our expression from **infix** into **postfix** notation. This way we can easily apply the operators in the defined order of precedence.

We won't dwell into the technical details of this algorithm. You can find one in < 40 lines of javascript [here](github) and a neat explanation with more examples [here](https://en.wikipedia.org/wiki/Shunting-yard_algorithm).

### Constructing an NFA following Thompson's algorithm
We represent an **NFA state** as an object with the following properties:

```javascript
function createState(isEnd) {
    return {
        isEnd,
        transition: {},
        epsilonTransitions: []
    };
}
```

We have two types of transitions - by a symbol and by epsilon (empty string). A state in in Thompson's NFA can either have a symbol transition to at most one state or epsilon transitions to up to two states, but it cannot have a symbol and epsilon transitions at the same time.  

```javascript
function addEpsilonTransition(from, to) {
    from.epsilonTransitions.push(to);
}

function addTransition(from, to, symbol) {
    from.transition[symbol] = to;
}
```

From our **basis**, we have two types of NFA's that would serve as our building blocks - an &epsilon;-NFA and a symbol-NFA. The implementation is trivial:

```javascript
function fromEpsilon() {
    const start = createState(false);
    const end = createState(true);
    addEpsilonTransition(start, end);
    
    return { start, end };
}


function fromSymbol(symbol) {
    const start = createState(false);
    const end = createState(true);
    addTransition(start, end, symbol);

    return { start, end };
}
```

The NFA is simply an object which holds references to its start and end states. By following the **inductive rules**, we build larger NFAs by applying the three operations on smaller NFAs.

```javascript
function concat(first, second) {
    addEpsilonTransition(first.end, second.start);
    first.end.isEnd = false;

    return { start: first.start, end: second.end };
}

function union(first, second) {
    const start = createState(false);
    addEpsilonTransition(start, first.start);
    addEpsilonTransition(start, second.start);

    const end = createState(true);

    addEpsilonTransition(first.end, end);
    first.end.isEnd = false;
    addEpsilonTransition(second.end, end);
    second.end.isEnd = false;

    return { start, end };
}

function closure(nfa) {
    const start = createState(false);
    const end = createState(true);

    addEpsilonTransition(start, end);
    addEpsilonTransition(start, nfa.start);

    addEpsilonTransition(nfa.end, end);
    addEpsilonTransition(nfa.end, nfa.start);
    nfa.end.isEnd = false;

    return { start, end };
}
```

Now is time to put it all together. We **scan our postfix expression one symbol at a time** and store our context in a stack. Note that **the stack contains NFAs**. If we scan a character - we construct a character-NFA and push it to the stack. If we scan an operator, we pop from the stack, apply in on the NFA or NFAs and push the resulting NFA back to the stack.

```javascript
function toNFA(postfixExp) {
    if(postfixExp === '') {
        return fromEpsilon();
    }

    const stack = [];

    for (const token of postfixExp) {
        if(token === '*') {
            stack.push(closure(stack.pop()));
        } else if (token === '|') {
            const right = stack.pop();
            const left = stack.pop();
            stack.push(union(left, right));
        } else if (token === '.') {
            const right = stack.pop();
            const left = stack.pop();
            stack.push(concat(left, right));
        } else {
            stack.push(fromSymbol(token));
        }
    }

    return stack.pop();
}
```

#### Example
Let's simulate our algorithm on **(a&#8739;b)*c**

1) Insert explicit concatenation operator (.): **(a&#8739;b)*.c**  
2) Convert to postfix notation: **ab&#8739;*c.**  
3) Construct an NFA  

| Step | &nbsp;&nbsp;&nbsp;&nbsp;Scan&nbsp;&nbsp;&nbsp;&nbsp; | &nbsp;Operations                                             | &nbsp;Stack&nbsp;          |
|:----:|:----------------------------------------------------:|:-------------------------------------------------------------|:---------------------------|
| 0    | <u><b>a</b></u>b&#8739;*c.                           | fromSymbol(a); push;                                         | { N(a) }                   |
| 1    | a<u><b>b</b></u>&#8739;*c.                           | fromSymbol(b); push;                                         | { N(a), N(b) }             |
| 2    | ab<u><b>&#8739;</b></u>*c.                           | pop; pop; union(N(a), N(b)); push; &nbsp;&nbsp;&nbsp;        | { N(a&#8739;b) }           |
| 4    | ab&#8739;<u><b>*</b></u>c.                           | pop; closure(N(a&#8739;b)); push;                            | { N((a&#8739;b)*) }        |
| 5    | ab&#8739;*<u><b>c</b></u>.                           | fromSymbol\(c); push;                                        | { N((a&#8739;b)*), N\(c) } |
| 6    | ab&#8739;*c<u><b>.</b></u>                           | pop; pop; concat(N((a&#8739;b)*), N\(c)); push; &nbsp;&nbsp; | { N((a&#8739;b)*c) }       |

## Recognizing a string using through an NFA

### Recursive Backtracking
The simplest way to is by computing all the possible paths when processing the string through the NFA until we end up in an accepting state or exhaust all the possibilities. It falls down to a recursive depth first search with backtracking. Let's see an example:

<img id="fig5.1" src="/images/posts/2019-02-20-regex/nfa-search.png" />
<p class="text-center"><small>Figure 5.1: NFA for the expression _(aba)&#8739;(abb)_</small></p>

The automation in [Fig. 5.1](#fig5.1) recognizes either the string **"aba"** or **"abb"**. If we want to process **"abb"**, our simulation with recursive backtracking would process the the input **one state at a time** so we'll first end up reaching **q<sub>3</sub>** after reading 'a' and 'b' from the input string **"abb"**. The next symbol is 'b' but there's no transition from **q<sub>3</sub>** on 'b', therefore, we backtrack to q<sub>0</sub> and take the other path which leads us to the accepting state. 

For an NFA with N states where from each state it can transition to at most N possible states, there might be maximum \\(2^N\\) possible paths, therefore, worst case this procedure will end up going through all of these paths until it finds a match (or not). Obviously this is not an acceptable performance so we need to come up with a more clever solution.

### Being in multiple states at once
We can represent our NFA to be at multiple states at once. 

So when reading a symbol from the input string, instead of transitioning one state at a time we'll transition into all the possible states, reachable from the current set of states whith the input symbol symbol. In other words, the current state of the NFA will actually become a set of states. Each turn after reading a symbol from the input string, for each state in the current set we find the states that can be transitioned into using that symbol. So with this approach our simulation for "abb" would be:

| Step | Current                          | Input             | Next                             |
|:-----|:---------------------------------|:------------------|:---------------------------------|
| 1    | { q<sub>1</sub>, q<sub>5</sub> } | <b><u>a</u></b>bb | { q<sub>2</sub>, q<sub>6</sub> } |
| 2    | { q<sub>2</sub>, q<sub>6</sub> } | a<b><u>b</u></b>b | { q<sub>3</sub>, q<sub>7</sub> } |
| 3    | { q<sub>3</sub>, q<sub>7</sub> } | ab<b><u>b</u></b> | { q<sub>9</sub> }                |

We check if any state in the set of states that we end up with is an accepting state and if so - the string is recognized by the NFA. In the above case the final set contains only q<sub>9</sub> which is an accepting state.

#### Dealing with &epsilon;-transitions
As mentioned above, Thompson's construction has two types of states. Ones with &epsilon;-transitions and ones with a transition on a symbol. So each time we end up in a state with an &epsilon;-transition(s) we simply follow through to the next state(s) until we end up in one that has a transition on a symbol and insert it to the set of next states. This is a recursive procedure.

```javascript
function addNextStates(state, nextStates, visited) {
    if (state.epsilonTransitions.length) {
        for (const st of state.epsilonTransitions) {
            if (!visited.find(vs => vs === st)) {
                visited.push(st);
                addNextStates(st, nextStates, visited);
            }
        }
    } else {
        nextStates.push(state);
    }
}
```

We also have to mark the &epsilon;-transition states as visited to prevent infinte looping. The bottom of the recurison is when we reach a state with no &epsilon-transitions. After putting it all together, this is the code or the search procedure.

```javascript
function search(nfa, word) {
    let currentStates = [];
    addNextStates(nfa.start, currentStates, []);

    for (const symbol of word) {
        const nextStates = [];

        for (const state of currentStates) {
            const nextState = state.transition[symbol];
            if (nextState) {
                addNextStates(nextState, nextStates, []);
            }
        }
        currentStates = nextStates;
    }

    return currentStates.find(s => s.isEnd) ? true : false;
}
```

At the start, the initial set of current states is either the start state itself or the set of states, reachable by epsilon transitions from it. In the example on [Fig 6.1](#fig6.1)

## Recap
We started by learning about finite automata and how do they work. We classified them as deterministic and nondeterministic. We also introduced the concept of &epsilon;-NFAs which are a specific type of nondeterministic automata. We've seen how NFAs and DFAs have the same expressive power and how they can be used for string recognition. 

We defined the building blocks of the regular expressions and saw how by applying operators (concatenation, union, and closure) we can construct more complex expressions from smaller ones. Then we learned how to convert a regular expression into its equivalent NFA then implemented an algorithm that processes a string through an NFA to determine whether it is matched or not.

The complete code reference for this article is available on [GitHub](https://github.com/deniskyashif/regexjs/).

## References
* Hopcroft, Motwani, Ullman (2001) _Introduction to Automata Theory, Languages, and Computation_ - Chapter 3: Regular Expressions and Languages
* Aho, Lam, Sethi, Ullmann (2007) _Compilers: Principles, Techniques, and Tools_ (The Dragon Book) - Chapter 3: Lexical Analysis
* Ken Thompson (1968) _Regular Expression Search Algorithm_
* [_“Regular Expression Matching Can Be Simple And Fast”_](https://swtch.com/~rsc/regexp/regexp1.html) by Russ Cox
* [Thompson's Construction Algorithm](https://en.wikipedia.org/wiki/Thompson%27s_construction) on Wikipedia
* [Finite State Machine Designer](http://madebyevan.com/fsm/) by Evan Wallace
