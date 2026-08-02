---
title: "Couple the Experience, Decouple the Systems"
date: 2026-07-25T14:00:00+03:00
draft: true
summary: "How a business process that spans many systems and roles becomes an integration topology — seamless for the user, independent underneath."
tags: ["software-architecture", "domain-driven-design", "distributed-systems", "enterprise-integration"]
editLink: "https://github.com/deniskyashif/blog/blob/main/content/posts/ddd-integration-topology.md"
---

This is the third part of a series. [Modeling Facts and Reactions with Domain Events](/posts/2026-07-25-ddd-domain-events/) covered facts and consequences inside a bounded context. [Crossing Boundaries with Integration Events](/posts/ddd-integration-events/) covered one fact crossing one boundary reliably. Here we zoom out to many systems and one business process.

A business process rarely lives in a single system. The user experiences one seamless journey; the architecture wants many independent systems. This article is about resolving that tension deliberately:

> Couple the experience, decouple the systems.

## A process that spans systems

- Start from a business process everyone recognizes: order → fulfillment → invoice.
- **Model the process first — this is step zero.** Before choosing system boundaries, model the end-to-end process collaboratively with domain experts. The process model supplies evidence for the topology; it does not mechanically determine it. Skip this step and architecture follows an implicit, unshared understanding of the business.
- Start with participants and roles in a BPMN diagram, without assuming that every pool, lane, team, or human handoff must become a bounded context or deployed system. They are **boundary hypotheses** to test against changes in language, decision ownership, invariants, data ownership, and change cadence.
- The model is a shared, agreed artifact — not one architect's sketch. It makes both the user journey and the organizational handoffs visible before either calcifies into architecture.
- The user (or a chain of users across roles) experiences one journey; no single system owns the whole thing.
- Establish the altitude change from the previous article: not one boundary, but the shape of many.
- This sets up the throughline: **model the process → test candidate boundaries → classify the handoffs → derive the topology → govern it → reseal the UX.**

## Two forces in tension

- Seamlessness pulls toward coupling: shared state, synchronous calls, one process that knows everything. Easy UX, brittle architecture.
- Autonomy pulls toward decoupling: async events, duplicated read models, eventual consistency. Robust architecture, but seams can leak into the experience.
- Name the two failure modes:
  - Couple the systems to make the UX seamless → distributed monolith.
  - Let the decoupling leak into the UX → the user feels the seams (waiting on handoffs, stale data, "please refresh").
- The craft is placing the tension deliberately, not resolving it accidentally.
- **Less integration is better design:** every seam is a contract, a failure mode, and a coordination cost. The fewer boundaries a process crosses, the simpler it is to build, reason about, and operate. Prefer keeping cohesive work inside one context.
- **But integration can't always be avoided:** organizations have distinct systems, teams, and lifecycles for good reasons. When a process genuinely spans them, the goal shifts from *eliminating* integration to making it explicit, reliable, and invisible to the user.

## Slice vertically before integrating

- **A bounded context should form a vertical slice.** It owns one business capability end-to-end — its data, decisions, and the relevant slice of the UI. Slice along business capabilities, not horizontally into a shared UI shell, services tier, and database.
- Vertical slicing is the design ideal because cohesive work remains inside one context. If a whole process genuinely fit one slice, there would be no integration seam.
- The business often divides into distinct capabilities — ordering, fulfillment, invoicing — with different language, rules, data, ownership, and rates of change. Slice as vertically as those capabilities genuinely divide, then integrate only across the seams left behind.
- **Common pitfall:** treating familiar technical types such as money, identifiers, or addresses as an automatic shared kernel. Their meaning and rules often differ by context. A shared kernel is a small, deliberately jointly owned model with an explicit coordination cost, not merely a library of common-looking types.

## The seams are the handoffs

- Once candidate boundaries have been tested, a flow that crosses an accepted boundary is where integration lives.
- Use the following as discovery heuristics, not mechanical translations:

```text
BPMN concept                    Possible architectural implication
-----------------------------   ----------------------------------
Pool / participant           -> Candidate ownership or context boundary
Lane                         -> Role or responsibility boundary
Activity / task              -> Behavior owned by a capability
Sequence flow                -> Local control flow; sometimes a domain event
Message flow                 -> Event, command, query, reply, or human handoff
Process spanning pools       -> Cross-capability flow requiring visibility
```

- The principle becomes: **accepted business boundaries should become explicit architectural seams.** Do not create integration merely because two people exchange work; do not hide integration when two autonomous capabilities exchange responsibility.
- Classify every accepted cross-boundary interaction before choosing technology: is it a fact others may react to, an instruction to one owner, a request for information, a reply, or a human handoff? That determines whether it becomes an integration event, command, query, response, or workflow task.
- **Conway's Law is a warning and a design tool, not destiny:** systems tend to mirror organizational communication structures, so pools and lanes expose forces that may shape the architecture. But an organizational handoff does not automatically require a software boundary. With the inverse Conway maneuver, teams and communication paths can also be shaped around the architecture and business capabilities we want.
- BPMN message flows are therefore a top-down discovery tool for candidate integrations, to be validated through domain modeling rather than copied directly into code.

## Who owns the end-to-end flow?

