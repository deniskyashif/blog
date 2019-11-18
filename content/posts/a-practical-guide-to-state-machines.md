---
title: "A Practical Guide to  State Machines"
date: 2019-11-16T13:15:47+02:00
draft: true
summary: "Express complex application logic in an elegant and concise way using C#'s pattern matching."
tags: ["csharp", "state-machines", "software-design"]
---

In this article, we'll examine some examples of real-world problems that can be solved in an elegant way using state machines. The examples will make use of C#'s pattern matching capabilities.

## An Informal Definition

A finite state machine is an abstract construct that has states and transitions between these states. It is always in one of its states and while it reads an input it switches from state to state. Think of it as a directed graph (possibly cyclic). A finite state machine has no memory, that is, it does not keep track of the previous states it has been in. It only knows its current state.  
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

The state machine on Figure 1 can be implemented as:

```csharp
enum State { Open, Closed, Locked }
enum Input { Open, Close, Lock, Unlock }

State ChangeState(State current, Input input) => 
    (current, action) switch
	{
        (State.Closed, Input.Open) => State.Open,
        (State.Open, Input.Close) => State.Closed,
        (State.Closed, Input.Lock) => State.Locked,
        (State.Locked, Input.Unlock) => State.Closed,
        _ => throw new NotSupportedException(
            $"{current} has no transition on {input}")
	};
```

Processing the inputs is simple:

```csharp
var current = DoorState.Closed; // the initial state
current = ChangeState(current, Input.Open); // Opened
current = ChangeState(current, Input.Close); // Closed
current = ChangeState(current, Input.Lock); // Locked
current = ChangeState(current, Input.Open); // throws
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

We can also create conditional transitions using the `when` keyword.

```csharp
static State ChangeState(State current, Input input, bool hasPermission) =>
    (current, input) switch
    {
        (State.New, Input.Admit) when hasPermission => State.Ready
        // ...
    };
```

Suppose we want to execute some transition action on state change. Keep in mind that what comes after the `=>` is an expression. So far we were simply returning values, now we want to run some arbitrary code. The approach might not seem very intuitive, but for those familiar with JavaScript it should ring a bell. We're going to wrap our code in an _immediately invoked function expression [(IIFE)](https://developer.mozilla.org/en-US/docs/Glossary/IIFE)_ using C#'s lambda syntax.

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
                return State.Ready;
            }))(),
        (State.Ready, Input.ScheduleDispatch) => 
            await Task.FromResult(State.Running),
        // ...
   	};
```

```csharp
var current = await ChangeState(State.New, Input.Admit); // Ready
```

## Combining State Machines

In this example, we'll see how separate state machines can interact with each other to form a unified workflow.

We're given the task to design an online store where customers can purchase items by issuing bank transactions. The system has three participants: the customer, the store and the bank with the following requirements:

- The customer may _pay_ or _cancel_ at any given time.
- The store may _ship_ items to the customer.
- The store may _redeem_ the money from the bank.
- The bank may _transfer_ any redeemed money (e.g. to the store).

We have to ensure that the customer cannot use one issued payment multiple times, neither _pay_ and immediately _cancel_ the money, thus getting the items for free. The bank must ensure that money cannot be _canceled_ and _redeemed_ at the same time. The store should _ship_ the items only when it gets the payment.

For each of the participants, we have to model a separate state machine. Let's start with the bank.

<svg width="800" height="200" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="none" cx="242.5" cy="160.5" rx="30" ry="30"/>
	<text x="237.5" y="166.5" font-family="Times New Roman" font-size="20">1</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="242.5" cy="35.5" rx="30" ry="30"/>
	<text x="237.5" y="41.5" font-family="Times New Roman" font-size="20">2</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="384.5" cy="160.5" rx="30" ry="30"/>
	<text x="379.5" y="166.5" font-family="Times New Roman" font-size="20">3</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="525.5" cy="160.5" rx="30" ry="30"/>
	<text x="520.5" y="166.5" font-family="Times New Roman" font-size="20">4</text>
	<polygon stroke="black" stroke-width="1" points="242.5,130.5 242.5,65.5"/>
	<polygon fill="black" stroke-width="1" points="242.5,65.5 237.5,73.5 247.5,73.5"/>
	<text x="247.5" y="104.5" font-family="Times New Roman" font-size="20">cancel</text>
	<polygon stroke="black" stroke-width="1" points="272.5,160.5 354.5,160.5"/>
	<polygon fill="black" stroke-width="1" points="354.5,160.5 346.5,155.5 346.5,165.5"/>
	<text x="284.5" y="181.5" font-family="Times New Roman" font-size="20">redeem</text>
	<polygon stroke="black" stroke-width="1" points="414.5,160.5 495.5,160.5"/>
	<polygon fill="black" stroke-width="1" points="495.5,160.5 487.5,155.5 487.5,165.5"/>
	<text x="424.5" y="181.5" font-family="Times New Roman" font-size="20">transfer</text>
</svg>
<p class="text-center"><small>Figure 3.1: The Bank</small></p>

The initial state `1` represents the situation when the bank has issued the money but has not been requested to either _redeem_ it (by the store) or _cancel_ it (by the customer).

