---
title: "System Integration Patterns"
date: 2025-05-31T11:05:30+03:00
draft: true
summary: "Exploring system integration patterns, avoiding common pitfalls, and best practices when building loosely coupled systems."
tags: ["integration", "software-design"]
---

Hi, this is my first post in a while. We'll explore several integration scenarios today.


## RPC

Make remote communication look like local communication. It is not an abstraction - it is actually an illusion.

## EIP 2

### Messaging

What? Why? Channel?
Asynchronous messaging style
+ Temporal decoupling - sender does not have to wait for receiver to process message
+ Limits Failure propagation
+ Throttling - conumer does in its own pace
+ Insertion of intermediaries - pipes and filters - composability, transformation, routing, etc.
+ Throughput over latency - wider bridges instead of faster cars

real world is asynchronous and unreliable
td: two phased commit; Starbucks doesn't use it

Asynchronous Architectural Style - vs Synchronous
Async Events vs Call Stack
it is difficult to transfer our design thinking which is sync (call stack) to asynchronous
async systems are harder to design

Messaging systems - transport, design, routing, transform, producers and consumers, monitoring

pattern language - used by enterprise ESBs

Competing Consumer (p2p), Pub Sub pattern
Kafka combines the two

### Conversation patterns
Focus on participants, follows time, multi-way, stateful
Rules of the conversation
all about state
Sub Pub - subscribe-notify
you subscribe one, but get many messages

Pattern Language
Design problem -> Forces -> Solution -> Related Patterns
Book: A Pattern Language - Towns, Buildings, Construction - Christopher Alexander
