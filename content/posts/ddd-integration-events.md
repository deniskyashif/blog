---
title: "Crossing Boundaries with Integration Events"
date: 2026-07-25T12:00:00+03:00
draft: true
summary: "How to turn internal domain facts into reliable contracts between bounded contexts."
tags: ["software-architecture", "domain-driven-design", "distributed-systems"]
editLink: "https://github.com/deniskyashif/blog/blob/main/content/posts/ddd-integration-events.md"
---

This is a follow-up to [Modeling Facts and Reactions with Domain Events](/posts/2026-07-25-ddd-domain-events/), which ended by naming the transactional outbox and versioned contracts as open problems. Here we pick those up.

Domain events describe meaningful facts inside a bounded context. But what happens when another context needs to know about one of those facts?

Sending the domain event directly may look like the simplest option. It also turns an internal model into a public contract and introduces delivery failures, duplicate messages, ordering concerns, and independent evolution on both sides of the boundary.

This article explores how to cross that boundary deliberately:

> Translate internal facts into stable integration contracts, then deliver them reliably.

## The boundary changes the problem

- Continue with `OrderPlaced` from the domain-events article.
- Introduce a separate Fulfillment bounded context that needs to prepare the order.
- Contrast an in-process handler with communication between independently owned contexts.
- Explain that crossing a boundary introduces two concerns:
  - What contract should be shared?
  - How should it be delivered reliably?

```text
Ordering context                       Fulfillment context

OrderPlaced (domain event)
        |
        v
OrderReadyForFulfillment  ----------->  PrepareOrder
   (integration event)
```

## Domain events are not integration events

- Domain events belong to the model that raises them.
- They may contain domain-specific types and change as that model evolves.
- Integration events are public contracts designed for external consumers.
- The two events may describe the same occurrence without having the same name or shape.
- Explain why sharing the domain event class creates coupling despite asynchronous transport.

```text
Domain event:
  OrderPlaced
    orderId
    customerId

Integration event:
  OrderReadyForFulfillmentV1
    messageId
    orderId
    warehouseId
    items
    occurredAt
```

## Design the contract for the boundary

- Include facts consumers need, not the producer's object graph.
- Prefer stable identifiers and values over internal entities or value-object types.
- Decide whether consumers should receive a notification or a self-contained event.
- Discuss the tradeoff between small payloads that require callbacks and larger payloads that create data duplication.
- Include message identity, event time, and schema version where useful.
- Avoid leaking sensitive or unnecessary data.
- Make ownership explicit: the producer owns the contract and consumers depend on it.

## When the consumer needs more data

- A consumer often needs data owned by the producing context to act on the fact.
- Present three approaches and their tradeoffs:
  - **Event-carried state transfer:** the event carries enough data that no callback is needed. Maximizes autonomy and availability at the cost of larger payloads and deliberate data duplication. Prefer this default across contexts.
  - **Thin notification plus callback:** the event carries identifiers; the consumer queries the producer for details. Smaller payloads, but reintroduces temporal coupling, a runtime dependency on the producer, and a consistency hazard (data may have changed since the event).
  - **Local replica:** the consumer maintains its own read model from a stream of events. Strong autonomy at the cost of more infrastructure.
- State the principle: **notification does not transfer ownership.** The consumer must not become authoritative for the producer's data.
- If the consumer needs the producer's data to make its own decision, prefer carrying it in the event (as of that moment) over calling back.

## Translate at the edge

- Keep translation outside the aggregate.
- Map the domain event to an integration event in the application or infrastructure layer.
- Show that not every domain event needs to leave the bounded context.
- Allow one domain event to produce no integration event, or a contract tailored to a specific boundary.

```text
on OrderPlaced event:
  message = OrderReadyForFulfillmentV1(
    messageId = newId(),
    orderId = event.orderId,
    warehouseId = selectWarehouse(event),
    items = loadFulfillmentItems(event.orderId),
    occurredAt = event.occurredAt
  )
```

## The dual-write problem

- Show the unsafe sequence:

```text
save(order)
publish(OrderReadyForFulfillmentV1)
```

- Explain both failure directions:
  - The order commits, but publishing never happens.
  - Publishing happens, but the order transaction rolls back.
