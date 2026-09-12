---
title: "Couple the Experience, Decouple the Systems"
date: 2026-07-25T14:00:00+03:00
draft: true
summary: "How a business process that spans many systems and roles becomes an integration topology — seamless for the user, independent underneath."
tags: ["software-architecture", "domain-driven-design", "distributed-systems", "enterprise-integration"]
editLink: "https://github.com/deniskyashif/blog/blob/main/content/posts/ddd-integration-topology.md"
---

This is the third part of a series. [Modeling Facts and Reactions with Domain Events](/2026/07/25/modeling-facts-and-reactions-with-domain-events/) covered a fact and its consequences inside a bounded context. [Crossing Boundaries with Integration Events](/2026/08/29/crossing-boundaries-with-integration-events/) covered one fact crossing one boundary reliably. Here we zoom out: many systems, one business process.

## A process that spans systems

- Open with a process everyone recognizes: order → fulfillment → invoice.
- The user (or a chain of users across roles) experiences one seamless journey; no single system owns the whole thing.
- Establish the altitude change from the previous article: not one boundary, but the shape of many.
- State the thesis early:

> Couple the experience, decouple the systems.

## The real world is asynchronous and unreliable

- The philosophical anchor of the article. A coffee shop does not use two-phase commit: it takes your order, writes your name on a cup, takes payment, and baristas work concurrently. When something goes wrong it retries (remakes the drink), compensates (refunds), or writes it off (tosses the coffee). The business stays available and correct *without* atomic coordination.
- Business processes have always worked this way: purchase orders, shipment notices, invoices — documents passed between parties who each work at their own pace and reconcile later. Asynchrony and compensation are not a technical compromise; they are how the domain already operates.
- The artificial construct is the synchronous one. RPC makes remote communication look like local communication — that is not an abstraction, it is an illusion. The call stack shapes our design intuition, which is why async systems feel harder to design: we're unlearning the illusion, not adding complexity.
- What messaging buys, stated as consequences of accepting reality: temporal decoupling, failure containment, consumers working at their own pace, throughput over latency ("wider bridges instead of faster cars").
- The catch that drives the rest of the article: the *systems* can embrace asynchrony, but the *user* still expects one coherent journey. That tension is the subject.

## Two forces in tension

- Seamlessness pulls toward coupling: shared state, synchronous calls, one process that knows everything. Easy UX, brittle architecture.
- Autonomy pulls toward decoupling: async events, duplicated read models, eventual consistency. Robust architecture, but seams can leak into the experience.
- Name the two failure modes:
  - Couple the systems to make the UX seamless → distributed monolith.
  - Let the decoupling leak into the UX → the user feels the seams (waiting on handoffs, stale data, "please refresh").
- The craft is placing the tension deliberately, not resolving it accidentally.
- **Less integration is better design:** every seam is a contract, a failure mode, and a coordination cost. The fewer boundaries a process crosses, the simpler it is to build, reason about, and operate.
- **But integration can't always be avoided:** organizations have distinct systems, teams, and lifecycles for good reasons. When a process genuinely spans them, the goal shifts from *eliminating* integration to making it explicit, reliable, and invisible to the user.

## Slice vertically before integrating

- **A bounded context should form a vertical slice.** It owns one business capability end-to-end — its data, decisions, and the relevant slice of the UI. Slice along business capabilities, not horizontally into a shared UI shell, services tier, and database.
- Vertical slicing is the design ideal because cohesive work remains inside one context. If a whole process genuinely fit one slice, there would be no integration seam.
- The business often divides into distinct capabilities — ordering, fulfillment, invoicing — with different language, rules, data, ownership, and rates of change. Slice as vertically as those capabilities genuinely divide, then integrate only across the seams left behind.
- **Common pitfall:** treating familiar technical types such as money, identifiers, or addresses as an automatic shared kernel. Their meaning and rules often differ by context. A shared kernel is a small, deliberately jointly owned model with an explicit coordination cost, not merely a library of common-looking types.

## The seams are the handoffs

- A flow that crosses an accepted context boundary is where integration lives. **Accepted business boundaries should become explicit architectural seams** — do not create integration merely because two people exchange work; do not hide integration when two autonomous capabilities exchange responsibility.
- Classify every cross-boundary interaction *before* choosing technology: is it a fact others may react to, an instruction to one owner, a request for information, a reply, or a human handoff? That determines whether it becomes an integration event, command, query, response, or workflow task.
- Connect back to the previous article: an integration event fits when the producer can finish without an answer; a command or query routed through a broker is still request–response in disguise.
- (Keep this section slim — how to *discover* boundaries via process modeling deserves its own article; here we take the boundaries as given and design what flows across them.)

## Who owns the end-to-end flow?

- With the handoffs classified, the process itself needs an owner. Two answers:
- **Choreography:** contexts react to each other's events, no central coordinator. The process is implicit — an emergent property of local reactions. Decoupled, but hard to see, monitor, and reason about as a whole.
- **Orchestration:** a coordinator drives the steps and holds process state. The flow is explicit and owned.
- The orchestrating coordinator *is* a **saga / process manager** — name it here.
- Echo the real-world anchor: the saga is the barista calling out a remake or issuing a refund — compensation is a business behavior, not a transport feature. Message retry redelivers one hop; it cannot *compensate* a completed business step. **Delivery is not coordination.**
- The trap: putting the process logic in the messaging infrastructure ("the hub runs the process") — that's the orchestration-engine ESB again. A saga makes decisions, so it is a participant, not plumbing.
- In a choreographed saga there is still no coordinator — the process logic (including compensation) is distributed across the participants, and the infrastructure only plumbs and observes it.
- Defer saga mechanics (compensation, timeouts, state machines) to the continuation; here we only answer *who owns the flow*.