<svg width="800" height="200" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="none" cx="125.5" cy="41.5" rx="30" ry="30"/>
	<text x="121.5" y="47.5" font-family="Times New Roman" font-size="20">a</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="258.5" cy="41.5" rx="30" ry="30"/>
	<text x="253.5" y="47.5" font-family="Times New Roman" font-size="20">b</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="258.5" cy="164.5" rx="30" ry="30"/>
	<text x="254.5" y="170.5" font-family="Times New Roman" font-size="20">c</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="402.5" cy="41.5" rx="30" ry="30"/>
	<text x="397.5" y="47.5" font-family="Times New Roman" font-size="20">d</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="402.5" cy="164.5" rx="30" ry="30"/>
	<text x="398.5" y="170.5" font-family="Times New Roman" font-size="20">e</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="551.5" cy="41.5" rx="30" ry="30"/>
	<text x="548.5" y="47.5" font-family="Times New Roman" font-size="20">f</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="551.5" cy="164.5" rx="30" ry="30"/>
	<text x="546.5" y="170.5" font-family="Times New Roman" font-size="20">g</text>
	<polygon stroke="black" stroke-width="1" points="155.5,41.5 228.5,41.5"/>
	<polygon fill="black" stroke-width="1" points="228.5,41.5 220.5,36.5 220.5,46.5"/>
	<text x="177.5" y="32.5" font-family="Times New Roman" font-size="20">pay</text>
	<polygon stroke="black" stroke-width="1" points="288.5,41.5 372.5,41.5"/>
	<polygon fill="black" stroke-width="1" points="372.5,41.5 364.5,36.5 364.5,46.5"/>
	<text x="301.5" y="32.5" font-family="Times New Roman" font-size="20">redeem</text>
	<polygon stroke="black" stroke-width="1" points="432.5,41.5 521.5,41.5"/>
	<polygon fill="black" stroke-width="1" points="521.5,41.5 513.5,36.5 513.5,46.5"/>
	<text x="446.5" y="32.5" font-family="Times New Roman" font-size="20">transfer</text>
	<polygon stroke="black" stroke-width="1" points="551.5,71.5 551.5,134.5"/>
	<polygon fill="black" stroke-width="1" points="551.5,134.5 556.5,126.5 546.5,126.5"/>
	<text x="556.5" y="109.5" font-family="Times New Roman" font-size="20">ship</text>
	<polygon stroke="black" stroke-width="1" points="402.5,71.5 402.5,134.5"/>
	<polygon fill="black" stroke-width="1" points="402.5,134.5 407.5,126.5 397.5,126.5"/>
	<text x="407.5" y="109.5" font-family="Times New Roman" font-size="20">ship</text>
	<polygon stroke="black" stroke-width="1" points="258.5,71.5 258.5,134.5"/>
	<polygon fill="black" stroke-width="1" points="258.5,134.5 263.5,126.5 253.5,126.5"/>
	<text x="263.5" y="109.5" font-family="Times New Roman" font-size="20">ship</text>
	<polygon stroke="black" stroke-width="1" points="288.5,164.5 372.5,164.5"/>
	<polygon fill="black" stroke-width="1" points="372.5,164.5 364.5,159.5 364.5,169.5"/>
	<text x="301.5" y="185.5" font-family="Times New Roman" font-size="20">redeem</text>
	<polygon stroke="black" stroke-width="1" points="432.5,164.5 521.5,164.5"/>
	<polygon fill="black" stroke-width="1" points="521.5,164.5 513.5,159.5 513.5,169.5"/>
	<text x="446.5" y="185.5" font-family="Times New Roman" font-size="20">transfer</text>
</svg>
<p class="text-center"><small>Figure 3.2: The Store</small></p>

The customer's state machine is simple. It has only one state and two input actions which lead back to the same state. This means that the customer **can do anything**.

<svg width="800" height="80" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="none" cx="268.5" cy="48.5" rx="30" ry="30"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 295.297,35.275 A 22.5,22.5 0 1 1 295.297,61.725"/>
	<text x="341.5" y="54.5" font-family="Times New Roman" font-size="20">pay, cancel</text>
	<polygon fill="black" stroke-width="1" points="295.297,61.725 298.83,70.473 304.708,62.382"/>
</svg>
<p class="text-center"><small>Figure 3.3: The Customer</small></p>

### Dealing with the Missing Action Inputs

To combine these state machines means that they should be able to process inputs simultaneously. 
We observe that some transitions are not present on all of them. For example, the store doesn't have a notion of the _cancel_ action. For the bank, _pay_ and _ship_ are irrelevant. To make the store "ignore" the _cancel_ action, we simply have to add a transition from each of its states to itself on _cancel_. We do this to every state for each missing action input.

<svg width="800" height="260" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="none" cx="242.5" cy="160.5" rx="30" ry="30"/>
	<text x="237.5" y="166.5" font-family="Times New Roman" font-size="20">1</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="242.5" cy="33.5" rx="30" ry="30"/>
	<text x="237.5" y="39.5" font-family="Times New Roman" font-size="20">2</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="384.5" cy="160.5" rx="30" ry="30"/>
	<text x="379.5" y="166.5" font-family="Times New Roman" font-size="20">3</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="525.5" cy="160.5" rx="30" ry="30"/>
	<text x="520.5" y="166.5" font-family="Times New Roman" font-size="20">4</text>
	<polygon stroke="black" stroke-width="1" points="242.5,130.5 242.5,63.5"/>
	<polygon fill="black" stroke-width="1" points="242.5,63.5 237.5,71.5 247.5,71.5"/>
	<text x="186.5" y="103.5" font-family="Times New Roman" font-size="20">cancel</text>
	<polygon stroke="black" stroke-width="1" points="272.5,160.5 354.5,160.5"/>
	<polygon fill="black" stroke-width="1" points="354.5,160.5 346.5,155.5 346.5,165.5"/>
	<text x="284.5" y="181.5" font-family="Times New Roman" font-size="20">redeem</text>
	<polygon stroke="black" stroke-width="1" points="414.5,160.5 495.5,160.5"/>
	<polygon fill="black" stroke-width="1" points="495.5,160.5 487.5,155.5 487.5,165.5"/>
	<text x="424.5" y="181.5" font-family="Times New Roman" font-size="20">transfer</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 215.703,173.725 A 22.5,22.5 0 1 1 215.703,147.275"/>
	<text x="99.5" y="166.5" font-family="Times New Roman" font-size="20">pay, ship</text>
	<polygon fill="black" stroke-width="1" points="215.703,147.275 212.17,138.527 206.292,146.618"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 217.469,49.824 A 22.5,22.5 0 1 1 214.318,23.562"/>
	<text x="97.5" y="48.5" font-family="Times New Roman" font-size="20">pay, ship</text>
	<polygon fill="black" stroke-width="1" points="214.318,23.562 209.768,15.298 204.896,24.031"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 371.275,133.703 A 22.5,22.5 0 1 1 397.725,133.703"/>
	<text x="284.5" y="84.5" font-family="Times New Roman" font-size="20">pay, redeem, cancel, ship</text>
	<polygon fill="black" stroke-width="1" points="397.725,133.703 406.473,130.17 398.382,124.292"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 538.725,187.297 A 22.5,22.5 0 1 1 512.275,187.297"/>
	<text x="425.5" y="249.5" font-family="Times New Roman" font-size="20">pay, redeem, cancel, ship</text>
	<polygon fill="black" stroke-width="1" points="512.275,187.297 503.527,190.83 511.618,196.708"/>
