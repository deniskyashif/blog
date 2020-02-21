---
title: "Resolving Ambiguity in Text Rewriting"
date: 2020-02-22T09:10:37+02:00
draft: true
useMath: true
summary: "Strategies for resolving conflicts that occur in text rewriting."
tags: ["compsci", "nlp"]
editLink: "https://github.com/deniskyashif/blog/blob/master/content/posts/2020-02-22-ambiguity-in-text-rewriting.md"
images: 
- "/images/posts/2020-02-22-ambiguity-text-rewriting/overlap.png"
---

In text rewriting we replace parts of a text matching a rewrite rule with other text. In some cases, however, these parts might overlap so we have to come up with a way to decide which one is to be replaced. In this article, we'll discuss several conflict resolution strategies and describe their implementations from a formal standpoint.  
This article builds on top of [Finite-State Transducers for Text-Rewriting](/2020/02/18/finite-state-transducers-for-text-rewriting/) and I recommend checking it out first, before proceeding with this one.

## Ambiguity

Consider the following rewrite rule and suppose we have build a finite-state transducer for obligatory replacement based on it.

\\[ 
R_1 = ab|bc \to x
\\]

and the following input text.

\\[ aabcb \\]

If we don't specify a conflict resolution strategy, the obligatory rewrite transducer for \\(R_1\\) will result in two distinct translations because the matches "ab" and "bc" clearly overlap.

\\[ 
a \cdot ab \cdot cb \longmapsto_{R(T_{R_1})} axcb \\\\
aa \cdot bc \cdot b \longmapsto_{R(T_{R_1})} aaxb
\\]

Let's see another example. A rewrite rule

\\[ 
R_2 = aa*b|aa \to x
\\]

and an input text

\\[ aaaaabbaa \\]

that results in even more potential decompositions.

\\[ 
aaaaab \cdot b \cdot aa \longmapsto_{R(T_{R_2})} xbx \\\\
aa \cdot aaab \cdot b \cdot aa \longmapsto_{R(T_{R_2})} xxbx \\\\
aa \cdot aa \cdot ab \cdot b \cdot aa \longmapsto_{R(T_{R_2})} xxxbx
\\]

A proper conflict resolution strategy would ensure that for each input text, we'll get a single output in accordance to this strategy.

## Definitions

Let's provide some formal definitions that will make it easier for us reason about the problem.

Given two strings \\( u, v \in \Sigma^* \\) we write \\( u < v \\) if \\( u \\) is a <strong>prefix</strong> of \\( v \\) and \\( u \leq v \\) if \\( u \\) is a <strong>proper prefix</strong> of \\( v \\).

An **infix occurence** of an string \\( v \\) in text \\( t \\) is a triple \\( \langle u, v, w \rangle \\) such that \\( t = u \cdot v \cdot w \\). For example in text \\( t = aabcb, \langle a, ab, cb \rangle \\) and \\( \langle aa, bc, b \rangle \\) infix occurences of the strings "ab" and "bc" respectively.

Two **infix occurences** \\( \langle u_1, v_1, w_1 \rangle \\) and \\( \langle u_2, v_2, w_2 \rangle \\) of a text **overlap** if \\( u_1 < u_2 < u_1 \cdot v_1 \\)  
A set of infix occurrences \\(A\\) of the text t is said to be **non-overlapping** if two distinct infix occurrences of \\(A\\) never overlap. 

<img src="/images/posts/2020-02-22-ambiguity-text-rewriting/overlap.png" alt="infix overlap" width="500"/>
<p class="text-center">
    <small>Figure 1: Overlapping infix occurences</small>
</p>

Let's see an example.

\\[
t = aabbbccab \\\\
A = \\{ \langle a, ab, bbccab \rangle, \langle aabb, bc, cab \rangle, \langle aabbbcc, ab, \epsilon \rangle \\}
\\]

## Leftmost-Longest Match

The _leftmost-longest_ match resolution strategy consists of two parts:

1. For two overlapping texts, we pick the one that starts first in the natural (_left-to-right_) reading order.
2. For distinct matches starting at the same place in the input text, we pick the longest one.

By applying the _leftmost-longest_ strategy in the example above, we get rid of the ambiguity and, thus, end up with the following translations:

\\[ 
aabcb \to_{R^{LML}(T_{R_1})} axcb \\\\
aaaaabbaa \to_{R^{LML}(T_{R_2})} xby
\\]