- **Choreography:** spokes react to each other's events, no central coordinator. The process is implicit — an emergent property of local reactions. Decoupled, but hard to see, monitor, and reason about as a whole.
- **Orchestration:** a coordinator drives the steps and holds process state. The flow is explicit and owned.
- The orchestrating coordinator *is* a **saga / process manager** — name it here.
- The trap: putting the process logic in the hub (the hub "runs the process") — that's orchestration-engine ESB again. Apply the litmus test: a saga makes decisions, so it is a spoke, not the hub.
- Cleaner answer: when the cross-system flow must be explicit, it belongs in a dedicated **process-manager spoke**, not the hub. The process is coupled (owned in one place); the participants stay decoupled.
- **Delivery is not coordination:** the hub logging actions, providing observability, and retrying messages does not make it a saga. Message retry redelivers one hop; it cannot *compensate* a completed business step. A saga owns the whole process and its consistent end state.
- In a choreographed saga there is still no coordinator — the process logic (including compensation) is distributed across the spokes, and the hub only plumbs and observes it.
- Defer saga mechanics (compensation, timeouts, state machines) to the continuation; here we only answer *who owns the flow*.

## Choose the integration topology

- Derive the topology only after identifying boundaries, classifying handoffs, and deciding whether coordination is implicit or explicit. Do not begin with a broker or integration pattern and force the process into it.
- A sparse set of direct APIs, producer-owned topics on shared messaging infrastructure, or decentralized pub/sub may be enough. Pairwise integration becomes problematic when contracts, routing rules, and operational concerns grow into an unmanaged N×N mesh — not merely because systems communicate directly.
- Introduce hub-and-spoke when many autonomous contexts need shared plumbing and consistent governance: spokes produce events, expose APIs, and consume events; the hub provides routing, schema storage and compatibility enforcement, broker capabilities, observability, replay tooling, and dead-letter handling.
- **Does the hub own a data model?** Distinguish two kinds of hub:
  - Plumbing + governance hub: stores contracts and routing/delivery metadata — *how* systems integrate. Semantic contract ownership and business-data ownership stay in the spokes.
  - Canonical-data-model hub: owns a shared enterprise model everything maps to. Warn that this recentralizes coupling (the ESB trap), becomes a bottleneck and lowest-common-denominator, and violates "notification does not transfer ownership."
- The hub is infrastructure, not a place for business logic. When it accumulates logic or becomes a source of truth, you've built a distributed monolith around a chokepoint.
- **The litmus test:** *does it make a decision, or does it move a message?* Decisions belong in a spoke; moving messages reliably belongs in the hub. The infrastructure can acknowledge, retry, retain, and route messages, but reliability remains end-to-end: transactional publication and idempotent consumption from the previous article are still required.

## Governing and documenting the topology

- Once integration is explicit, it must stay *knowable*. A decoupled system that nobody can see as a whole is just an undocumented distributed monolith.
- **Contracts as the source of truth:** event and command schemas are the public contract between spokes. Version them, register them (schema registry), and enforce compatibility so a producer can't silently break consumers.
- **Ownership is explicit:** every event, API, and process-manager spoke has a named owning team. The registry stores and validates a contract; the producing spoke (or another explicitly designated contract owner) owns its meaning and evolution. "Notification does not transfer ownership" only holds if ownership is written down.
- **A living map:** maintain a topology view — which spokes exist, what events they publish and consume, and which process managers coordinate them. Generate it from the contracts/registry where possible so it can't drift from reality.
- **Connect process and architecture documentation:** link each accepted cross-boundary BPMN flow to its implemented event, command, query, or human task. Keep them in sync; an unexplained discrepancy is a design smell, not just stale docs.
- **Governance is enablement, not gatekeeping:** shared infrastructure enforces schema compatibility and delivery policies so teams can integrate autonomously without a central committee approving every change. Standardize the seams, decentralize the decisions. End-to-end reliability still belongs to the whole producer-to-consumer path, not to governance infrastructure alone.
- **Observability closes the loop:** distributed tracing and correlation IDs let you follow one business process across spokes — turning an implicit choreography back into something you can see, monitor, and reason about.

## Keeping the experience seamless over decoupled systems

- A unified experience does not require one unified runtime or database. It requires one coherent account of the journey, assembled from state owned by autonomous capabilities.
- **Give the journey a durable status model:** expose business milestones such as `OrderReceived`, `Preparing`, `Shipped`, and `Invoiced`, not transport details such as "message queued." A process manager may own this view for an orchestrated flow; a dedicated experience/read-model spoke may project it from events in a choreography. Neither becomes authoritative for the participants' domain data.
- **Use event-carried state transfer and local read models:** the experience can render without synchronous callbacks to every participating system. This deliberately duplicates the minimum data needed for the view while ownership stays with the source context.
- **Make eventual consistency honest:** represent pending, delayed, and failed transitions explicitly. Optimistic UI, "we've received your order," progress indicators, actionable retry/support paths, and timestamps make the handoff understandable instead of pretending it is instantaneous.
- **Design recovery as part of the journey:** after refresh, reconnect, or failover, the user must be able to read the current durable status. Support staff need the same correlation and process history to explain where a journey is blocked.
- **Push updates as an optimization:** WebSockets/SSE can notify a connected client when the durable read model changes, but clients disconnect and notifications can be missed. On reconnect, reload authoritative status; client push never replaces durable system integration or the status model.
- This is the intended coupling: screens, language, and progress form one experience, while each capability retains its own logic, data, deployment, and failure lifecycle underneath.

## Closing

- The goal is not to eliminate the tension but to place it deliberately: coupling in the experience, decoupling in the systems, and a clear owner for the process between them.
- Recap the trilogy:
  - domain events — a fact and its consequences inside a context,
  - integration events — one fact crossing one boundary, reliably,
  - integration topology — many systems, one experience.
- Forward-point to the continuation: how the process-manager spoke actually works — driving a multi-step process to a consistent end through compensation, across decoupled and eventually consistent systems.