</svg>

<svg width="800" height="320" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="none" cx="125.5" cy="112.5" rx="30" ry="30"/>
	<text x="121.5" y="118.5" font-family="Times New Roman" font-size="20">a</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="258.5" cy="112.5" rx="30" ry="30"/>
	<text x="253.5" y="118.5" font-family="Times New Roman" font-size="20">b</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="258.5" cy="224.5" rx="30" ry="30"/>
	<text x="254.5" y="230.5" font-family="Times New Roman" font-size="20">c</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="402.5" cy="112.5" rx="30" ry="30"/>
	<text x="397.5" y="118.5" font-family="Times New Roman" font-size="20">d</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="402.5" cy="224.5" rx="30" ry="30"/>
	<text x="398.5" y="230.5" font-family="Times New Roman" font-size="20">e</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="551.5" cy="112.5" rx="30" ry="30"/>
	<text x="548.5" y="118.5" font-family="Times New Roman" font-size="20">f</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="551.5" cy="224.5" rx="30" ry="30"/>
	<text x="546.5" y="230.5" font-family="Times New Roman" font-size="20">g</text>
	<polygon stroke="black" stroke-width="1" points="155.5,112.5 228.5,112.5"/>
	<polygon fill="black" stroke-width="1" points="228.5,112.5 220.5,107.5 220.5,117.5"/>
	<text x="177.5" y="103.5" font-family="Times New Roman" font-size="20">pay</text>
	<polygon stroke="black" stroke-width="1" points="288.5,112.5 372.5,112.5"/>
	<polygon fill="black" stroke-width="1" points="372.5,112.5 364.5,107.5 364.5,117.5"/>
	<text x="301.5" y="103.5" font-family="Times New Roman" font-size="20">redeem</text>
	<polygon stroke="black" stroke-width="1" points="432.5,112.5 521.5,112.5"/>
	<polygon fill="black" stroke-width="1" points="521.5,112.5 513.5,107.5 513.5,117.5"/>
	<text x="446.5" y="103.5" font-family="Times New Roman" font-size="20">transfer</text>
	<polygon stroke="black" stroke-width="1" points="551.5,142.5 551.5,194.5"/>
	<polygon fill="black" stroke-width="1" points="551.5,194.5 556.5,186.5 546.5,186.5"/>
	<text x="556.5" y="174.5" font-family="Times New Roman" font-size="20">ship</text>
	<polygon stroke="black" stroke-width="1" points="402.5,142.5 402.5,194.5"/>
	<polygon fill="black" stroke-width="1" points="402.5,194.5 407.5,186.5 397.5,186.5"/>
	<text x="407.5" y="174.5" font-family="Times New Roman" font-size="20">ship</text>
	<polygon stroke="black" stroke-width="1" points="258.5,142.5 258.5,194.5"/>
	<polygon fill="black" stroke-width="1" points="258.5,194.5 263.5,186.5 253.5,186.5"/>
	<text x="263.5" y="174.5" font-family="Times New Roman" font-size="20">ship</text>
	<polygon stroke="black" stroke-width="1" points="288.5,224.5 372.5,224.5"/>
	<polygon fill="black" stroke-width="1" points="372.5,224.5 364.5,219.5 364.5,229.5"/>
	<text x="301.5" y="245.5" font-family="Times New Roman" font-size="20">redeem</text>
	<polygon stroke="black" stroke-width="1" points="432.5,224.5 521.5,224.5"/>
	<polygon fill="black" stroke-width="1" points="521.5,224.5 513.5,219.5 513.5,229.5"/>
	<text x="446.5" y="245.5" font-family="Times New Roman" font-size="20">transfer</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 112.275,85.703 A 22.5,22.5 0 1 1 138.725,85.703"/>
	<text x="99.5" y="36.5" font-family="Times New Roman" font-size="20">cancel</text>
	<polygon fill="black" stroke-width="1" points="138.725,85.703 147.473,82.17 139.382,76.292"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 245.275,85.703 A 22.5,22.5 0 1 1 271.725,85.703"/>
	<text x="214.5" y="35.5" font-family="Times New Roman" font-size="20">pay, cancel</text>
	<polygon fill="black" stroke-width="1" points="271.725,85.703 280.473,82.17 272.382,76.292"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 389.275,85.703 A 22.5,22.5 0 1 1 415.725,85.703"/>
	<text x="358.5" y="35.5" font-family="Times New Roman" font-size="20">pay, cancel</text>
	<polygon fill="black" stroke-width="1" points="415.725,85.703 424.473,82.17 416.382,76.292"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 538.275,85.703 A 22.5,22.5 0 1 1 564.725,85.703"/>
	<text x="507.5" y="35.5" font-family="Times New Roman" font-size="20">pay, cancel</text>
	<polygon fill="black" stroke-width="1" points="564.725,85.703 573.473,82.17 565.382,76.292"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 271.725,251.297 A 22.5,22.5 0 1 1 245.275,251.297"/>
	<text x="214.5" y="313.5" font-family="Times New Roman" font-size="20">pay, cancel</text>
	<polygon fill="black" stroke-width="1" points="245.275,251.297 236.527,254.83 244.618,260.708"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 415.725,251.297 A 22.5,22.5 0 1 1 389.275,251.297"/>
	<text x="358.5" y="313.5" font-family="Times New Roman" font-size="20">pay, cancel</text>
	<polygon fill="black" stroke-width="1" points="389.275,251.297 380.527,254.83 388.618,260.708"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 564.725,251.297 A 22.5,22.5 0 1 1 538.275,251.297"/>
	<text x="507.5" y="313.5" font-family="Times New Roman" font-size="20">pay, cancel</text>
	<polygon fill="black" stroke-width="1" points="538.275,251.297 529.527,254.83 537.618,260.708"/>