### Formal Description

We've developed an intuition for this strategy, let's define the strategy in formal terms so we can be absolutely precise. Given two sets of infix occurences \\(A, B\\), we define the following functions:

\\[
AFTER(A, B) = \\{ \langle u, v, w \rangle \in A | \forall \langle u', v', w' \rangle \in B : u' \cdot v' \leq u \land u' < u  \\}
\\]

\\( AFTER \\) selects from all infix occurrences in \\( A \\) those where the middle component (\\(v\\)) starts after all middle components of the set \\(B\\).

\\[
LEFTMOST(A) = \\{ \langle u, v, w \rangle \in A | \forall \langle u', v', w' \rangle \in A : u \leq u' \\}
\\]

\\( LEFTMOST \\) selects the infix occurences where the middle components has the leftmost position. Note that we might have multiple such components that start from the same position.

\\[
LONGEST(A) = \\{ \langle u, v, w \rangle \in A | \forall \langle u', v', w' \rangle \in A : u \neq u' \lor v' \leq v \\}
\\]

From the infix occurences with a middle component starting from the same position, \\( LONGEST \\) selects the ones with the longest middle part.

Now we can define the function \\( LML(A) \\) which given a set of infix ocurrances is going to return its subset consisting of the _leftmost-longest_ ones.

\\[
LML(A) := \bigcup\limits_{i=0}^{\infty} V_{i}
\\]

Where the intermediate sets \\( V_i \\) are defined inductively.

\\[
V_0 = \emptyset \\\\
V_{i+1} = V_i \cup LONGEST(LEFTMOST(AFTER(V_i, A)))
\\]

\\( LML(A) \\) is always finite because \\( A \\) is finite. That is, after a certain induction step \\( i, V_i \\) will remain unchanged and that's when we break from the loop. At each step we add at most 1 element to the resulting set.

Note that also all the infix occurences in \\( LML(A) \\) are **non-overlapping** which means that for an input text \\(t\\) we're going to have only one output string. This leads to another useful property, that is, given a rewrite function \\( T: \Sigma^+ \to \Sigma^* \\) the obligatory rewrite relation \\( R^{LML}(T) \\) under the <em>leftmost-longest</em> match strategy is also a function.

## Other Conflict Resolution Strategies

Now we've defined _leftmost-longest_ match, it is trivial to modify the algorithm to obtain different replacement strategies. For example for _leftmost-shortest_ we can replace the \\(LONGEST\\) function with 

\\[
SHORTEST(A) = \\{ \langle u, v, w \rangle \in A | \forall \langle u', v', w' \rangle \in A : u \neq u' \lor v \leq v' \\}
\\]

We can also pick the rightmost out of several overlapping infix occurences by replacing \\(LEFTMOST\\) with

\\[
RIGHTMOST(A) = \\{ \langle u, v, w \rangle \in A | \forall \langle u', v', w' \rangle \in A : u' \leq u \\}
\\]

By combining those functinos, we can define any strategy we want, however, from now on, we'll focus on the _leftmost-longest_ as it is more applicable in practice. Most of the times, the reading order is from left to right and long occurences usually carry more information than the shorter ones.

## Conclusion

In this article, we discussed the problem of having overlapping replacement candidates in text rewriting. We defined the _leftmost-longest_ match strategy for resolving such conflicts and learned how to implement it from a formal standpoint. By performing small modifications, we saw how to define other kinds of strategies likes _leftmost-shortest_, _rightmost-longest_ etc.  

In the following article we'll put what we've learned into practice by converting these formalisms into code and implement a text rewriter for obligatory, _leftmost-longest_ match replacement.

## References

* [Finite-State Transducers for Text Rewriting](/2020/02/18/finite-state-transducers-for-text-rewriting/)
* ["Finite State Techniques - Automata, Transducers and Bimachines" by Mihov &
Schulz](https://www.cambridge.org/core/books/finitestate-techniques/E21E748468F0310DA12A2CFAEB989185) Chapter 11.3 - Letmost-Longest Match Text Rewriting
* [N-Way Composition of Weighted Finite-State Transducers](https://cs.nyu.edu/~mohri/pub/nway.pdf)
* ["Regular Models for Phonological Rule Systems" by Kaplan & Kay](https://web.stanford.edu/~mjkay/Kaplan%26Kay.pdf)