## Choose the integration topology

- Derive the topology only after identifying boundaries, classifying handoffs, and deciding whether coordination is implicit or explicit. Do not begin with a broker or integration pattern and force the process into it.
- A sparse set of direct APIs, producer-owned topics on shared messaging infrastructure, or decentralized pub/sub may be enough. Pairwise integration becomes problematic when contracts, routing rules, and operational concerns grow into an unmanaged N×N mesh — not merely because systems communicate directly.
- Introduce hub-and-spoke when many autonomous contexts need shared plumbing: spokes produce events, expose APIs, and consume events; the hub provides routing, schema storage and compatibility enforcement, broker capabilities, observability, replay tooling, and dead-letter handling.
- **Does the hub own a data model?** Distinguish two kinds of hub:
  - Plumbing + governance hub: stores contracts and routing/delivery metadata — *how* systems integrate. Semantic contract ownership and business-data ownership stay in the spokes.
  - Canonical-data-model hub: owns a shared enterprise model everything maps to. Warn that this recentralizes coupling (the ESB trap), becomes a bottleneck and lowest-common-denominator, and violates "notification does not transfer ownership."
- **The litmus test:** *does it make a decision, or does it move a message?* Decisions belong in a spoke; moving messages reliably belongs in the hub. When the hub accumulates logic or becomes a source of truth, you've built a distributed monolith around a chokepoint.
- Note that reliability remains end-to-end: transactional publication and idempotent consumption from the previous article are still required regardless of topology.
- One-liner on endpoint frameworks: libraries such as NServiceBus and MassTransit package the outbox, recoverability, and saga patterns *inside each endpoint* — the modern answer to the ESB: same reliability patterns, but logic and ownership pushed to the edges.
- Close the section with a knowability note: explicit integration must stay *visible* — versioned contracts with named owners, and correlation IDs / tracing so one business process can be followed across contexts. A decoupled system nobody can see as a whole is just an undocumented distributed monolith. (Full governance treatment deferred to its own article.)

## Keeping the experience seamless over decoupled systems

- Return to the opening tension: the systems are now honestly asynchronous — how does the user still get one journey?
- The real world answers this too: businesses give you a *status*, not a lock. "Your order has shipped", a tracking number, an estimated delivery date. Nobody holds a transaction open across the warehouse.
- A unified experience does not require one unified runtime or database. It requires one coherent account of the journey, assembled from state owned by autonomous capabilities.
- **Give the journey a durable status model:** expose business milestones such as `OrderReceived`, `Preparing`, `Shipped`, `Invoiced`, not transport details such as "message queued." A process manager may own this view in an orchestrated flow; a dedicated experience/read-model context may project it from events in a choreography. Neither becomes authoritative for the participants' domain data.
- **Use event-carried state transfer and local read models:** the experience renders without synchronous callbacks to every participating system. This deliberately duplicates the minimum data needed for the view while ownership stays with the source context.
- **Make eventual consistency honest:** represent pending, delayed, and failed transitions explicitly. "We've received your order," progress indicators, timestamps, and actionable retry/support paths make the handoff understandable instead of pretending it is instantaneous.
- **Design recovery as part of the journey:** after refresh, reconnect, or failover, the user must be able to read the current durable status. Support staff need the same correlation and process history to explain where a journey is blocked.
- **Push updates are an optimization:** WebSockets/SSE can notify a connected client when the durable read model changes, but clients disconnect and notifications get missed. On reconnect, reload the authoritative status; client push never replaces the durable status model.
- This is the intended coupling: screens, language, and progress form one experience, while each capability retains its own logic, data, deployment, and failure lifecycle underneath.

## Closing

- The goal is not to eliminate the tension but to place it deliberately: coupling in the experience, decoupling in the systems, and a clear owner for the process between them.
- Return once more to the anchor: the real world is asynchronous and unreliable — and businesses have thrived in it for centuries by working concurrently, reconciling later, and compensating when things go wrong. Our systems can do the same; only the user's view of the journey needs to feel whole.
- Recap the trilogy:
  - domain events — a fact and its consequences inside a context,
  - integration events — one fact crossing one boundary, reliably,
  - integration topology — many systems, one experience.
- Forward-point to the continuation: how the process manager actually works — driving a multi-step process to a consistent end through compensation, across decoupled and eventually consistent systems.

## Further Reading (candidates)

- [Your Coffee Shop Doesn't Use Two-Phase Commit](https://martinfowler.com/ieeeSoftware/coffeeShop.pdf) by Gregor Hohpe
- _Enterprise Integration Patterns_ by Gregor Hohpe and Bobby Woolf — messaging and conversation patterns
- [What do you mean by "Event-Driven"?](https://martinfowler.com/articles/201701-event-driven.html) by Martin Fowler

# Random notes that might be useful for the article

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