</svg>

<svg width="800" height="80" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="none" cx="170.5" cy="44.5" rx="30" ry="30"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 197.297,31.275 A 22.5,22.5 0 1 1 197.297,57.725"/>
	<text x="243.5" y="50.5" font-family="Times New Roman" font-size="20">pay, cancel, ship, redeem, transfer</text>
	<polygon fill="black" stroke-width="1" points="197.297,57.725 200.83,66.473 206.708,58.382"/>
</svg>
<p class="text-center"><small>Figure 3.4: The complete sets of transitions for the bank, the store and the customer</small></p>

### Constructing the Product State Machine

It's still not quite obvious how do the states in the store and the bank interact. To see this more clearly, we need to construct **the product** of these state machines. The product is a state machine itself, and since the customer automaton only has one state, the states of the product are pairs of states from the bank and from the store.
For example, the state `(3,d)` represents the situation where the bank is in state `3` and the store is in state `d`. The total amount of states is 4x7=28. We group them in a table where the row corresponds to the state of the bank and the column to the state of the store. The initial state is the pair, both items of which are initial states of their respective machines - in our case `(1, a)`.

<svg width="800" height="600" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="#e8f7e4" cx="46.5" cy="91.5" rx="30" ry="30"/>
	<text x="32.5" y="97.5" font-family="Times New Roman" font-size="20">1, a</text>
	<ellipse stroke="black" stroke-width="1" fill="#e8f7e4" cx="160.5" cy="91.5" rx="30" ry="30"/>
	<text x="145.5" y="97.5" font-family="Times New Roman" font-size="20">1, b</text>
	<ellipse stroke="black" stroke-width="1" fill="#e8f7e4" cx="274.5" cy="91.5" rx="30" ry="30"/>
	<text x="260.5" y="97.5" font-family="Times New Roman" font-size="20">1, c</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="389.5" cy="91.5" rx="30" ry="30"/>
	<text x="374.5" y="97.5" font-family="Times New Roman" font-size="20">1, d</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="509.5" cy="91.5" rx="30" ry="30"/>
	<text x="495.5" y="97.5" font-family="Times New Roman" font-size="20">1, e</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="627.5" cy="91.5" rx="30" ry="30"/>
	<text x="614.5" y="97.5" font-family="Times New Roman" font-size="20">1, f</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="736.5" cy="91.5" rx="30" ry="30"/>
	<text x="721.5" y="97.5" font-family="Times New Roman" font-size="20">1, g</text>
	<ellipse stroke="black" stroke-width="1" fill="#e8f7e4" cx="46.5" cy="229.5" rx="30" ry="30"/>
	<text x="32.5" y="235.5" font-family="Times New Roman" font-size="20">2, a</text>
	<ellipse stroke="black" stroke-width="1" fill="#e8f7e4" cx="160.5" cy="229.5" rx="30" ry="30"/>
	<text x="145.5" y="235.5" font-family="Times New Roman" font-size="20">2, b</text>
	<ellipse stroke="black" stroke-width="1" fill="#e8f7e4" cx="274.5" cy="229.5" rx="30" ry="30"/>
	<text x="260.5" y="235.5" font-family="Times New Roman" font-size="20">2, c</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="389.5" cy="229.5" rx="30" ry="30"/>
	<text x="374.5" y="235.5" font-family="Times New Roman" font-size="20">2, d</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="509.5" cy="229.5" rx="30" ry="30"/>
	<text x="495.5" y="235.5" font-family="Times New Roman" font-size="20">2, e</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="627.5" cy="229.5" rx="30" ry="30"/>
	<text x="614.5" y="235.5" font-family="Times New Roman" font-size="20">2, f</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="736.5" cy="229.5" rx="30" ry="30"/>
	<text x="721.5" y="235.5" font-family="Times New Roman" font-size="20">2, g</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="46.5" cy="361.5" rx="30" ry="30"/>
	<text x="32.5" y="367.5" font-family="Times New Roman" font-size="20">3, a</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="169.5" cy="361.5" rx="30" ry="30"/>
	<text x="154.5" y="367.5" font-family="Times New Roman" font-size="20">3, b</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="281.5" cy="361.5" rx="30" ry="30"/>
	<text x="267.5" y="367.5" font-family="Times New Roman" font-size="20">3, c</text>
	<ellipse stroke="black" stroke-width="1" fill="#e8f7e4" cx="389.5" cy="361.5" rx="30" ry="30"/>
	<text x="374.5" y="367.5" font-family="Times New Roman" font-size="20">3, d</text>
	<ellipse stroke="black" stroke-width="1" fill="#e8f7e4" cx="509.5" cy="361.5" rx="30" ry="30"/>
	<text x="495.5" y="367.5" font-family="Times New Roman" font-size="20">3, e</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="627.5" cy="361.5" rx="30" ry="30"/>
	<text x="614.5" y="367.5" font-family="Times New Roman" font-size="20">3, f</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="736.5" cy="361.5" rx="30" ry="30"/>
	<text x="721.5" y="367.5" font-family="Times New Roman" font-size="20">3, g</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="46.5" cy="507.5" rx="30" ry="30"/>
	<text x="32.5" y="513.5" font-family="Times New Roman" font-size="20">4, a</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="169.5" cy="507.5" rx="30" ry="30"/>
	<text x="154.5" y="513.5" font-family="Times New Roman" font-size="20">4, b</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="281.5" cy="507.5" rx="30" ry="30"/>
	<text x="267.5" y="513.5" font-family="Times New Roman" font-size="20">4, c</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="389.5" cy="507.5" rx="30" ry="30"/>
	<text x="374.5" y="513.5" font-family="Times New Roman" font-size="20">4, d</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="509.5" cy="507.5" rx="30" ry="30"/>
	<text x="495.5" y="513.5" font-family="Times New Roman" font-size="20">4, e</text>
	<ellipse stroke="black" stroke-width="1" fill="#e8f7e4" cx="633.5" cy="507.5" rx="30" ry="30"/>
	<text x="620.5" y="513.5" font-family="Times New Roman" font-size="20">4, f</text>
	<ellipse stroke="black" stroke-width="1" fill="#e8f7e4" cx="736.5" cy="507.5" rx="30" ry="30"/>
	<text x="721.5" y="513.5" font-family="Times New Roman" font-size="20">4, g</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 715.525,70.215 A 22.5,22.5 0 1 1 740.636,61.905"/>
	<text x="700.5" y="18.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="740.636,61.905 747.83,55.802 738.303,52.764"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 69.756,526.266 A 22.5,22.5 0 1 1 45.75,537.374"/>
	<text x="79.5" y="589.5" font-family="Times New Roman" font-size="20">C</text>
	<polygon fill="black" stroke-width="1" points="45.75,537.374 39.295,544.253 49.106,546.191"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 361.659,350.35 A 350.621,350.621 0 0 1 166.956,120.788"/>
	<polygon fill="black" stroke-width="1" points="361.659,350.35 356.422,342.503 352.311,351.619"/>
	<text x="219.5" y="277.5" font-family="Times New Roman" font-size="20">R</text>
	<polygon stroke="black" stroke-width="1" points="76.5,91.5 130.5,91.5"/>
	<polygon fill="black" stroke-width="1" points="130.5,91.5 122.5,86.5 122.5,96.5"/>
	<text x="97.5" y="82.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon stroke="black" stroke-width="1" points="46.5,121.5 46.5,199.5"/>
	<polygon fill="black" stroke-width="1" points="46.5,199.5 51.5,191.5 41.5,191.5"/>
	<text x="28.5" y="166.5" font-family="Times New Roman" font-size="20">C</text>
	<polygon stroke="black" stroke-width="1" points="160.5,121.5 160.5,199.5"/>
	<polygon fill="black" stroke-width="1" points="160.5,199.5 165.5,191.5 155.5,191.5"/>
	<text x="142.5" y="166.5" font-family="Times New Roman" font-size="20">C</text>
	<polygon stroke="black" stroke-width="1" points="76.5,229.5 130.5,229.5"/>
	<polygon fill="black" stroke-width="1" points="130.5,229.5 122.5,224.5 122.5,234.5"/>
	<text x="97.5" y="220.5" font-family="Times New Roman" font-size="20">P</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 180.588,251.624 A 22.5,22.5 0 1 1 155.158,258.902"/>
	<text x="182.5" y="315.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="155.158,258.902 147.721,264.705 157.116,268.13"/>
	<polygon stroke="black" stroke-width="1" points="190.5,229.5 244.5,229.5"/>
	<polygon fill="black" stroke-width="1" points="244.5,229.5 236.5,224.5 236.5,234.5"/>
	<text x="211.5" y="220.5" font-family="Times New Roman" font-size="20">S</text>
	<polygon stroke="black" stroke-width="1" points="190.5,91.5 244.5,91.5"/>
	<polygon fill="black" stroke-width="1" points="244.5,91.5 236.5,86.5 236.5,96.5"/>
	<text x="211.5" y="82.5" font-family="Times New Roman" font-size="20">S</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 140.11,69.654 A 22.5,22.5 0 1 1 165.437,62.028"/>
	<text x="126.5" y="17.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="165.437,62.028 172.794,56.122 163.353,52.827"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 253.39,70.35 A 22.5,22.5 0 1 1 278.447,61.879"/>
	<text x="237.5" y="18.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="278.447,61.879 285.602,55.73 276.056,52.753"/>
	<polygon stroke="black" stroke-width="1" points="274.5,121.5 274.5,199.5"/>
	<polygon fill="black" stroke-width="1" points="274.5,199.5 279.5,191.5 269.5,191.5"/>
	<text x="256.5" y="166.5" font-family="Times New Roman" font-size="20">C</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 296.812,249.379 A 22.5,22.5 0 1 1 272.293,259.301"/>
	<text x="304.5" y="313.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="272.293,259.301 265.51,265.858 275.214,268.272"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 481.364,351.116 A 345.542,345.542 0 0 1 280.903,120.799"/>
	<polygon fill="black" stroke-width="1" points="481.364,351.116 475.92,343.412 472.053,352.634"/>
	<text x="335.5" y="279.5" font-family="Times New Roman" font-size="20">R</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 368.167,70.574 A 22.5,22.5 0 1 1 393.133,61.839"/>
	<text x="352.5" y="18.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="393.133,61.839 400.223,55.615 390.645,52.739"/>
	<polygon stroke="black" stroke-width="1" points="389.5,121.5 389.5,199.5"/>
	<polygon fill="black" stroke-width="1" points="389.5,199.5 394.5,191.5 384.5,191.5"/>
	<text x="371.5" y="166.5" font-family="Times New Roman" font-size="20">C</text>
	<polygon stroke="black" stroke-width="1" points="419.5,91.5 479.5,91.5"/>
	<polygon fill="black" stroke-width="1" points="479.5,91.5 471.5,86.5 471.5,96.5"/>
	<text x="443.5" y="82.5" font-family="Times New Roman" font-size="20">S</text>
	<polygon stroke="black" stroke-width="1" points="509.5,121.5 509.5,199.5"/>
	<polygon fill="black" stroke-width="1" points="509.5,199.5 514.5,191.5 504.5,191.5"/>
	<text x="491.5" y="166.5" font-family="Times New Roman" font-size="20">C</text>
	<polygon stroke="black" stroke-width="1" points="419.5,229.5 479.5,229.5"/>
	<polygon fill="black" stroke-width="1" points="479.5,229.5 471.5,224.5 471.5,234.5"/>
	<text x="443.5" y="220.5" font-family="Times New Roman" font-size="20">S</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 487.884,70.866 A 22.5,22.5 0 1 1 512.73,61.792"/>
	<text x="470.5" y="18.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="512.73,61.792 519.734,55.472 510.118,52.727"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 414.474,245.91 A 22.5,22.5 0 1 1 391.666,259.304"/>
	<text x="428.5" y="308.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="391.666,259.304 385.913,266.781 395.865,267.752"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 532.594,248.465 A 22.5,22.5 0 1 1 508.494,259.366"/>
	<text x="542.5" y="312.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="508.494,259.366 501.98,266.19 511.774,268.212"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 606.457,70.283 A 22.5,22.5 0 1 1 631.541,61.891"/>
	<text x="591.5" y="18.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="631.541,61.891 638.715,55.766 629.178,52.758"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 649.796,249.397 A 22.5,22.5 0 1 1 625.269,259.3"/>
	<text x="657.5" y="313.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="625.269,259.3 618.481,265.851 628.183,268.272"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 759.398,248.701 A 22.5,22.5 0 1 1 735.188,259.354"/>
	<text x="768.5" y="312.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="735.188,259.354 728.604,266.111 738.377,268.233"/>
	<polygon stroke="black" stroke-width="1" points="627.5,121.5 627.5,199.5"/>
	<polygon fill="black" stroke-width="1" points="627.5,199.5 632.5,191.5 622.5,191.5"/>
	<text x="609.5" y="166.5" font-family="Times New Roman" font-size="20">C</text>
	<polygon stroke="black" stroke-width="1" points="736.5,121.5 736.5,199.5"/>
	<polygon fill="black" stroke-width="1" points="736.5,199.5 741.5,191.5 731.5,191.5"/>
	<text x="718.5" y="166.5" font-family="Times New Roman" font-size="20">C</text>
	<polygon stroke="black" stroke-width="1" points="657.5,91.5 706.5,91.5"/>
	<polygon fill="black" stroke-width="1" points="706.5,91.5 698.5,86.5 698.5,96.5"/>
	<text x="676.5" y="82.5" font-family="Times New Roman" font-size="20">S</text>
	<polygon stroke="black" stroke-width="1" points="657.5,229.5 706.5,229.5"/>
	<polygon fill="black" stroke-width="1" points="706.5,229.5 698.5,224.5 698.5,234.5"/>
	<text x="676.5" y="220.5" font-family="Times New Roman" font-size="20">S</text>
	<polygon stroke="black" stroke-width="1" points="76.5,361.5 139.5,361.5"/>
	<polygon fill="black" stroke-width="1" points="139.5,361.5 131.5,356.5 131.5,366.5"/>
	<text x="102.5" y="352.5" font-family="Times New Roman" font-size="20">P</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 68.463,381.763 A 22.5,22.5 0 1 1 43.776,391.259"/>
	<text x="75.5" y="445.5" font-family="Times New Roman" font-size="20">C</text>
	<polygon fill="black" stroke-width="1" points="43.776,391.259 36.88,397.696 46.541,400.278"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 189.625,383.591 A 22.5,22.5 0 1 1 164.207,390.91"/>
	<text x="189.5" y="448.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="164.207,390.91 156.779,396.726 166.18,400.136"/>
	<polygon stroke="black" stroke-width="1" points="199.5,361.5 251.5,361.5"/>
	<polygon fill="black" stroke-width="1" points="251.5,361.5 243.5,356.5 243.5,366.5"/>
	<text x="219.5" y="352.5" font-family="Times New Roman" font-size="20">S</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 195.596,346.76 A 197.701,197.701 0 0 1 363.404,346.76"/>
	<polygon fill="black" stroke-width="1" points="363.404,346.76 358.282,338.837 354.038,347.892"/>
	<text x="272.5" y="319.5" font-family="Times New Roman" font-size="20">R</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 301.136,384.026 A 22.5,22.5 0 1 1 275.564,390.788"/>
	<text x="298.5" y="449.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="275.564,390.788 268.011,396.439 277.335,400.054"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 491.494,385.415 A 132.279,132.279 0 0 1 299.506,385.415"/>
	<polygon fill="black" stroke-width="1" points="491.494,385.415 482.362,387.781 489.619,394.661"/>
	<text x="388.5" y="447.5" font-family="Times New Roman" font-size="20">R</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 412.958,380.012 A 22.5,22.5 0 1 1 389.075,391.38"/>
	<text x="423.5" y="443.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="389.075,391.38 382.695,398.33 392.526,400.16"/>
	<polygon stroke="black" stroke-width="1" points="419.5,361.5 479.5,361.5"/>
	<polygon fill="black" stroke-width="1" points="479.5,361.5 471.5,356.5 471.5,366.5"/>
	<text x="443.5" y="352.5" font-family="Times New Roman" font-size="20">S</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 488.111,340.632 A 22.5,22.5 0 1 1 513.053,331.829"/>
	<text x="451.5" y="288.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="513.053,331.829 520.126,325.586 510.541,322.736"/>
	<polygon stroke="black" stroke-width="1" points="415.243,376.904 607.757,492.096"/>
	<polygon fill="black" stroke-width="1" points="607.757,492.096 603.459,483.698 598.324,492.279"/>
	<text x="516.5" y="425.5" font-family="Times New Roman" font-size="20">T</text>
	<polygon stroke="black" stroke-width="1" points="534.732,377.728 711.268,491.272"/>
	<polygon fill="black" stroke-width="1" points="711.268,491.272 707.245,482.739 701.835,491.149"/>
	<text x="628.5" y="425.5" font-family="Times New Roman" font-size="20">T</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 606.235,340.505 A 22.5,22.5 0 1 1 631.229,331.851"/>
	<text x="570.5" y="288.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="631.229,331.851 638.339,325.65 628.771,322.743"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 715.007,340.739 A 22.5,22.5 0 1 1 739.905,331.812"/>
	<text x="678.5" y="288.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="739.905,331.812 746.947,325.534 737.348,322.731"/>
	<polygon stroke="black" stroke-width="1" points="657.5,361.5 706.5,361.5"/>
	<polygon fill="black" stroke-width="1" points="706.5,361.5 698.5,356.5 698.5,366.5"/>
	<text x="676.5" y="352.5" font-family="Times New Roman" font-size="20">S</text>
	<polygon stroke="black" stroke-width="1" points="76.5,507.5 139.5,507.5"/>
	<polygon fill="black" stroke-width="1" points="139.5,507.5 131.5,502.5 131.5,512.5"/>
	<text x="102.5" y="498.5" font-family="Times New Roman" font-size="20">P</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 193.684,525.054 A 22.5,22.5 0 1 1 170.277,537.373"/>
	<text x="205.5" y="588.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="170.277,537.373 164.182,544.573 174.079,546.007"/>
	<polygon stroke="black" stroke-width="1" points="199.5,507.5 251.5,507.5"/>
	<polygon fill="black" stroke-width="1" points="251.5,507.5 243.5,502.5 243.5,512.5"/>
	<text x="219.5" y="498.5" font-family="Times New Roman" font-size="20">S</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 194.063,490.343 A 171.3,171.3 0 0 1 364.937,490.343"/>
	<polygon fill="black" stroke-width="1" points="364.937,490.343 360.497,482.019 355.51,490.686"/>
	<text x="272.5" y="458.5" font-family="Times New Roman" font-size="20">R</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 306.942,523.175 A 22.5,22.5 0 1 1 284.534,537.229"/>
	<text x="322.5" y="585.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="284.534,537.229 279.001,544.869 288.977,545.55"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 307.862,493.233 A 212.35,212.35 0 0 1 483.138,493.233"/>
	<polygon fill="black" stroke-width="1" points="483.138,493.233 477.915,485.377 473.788,494.486"/>
	<text x="388.5" y="465.5" font-family="Times New Roman" font-size="20">R</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 416.128,521.062 A 22.5,22.5 0 1 1 394.933,536.885"/>
	<text x="434.5" y="582.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="394.933,536.885 390.037,544.949 400.036,544.819"/>
	<polygon stroke="black" stroke-width="1" points="419.5,507.5 479.5,507.5"/>
	<polygon fill="black" stroke-width="1" points="479.5,507.5 471.5,502.5 471.5,512.5"/>
	<text x="443.5" y="498.5" font-family="Times New Roman" font-size="20">S</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 536.133,521.053 A 22.5,22.5 0 1 1 514.942,536.883"/>
	<text x="554.5" y="582.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="514.942,536.883 510.049,544.949 520.048,544.816"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 657.621,525.14 A 22.5,22.5 0 1 1 634.17,537.375"/>
	<text x="669.5" y="588.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="634.17,537.375 628.049,544.554 637.941,546.023"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 758.204,528.041 A 22.5,22.5 0 1 1 733.398,537.222"/>
	<text x="763.5" y="592.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="733.398,537.222 726.421,543.571 736.048,546.276"/>
	<polygon stroke="black" stroke-width="1" points="663.5,507.5 706.5,507.5"/>
	<polygon fill="black" stroke-width="1" points="706.5,507.5 698.5,502.5 698.5,512.5"/>
	<text x="679.5" y="498.5" font-family="Times New Roman" font-size="20">S</text>
