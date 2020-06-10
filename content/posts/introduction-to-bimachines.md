---
title: "Introduction to Bimachines"
date: 2020-06-10T09:31:46+03:00
draft: true
useMath: true
---

Bimachines are deterministic finite-state devices that represent the class regular string functions and are commonly used for text rewriting. What is fascinating about them is that they process the input text in both directions - from left to right and from right to left which makes them very efficient in practice as they guarantee linear time input processing.

Bimachines have less expressive power that the finite-state transducers (FST), because FSTs are able to also model the class of regular relations (for one input we may have more that one output). FSTs, however, are generally non-deterministic, which means that, unlike bimachines, they don't might not be as efficient. Unlike finite automata, not every non-deterministic transducer can be converted to an equivalent deterministic one. Also deterministic FSTs are not powerful enough to cover the class of regular functions, so bimachines stands kind of inbetween.

In practice, when implementing algorithms, we are typically more interested in deterministic outputs, which makes the bimachines suitable for a variety of tasks due to also their high performance. In this article we'll introduce the concept of a bimachine and see how they work.

## Bimachines

A bimachine is a triple \\( \mathcal{B} = \langle \mathcal{A}_L, \mathcal{A}_R, \psi \rangle \\), where \\( \mathcal{A}_L, \mathcal{A}_R \\) are deterministic finite automata and \\( \psi: Q_L \times \Sigma \times Q_R \to \Sigma^* \\) is called the output function of the bimachine which given a state of the left automaton, an input symbol and a state in the right automaton, it returns a string. The left automaton of the bimachine scans the text from left to right, whereas the right automaton scans from right to left. Once we get the two paths, we construct the output by concatenating the results of the output function. 

Let's go though an example.

Simulating a bimachine

## Regular functions as Finite-State Transducers


## Conclusion

Besides plain text rewriting, bimachines can be used for various linguistic tasks, such as text annottation, tagging, tokenization etc. They can also be generalized in a sense that the output is in a monoid (string output is a just special 
type called a [free monoid](https://en.wikipedia.org/wiki/Free_monoid)). Although having been around since the 1960's, bimachines remain relatively unexplored. As of now, the state of the are algorithms for constructing bimachines require us to build the corresponding FST first. Links to some papers on the topic can be found in the references below.

## References

- Finite State Techniques

