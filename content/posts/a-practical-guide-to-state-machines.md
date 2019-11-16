---
title: "A Practical Guide to  State Machines"
date: 2019-11-16T13:15:47+02:00
draft: true
summary: "Express complex application logic in an elegant and concise way using C#'s pattern matching."
tags: ["csharp", "state-machines", "software-design"]
---

In this article, we'll examine some examples of real-world problems that can be solved in an elegant way using state machines. The examples will make use of C#'s pattern matching capabilities.

## An Informal Definition

A finite state machine is an abstract construct that has states and transitions between these states. It is always in one of its states and while it reads an input it switches from state to state. Think of it as a directed graph. A finite state machine has no memory, that is, it does not keep track of the previous states it has been in. It only knows its current state.  
For a state machine to be deterministic means that on each input there is one and only one state to which it can transition from its current state. All of the examples in this article are of deterministic state machines.  

<svg width="800" height="170" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="none" cx="218.5" cy="89.5" rx="30" ry="30"/>
	<text x="196.5" y="95.5" font-family="Times New Roman" font-size="20">Open</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="365.5" cy="89.5" rx="30" ry="30"/>
	<text x="337.5" y="95.5" font-family="Times New Roman" font-size="20">Closed</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="498.5" cy="89.5" rx="30" ry="30"/>
	<text x="468.5" y="95.5" font-family="Times New Roman" font-size="20">Locked</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 234.842,64.554 A 79.509,79.509 0 0 1 349.158,64.554"/>
	<polygon fill="black" stroke-width="1" points="349.158,64.554 347.192,55.327 340.003,62.278"/>
	<text x="271.5" y="31.5" font-family="Times New Roman" font-size="20">close</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 346.918,112.846 A 83.41,83.41 0 0 1 237.082,112.846"/>
	<polygon fill="black" stroke-width="1" points="237.082,112.846 239.811,121.877 246.395,114.35"/>
	<text x="272.5" y="154.5" font-family="Times New Roman" font-size="20">open</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 378.187,62.582 A 68.147,68.147 0 0 1 485.813,62.582"/>
	<polygon fill="black" stroke-width="1" points="485.813,62.582 484.853,53.197 476.956,59.332"/>
	<text x="414.5" y="27.5" font-family="Times New Roman" font-size="20">lock</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 483.841,115.409 A 69.595,69.595 0 0 1 380.159,115.409"/>
	<polygon fill="black" stroke-width="1" points="380.159,115.409 381.771,124.704 389.22,118.032"/>
	<text x="404.5" y="159.5" font-family="Times New Roman" font-size="20">unlock</text>
</svg>
<p class="text-center"><small>Figure 1: Representation of a door using a state machine</small></p>

The state machine on Figure 1 has:

- a set of states: `Open`, `Closed`, `Locked`
- a set of inputs: `open`, `close`, `lock`, `unlock`
- a transition function of type: `(State, Input) -> State`

In addition a state machine has an **initial state** (we can pick an arbitrary one in Figure 1) and a **set of final states** (which is empty on Figure 1).


## Representing State Machines in C\#

State machines are simply directed graphs and there're various ways to construct one. We can use an [adjacency matrix](https://en.wikipedia.org/wiki/Adjacency_matrix), use an object oriented approach via [the state pattern](https://en.wikipedia.org/wiki/State_pattern) or encode the state and the transitions in a mapping structure such as C#'s `Dictionary<TKey, TValue>`.

The recent developments in C# version 8 added significant improvements in its pattern matching capabilities and the **switch expressions** in particular  (which provided some inspiration for this article). If you haven't checked them out, I recommend reading [this article](https://devblogs.microsoft.com/dotnet/do-more-with-patterns-in-c-8-0/) by Mads Torgersen.

The state machine on Figure 1 can be encoded as:

```csharp
enum State { Open, Closed, Locked }
enum Action { Open, Close, Lock, Unlock }

State ChangeState(State current, Action action) => 
    (current, action) switch
	{
        (State.Closed, Action.Open) => State.Open,
        (State.Open, Action.Close) => State.Closed,
        (State.Closed, Action.Lock) => State.Locked,
        (State.Locked, Action.Unlock) => State.Closed,
        _ => throw new NotSupportedException(
            $"{current} has no transition on {action}")
	};
```

Processing the inputs is simple:

```csharp
var current = DoorState.Closed; // the initial state
current = ChangeState(current, Action.Open); // Opened
current = ChangeState(current, Action.Close); // Closed
current = ChangeState(current, Action.Lock); // Locked
current = ChangeState(current, Action.Open); // throws
```

## State Machines with Async Actions

Let's take a look at another example. This time we'll implement process management workflow similar to the one in UNIX.