</svg>
<p class="text-center"><small>Figure 3.5: The product of the bank's and the store's state machines</small></p>

We've drawn all the state pairs and the transitions among them. We can now clearly see how the inputs affect the system as a whole. For instance, on _pay_, the store goes from `a` to `b`, whereas the bank stays put, therefore, `change_state((1, a), "pay") -> (1, b)`. From state `(1, b)`, on _redeem_, the store goes from state `b` to `d` and the bank from `1` to `3`, therefore, we end up in `(3, d)`.

### Eliminating the Unreachable States

We observe that not all the states are reachable from the initial state `(1, a)`. For example, `(1, g)`, where the store has shipped the goods after the money is transferred, but the bank is still waiting for a redeem. We've modelled our system such that a case like this cannot occur. After discarding the unreachable states and all of their outgoing transitions, we end up with a much clearer picture, that is, a state machine which is easy to understand yet encodes complex business logic.

<svg width="800" height="320" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="none" cx="46.5" cy="91.5" rx="30" ry="30"/>
	<text x="32.5" y="97.5" font-family="Times New Roman" font-size="20">1, a</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="160.5" cy="91.5" rx="30" ry="30"/>
	<text x="145.5" y="97.5" font-family="Times New Roman" font-size="20">1, b</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="294.5" cy="91.5" rx="30" ry="30"/>
	<text x="280.5" y="97.5" font-family="Times New Roman" font-size="20">1, c</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="46.5" cy="229.5" rx="30" ry="30"/>
	<text x="32.5" y="235.5" font-family="Times New Roman" font-size="20">2, a</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="160.5" cy="229.5" rx="30" ry="30"/>
	<text x="145.5" y="235.5" font-family="Times New Roman" font-size="20">2, b</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="294.5" cy="229.5" rx="30" ry="30"/>
	<text x="280.5" y="235.5" font-family="Times New Roman" font-size="20">2, c</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="471.5" cy="229.5" rx="30" ry="30"/>
	<text x="456.5" y="235.5" font-family="Times New Roman" font-size="20">3, d</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="471.5" cy="91.5" rx="30" ry="30"/>
	<text x="457.5" y="97.5" font-family="Times New Roman" font-size="20">3, e</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="612.5" cy="229.5" rx="30" ry="30"/>
	<text x="599.5" y="235.5" font-family="Times New Roman" font-size="20">4, f</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="612.5" cy="91.5" rx="30" ry="30"/>
	<text x="597.5" y="97.5" font-family="Times New Roman" font-size="20">4, g</text>
	<polygon stroke="black" stroke-width="1" points="187.922,103.668 444.078,217.332"/>
	<polygon fill="black" stroke-width="1" points="444.078,217.332 438.794,209.517 434.738,218.658"/>
	<text x="320.5" y="151.5" font-family="Times New Roman" font-size="20">R</text>
	<polygon stroke="black" stroke-width="1" points="76.5,91.5 130.5,91.5"/>
	<polygon fill="black" stroke-width="1" points="130.5,91.5 122.5,86.5 122.5,96.5"/>
	<text x="97.5" y="82.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon stroke="black" stroke-width="1" points="46.5,121.5 46.5,199.5"/>
	<polygon fill="black" stroke-width="1" points="46.5,199.5 51.5,191.5 41.5,191.5"/>
	<text x="28.5" y="166.5" font-family="Times New Roman" font-size="20">C</text>
	<polygon stroke="black" stroke-width="1" points="160.5,121.5 160.5,199.5"/>
	<polygon fill="black" stroke-width="1" points="160.5,199.5 165.5,191.5 155.5,191.5"/>
	<text x="142.5" y="166.5" font-family="Times New Roman" font-size="20">C</text>
	<polygon stroke="black" stroke-width="1" points="76.5,229.5 130.5,229.5"/>
	<polygon fill="black" stroke-width="1" points="130.5,229.5 122.5,224.5 122.5,234.5"/>
	<text x="97.5" y="220.5" font-family="Times New Roman" font-size="20">P</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 180.588,251.624 A 22.5,22.5 0 1 1 155.158,258.902"/>
	<text x="182.5" y="315.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="155.158,258.902 147.721,264.705 157.116,268.13"/>
	<polygon stroke="black" stroke-width="1" points="190.5,229.5 264.5,229.5"/>
	<polygon fill="black" stroke-width="1" points="264.5,229.5 256.5,224.5 256.5,234.5"/>
	<text x="221.5" y="220.5" font-family="Times New Roman" font-size="20">S</text>
	<polygon stroke="black" stroke-width="1" points="190.5,91.5 264.5,91.5"/>
	<polygon fill="black" stroke-width="1" points="264.5,91.5 256.5,86.5 256.5,96.5"/>
	<text x="221.5" y="82.5" font-family="Times New Roman" font-size="20">S</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 140.11,69.654 A 22.5,22.5 0 1 1 165.437,62.028"/>
	<text x="126.5" y="17.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="165.437,62.028 172.794,56.122 163.353,52.827"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 273.39,70.35 A 22.5,22.5 0 1 1 298.447,61.879"/>
	<text x="257.5" y="18.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="298.447,61.879 305.602,55.73 296.056,52.753"/>
	<polygon stroke="black" stroke-width="1" points="294.5,121.5 294.5,199.5"/>
	<polygon fill="black" stroke-width="1" points="294.5,199.5 299.5,191.5 289.5,191.5"/>
	<text x="276.5" y="166.5" font-family="Times New Roman" font-size="20">C</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 316.812,249.379 A 22.5,22.5 0 1 1 292.293,259.301"/>
	<text x="324.5" y="313.5" font-family="Times New Roman" font-size="20">P</text>
	<polygon fill="black" stroke-width="1" points="292.293,259.301 285.51,265.858 295.214,268.272"/>
	<polygon stroke="black" stroke-width="1" points="324.5,91.5 441.5,91.5"/>
	<polygon fill="black" stroke-width="1" points="441.5,91.5 433.5,86.5 433.5,96.5"/>
	<text x="376.5" y="112.5" font-family="Times New Roman" font-size="20">R</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 494.958,248.012 A 22.5,22.5 0 1 1 471.075,259.38"/>
	<text x="505.5" y="311.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="471.075,259.38 464.695,266.33 474.526,268.16"/>
	<polygon stroke="black" stroke-width="1" points="471.5,199.5 471.5,121.5"/>
	<polygon fill="black" stroke-width="1" points="471.5,121.5 466.5,129.5 476.5,129.5"/>
	<text x="455.5" y="166.5" font-family="Times New Roman" font-size="20">S</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 450.111,70.632 A 22.5,22.5 0 1 1 475.053,61.829"/>
	<text x="413.5" y="18.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="475.053,61.829 482.126,55.586 472.541,52.736"/>
	<polygon stroke="black" stroke-width="1" points="501.5,229.5 582.5,229.5"/>
	<polygon fill="black" stroke-width="1" points="582.5,229.5 574.5,224.5 574.5,234.5"/>
	<text x="535.5" y="220.5" font-family="Times New Roman" font-size="20">T</text>
	<polygon stroke="black" stroke-width="1" points="501.5,91.5 582.5,91.5"/>
	<polygon fill="black" stroke-width="1" points="582.5,91.5 574.5,86.5 574.5,96.5"/>
	<text x="535.5" y="82.5" font-family="Times New Roman" font-size="20">T</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 640.687,219.577 A 22.5,22.5 0 1 1 637.521,245.838"/>
	<text x="686.5" y="244.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="637.521,245.838 639.982,254.945 646.786,247.616"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 640.588,81.298 A 22.5,22.5 0 1 1 637.682,107.588"/>
	<text x="686.5" y="105.5" font-family="Times New Roman" font-size="20">P, C</text>
	<polygon fill="black" stroke-width="1" points="637.682,107.588 640.233,116.671 646.964,109.275"/>
	<polygon stroke="black" stroke-width="1" points="612.5,199.5 612.5,121.5"/>
	<polygon fill="black" stroke-width="1" points="612.5,121.5 607.5,129.5 617.5,129.5"/>
	<text x="596.5" y="166.5" font-family="Times New Roman" font-size="20">S</text>
</svg>
<p class="text-center"><small>Figure 3.6: The combined state machine for the bank and the store</small></p>

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