- State why a database transaction cannot usually include an external broker atomically.
- Establish the requirement: commit the domain change and the intent to publish together.

## Use a transactional outbox

- Store the integration event in an outbox table in the **same database transaction** as the order.
- Emphasize that these are two writes but one transaction.

```text
transaction:
  save(order)
  saveOutboxMessage(OrderReadyForFulfillmentV1)
commit
```

- A separate publisher reads pending messages and sends them to the broker.
- Mark messages as published only after the broker acknowledges them.
- Explain polling and change-data-capture approaches briefly.
- State what the outbox guarantees and what it does not:
  - It prevents a committed change from losing its publication intent.
  - It does not guarantee immediate delivery.
  - It does not produce exactly-once processing.

## Expect duplicate delivery

- Show the acknowledgement failure window: the broker accepts a message, but the publisher crashes before marking it complete.
- Explain why retries create at-least-once delivery.
- Make the consumer idempotent using `messageId` and an inbox or processed-message record.

```text
transaction:
  if messageId was already processed:
    return
  prepareOrder(message)
  recordProcessed(messageId)
commit
```

- Keep the business change and processed marker in the same consumer transaction.
- Mention naturally idempotent operations where appropriate.

## Ordering is contextual

- Explain that global ordering is rarely available or necessary.
- Identify the stream that actually requires order, such as events for one `orderId`.
- Discuss partitioning by aggregate identifier.
- Explain how retries can cause later messages to arrive first.
- Prefer version checks or explicit state transitions over assumptions about broker timing.
- Keep a deeper treatment for the eventual-consistency article.

## Evolve contracts deliberately

- Prefer additive, backward-compatible changes where practical.
- Do not silently change the meaning of an existing field.
- Introduce a new version for breaking semantic or structural changes.
- Support old versions for an explicit migration period.
- Discuss tolerant readers without using them to excuse unspecified contracts.
- Add producer and consumer contract tests.

## Make failures observable

- Track outbox age, publication lag, retries, and failed messages.
- Use correlation and causation identifiers to trace a business flow across contexts.
- Provide dead-letter handling with a deliberate replay process.
- Alert on stalled delivery rather than relying on logs alone.
- Be careful when replaying messages whose side effects are not idempotent.

## Transport is a separate decision

- Integration events are a pattern, not a transport. Separate three things:
  - the fact itself (the contract),
  - how it is delivered between contexts (durable messaging),
  - how end users are notified (a downstream concern).
- **Pub/sub (topic-based messaging):** the natural fit. The producer publishes a fact; any number of consumers subscribe independently. Matches producer-owned contracts and one-to-many, fire-and-forget delivery. This is the model assumed throughout this article.
- **Async request-response:** this is a **command or query with a reply, not an event**. It solves a different use case — "I need a specific answer from a specific context to continue" — versus "this fact happened; whoever cares can react." It is one-to-one and expects a response, and it reintroduces a logical dependency on the responder. The callback from the earlier section is exactly this: an async query, not part of the event contract. Mixing it up with pub/sub is how synchronous coupling gets disguised behind a broker.
- **WebSockets / SSE:** these push to end clients (a browser or UI), not between contexts. They provide no durability, outbox semantics, ordering, or replay, and are connection-scoped. They sit downstream: a context consumes an integration event, updates its state, and then pushes a UI notification. They are not a substitute for durable messaging between contexts.

## Know when not to use an integration event

- Prefer a direct API when the caller needs an immediate answer to continue.
- Do not use events to disguise synchronous coupling.
- Avoid asynchronous messaging when the operational cost outweighs the independence it provides.
- Distinguish notification from data ownership: an event does not make another context authoritative for the producer's data.

## Closing

- Return to the two concerns introduced at the start: contract and delivery.
- Recap the complete path:

```text
domain decision
  -> domain event
  -> integration event in the outbox
  -> publisher
  -> broker
  -> idempotent consumer
  -> receiving domain decision
```

- Reinforce that messaging creates temporal decoupling, not an absence of coupling.
- Point to related topics beyond this article:
  - eventual consistency,
  - sagas/process managers,
  - failure recovery,
  - integrating many systems (hub-and-spoke, contract governance, and why the hub should not own a data model) — where the business process spans systems and the goal is to couple the experience while decoupling the systems.