<svg width="800" height="275" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="none" cx="142.5" cy="42.5" rx="30" ry="30"/>
	<text x="123.5" y="48.5" font-family="Times New Roman" font-size="20">New</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="228.5" cy="109.5" rx="30" ry="30"/>
	<text x="202.5" y="115.5" font-family="Times New Roman" font-size="20">Ready</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="451.5" cy="117.5" rx="30" ry="30"/>
	<text x="417.5" y="123.5" font-family="Times New Roman" font-size="20">Running</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="339.5" cy="226.5" rx="30" ry="30"/>
	<text x="308.5" y="232.5" font-family="Times New Roman" font-size="20">Waiting</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="548.5" cy="42.5" rx="30" ry="30"/>
	<text x="503.5" y="48.5" font-family="Times New Roman" font-size="20">Terminated</text>
	<polygon stroke="black" stroke-width="1" points="166.166,60.937 204.834,91.063"/>
	<polygon fill="black" stroke-width="1" points="204.834,91.063 201.596,82.202 195.451,90.09"/>
	<text x="134.5" y="96.5" font-family="Times New Roman" font-size="20">admit</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 249.315,87.979 A 137.456,137.456 0 0 1 432.28,94.543"/>
	<polygon fill="black" stroke-width="1" points="432.28,94.543 429.969,85.396 423.046,92.613"/>
	<text x="273.5" y="44.5" font-family="Times New Roman" font-size="20">schedule dispatch</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 426.252,133.634 A 175.265,175.265 0 0 1 252.527,127.402"/>
	<polygon fill="black" stroke-width="1" points="252.527,127.402 256.694,135.866 261.961,127.366"/>
	<text x="303.5" y="175.5" font-family="Times New Roman" font-size="20">interrupt</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 449.645,147.332 A 100.929,100.929 0 0 1 369.371,225.456"/>
	<polygon fill="black" stroke-width="1" points="369.371,225.456 378.15,228.911 376.324,219.079"/>
	<text x="426.5" y="219.5" font-family="Times New Roman" font-size="20">I/O or event wait</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 309.779,223.198 A 108.337,108.337 0 0 1 230.234,139.354"/>
	<polygon fill="black" stroke-width="1" points="230.234,139.354 226.892,148.176 236.7,146.224"/>
	<text x="77.5" y="213.5" font-family="Times New Roman" font-size="20">I/O or event complete</text>
	<polygon stroke="black" stroke-width="1" points="475.233,99.15 524.767,60.85"/>
	<polygon fill="black" stroke-width="1" points="524.767,60.85 515.38,61.788 521.496,69.699"/>
	<text x="464.5" y="71.5" font-family="Times New Roman" font-size="20">exit</text>
</svg>
<p class="text-center"><small>Figure 2: A process state machine</small></p>

On Figure 2, we have the following sets of states and inputs:

```csharp
enum State { New, Ready, Running, Waiting, Terminated }
enum Input { 
    Admit, ScheduleDispatch, Interrupt,
    IOorEventWait, IOorEventComplete, Exit 
}
```

The transitions are described as:

```csharp
static State ChangeState(State current, Input input) =>
    (current, input) switch
    {
        (State.New, Input.Admit) => State.Ready,
        (State.Ready, Input.ScheduleDispatch) => State.Running,
        (State.Running, Input.IOorEventWait) => State.Waiting,
        (State.Waiting, Input.IOorEventComplete) => State.Ready,
        (State.Running, Input.Interrupt) => State.Ready,
        (State.Running, Input.Exit) => State.Terminated,
        _ => throw new NotSupportedException(
            $"{current} has no transition on {input}")
    };
```

Suppose we want to execute some action on state change. Keep in mind that what comes after the `=>` is an expression. So far we were simply returning values, now we want to run some arbitrary code. The approach might not seem very intuitive, but for those familiar with JavaScript it should ring a bell. We're going to wrap our code in an _immediately invoked function expression [(IIFE)](https://developer.mozilla.org/en-US/docs/Glossary/IIFE)_ using C#'s lambda syntax.

```csharp
static State ChangeState(State current, Input input) =>
    (current, input) switch
	{
        (State.New, Input.Admit) => ((Func<State>)(() => {
            ExecuteSomeAction();
            return State.Ready;
        }))(),
        // ...
    };
```

The usage of our `ChangeState` method remains unchanged.

```csharp
var current = ChangeState(State.New, Input.Admit); // Ready
```

If `ExecuteSomeAction()` is a long running operation, it would make sense to unblock the current thread by making it asynchronous. That requires us to refactor our state machine.

```csharp
static async Task<State> ChangeState(State current, Input input) =>
    (current, input) switch
    {
        (State.New, Input.Admit) => 
            await ((Func<Task<State>>)(async () => {
                await ExecuteSomeActionAsync();
                return State.Ready;g
            }))(),
        (State.Ready, Input.ScheduleDispatch) => 
            await Task.FromResult(State.Running),
        // ...
   	};
```

```csharp
var current = await ChangeState(State.New, Input.Admit); // Ready
```

## Composition Of State Machines

## State Machine Libraries
* [stateless](https://github.com/dotnet-state-machine/stateless)  
* [workflow-core](https://github.com/danielgerlag/workflow-core)  

## Recap
Why is this important?  
Mention applications in language recognition, async state machines.

## References and Further Reading
* _"Introduction To Automata Theory Languages, and Computation"_ Hopcroft, Ullmann, Motwani; Chapter 2 - Finite Automata
* [Do more with patterns in C# 8.0](https://devblogs.microsoft.com/dotnet/do-more-with-patterns-in-c-8-0/)
* [Robust React User Interfaces with Finite State Machines](https://css-tricks.com/robust-react-user-interfaces-with-finite-state-machines/)
* [Finite State Machine Desigher](http://madebyevan.com/fsm/)
