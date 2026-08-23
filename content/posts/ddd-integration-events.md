---
title: "Crossing Boundaries with Integration Events"
date: 2026-07-25T12:00:00+03:00
draft: false
summary: "How to turn internal domain facts into reliable contracts between bounded contexts."
tags: ["software-architecture", "domain-driven-design", "distributed-systems"]
editLink: "https://github.com/deniskyashif/blog/blob/main/content/posts/ddd-integration-events.md"
---

A domain event records a meaningful business fact, such as `OrderPlaced`, inside the model where that fact became true. It can trigger local reactions, but its name, types, and payload belong to that model and are free to evolve with it. We can therefore think of domain events as **internal events**. For a fuller introduction, see [Modeling Facts and Reactions with Domain Events](/2026/07/25/modeling-facts-and-reactions-with-domain-events).

This article concerns what happens when that fact must cross into another bounded context: a model with its own language and responsibilities. Suppose Ordering and Fulfillment are independently owned and deployed, and Fulfillment needs to prepare a newly placed order.

## Crossing the boundary

The contexts need a stable public contract containing the business information Fulfillment requires, without coupling it to Ordering's internal model. That contract is an **integration event**, which we can think of as an **external event**.

<img src="/images/posts/ddd-integration-events/1.svg" alt="Integration event between two contexts" />

Delivered through durable asynchronous messaging, using infrastructure such as Apache Kafka, RabbitMQ, or Azure Service Bus, integration events let Ordering finish without waiting for Fulfillment. Each context can process work at its own pace, temporary outages do not have to propagate back to the producer, and messages can remain available until a consumer is ready to handle them.

These benefits come with new challenges: messages may be delayed or duplicated, either context may be unavailable, and the two contexts cannot share a database transaction. We therefore have two problems to solve:

1. Designing the contract
2. Delivering it reliably

## Design the contract for the boundary

### Shape the payload for the boundary, not the producer's model

Suppose Ordering raises this domain event when an order is placed:

```text
OrderPlaced
  orderId: OrderId
  customer: CustomerRef
  items: List<OrderItem>
  placedAt: Timestamp

OrderItem
  product: ProductId
  quantity: Quantity
```

This event is designed for Ordering's internal handlers. Its `ProductId`, `Quantity`, and other typed identifiers are value objects that enforce invariants and communicate intent: `Quantity` forbids negative values, and an `OrderId` cannot be passed where a `CustomerRef` is expected. These are classes with behavior, not a serialization format. Publishing the event directly would turn its internal types and payload into a public contract, so consumers would be affected by changes made only for Ordering's own needs. The payload is also a poor fit for the Fulfillment boundary: it exposes `customer`, which Fulfillment does not need, and expresses products using Ordering's internal identifiers.

Instead, Ordering can translate the same fact into an integration event designed for the Fulfillment boundary:

```json
{
  "type": "order-ready-for-fulfillment",
  "version": 1,
  "messageId": "01JZ2Q5Y7M8K9N0P1R2S3T4V5W",
  "occurredAt": "2026-07-25T09:42:18Z",
  "orderId": "ord-8472",
  "items": [
    { "productCode": "CHAIR-BLK", "quantity": 2 },
    { "productCode": "DESK-OAK", "quantity": 1 }
  ]
}
```

The two events describe the same occurrence without having the same name or shape. The integration event uses stable, serialized values rather than Ordering's value-object classes. It includes a message identifier for deduplication, the time of the fact, and a schema version (more on this later when we talk about versioning). It omits `customer`, which Fulfillment does not need, and represents each product with a stable code agreed at this boundary. Fulfillment maps that code to its own SKU and selects a warehouse according to inventory and picking rules that it owns.

Ordering still owns this boundary-specific contract. Another boundary may require a different integration event derived from the same domain event.

A related mistake is to over-share by serializing the producer's object graph, letting the payload follow every association.

```json
{
  "order": {
    "customer": { "id": "cust-19", "email": "...", "billingAddress": { "...": "..." } },
    "lines": [
      { "product": { "id": "prod-44", "name": "Chair", "supplier": { "...": "..." } },
        "unitPrice": { "amount": 12900, "currency": "EUR" }, "quantity": 2 }
    ],
    "payment": { "...": "..." }
  }
}
```

The nesting becomes part of the public contract, so a refactor such as replacing `OrderLine.product` with a product reference now affects every consumer. Include the facts the consumer needs to act and nothing more. How much data to carry, versus letting the consumer fetch it, is its own tradeoff, covered in the next section.

### Publish business facts, not internal state changes

Names can leak the internal model just as easily as payloads. Consider an event published only to keep another context's status field synchronized:

```text
OrderStatusChanged
  orderId
  oldStatus: PROCESSING
  newStatus: FULFILLED
```

This contract makes Fulfillment understand Ordering's set of statuses and reproduce part of its state machine. Adding, renaming, removing, or redefining a status can therefore require changes in Fulfillment and any other consumer, even when the business fact they care about has not changed. The contract couples other contexts to Ordering's internal data model and leaks implementation details.

> Asynchronous messaging does not remove that coupling; the contract still needs to express the business meaning at the boundary.

Prefer publishing the fact that matters at the boundary:

```text
OrderFulfilled
  orderId
  fulfilledAt
```

`OrderFulfilled` communicates what happened without requiring consumers to interpret Ordering's internal lifecycle. `OrderStatusChanged` is not inherently wrong when a status change is itself meaningful domain language; it becomes problematic when it merely synchronizes fields or exposes implementation details.

### Data events are not integration contracts

[Event sourcing](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing) and [Change Data Capture](https://en.wikipedia.org/wiki/Change_data_capture) produce their own streams of events, and it is tempting to treat them as ready-made integration events. They are not. An event-sourced event is a persistence record used to reconstruct internal state. It may express a genuine business fact such as `OrderPlaced`, but its name, schema, and evolution still serve the producer's model and persistence needs. A change-data-capture stream is lower-level: it is a row-level feed of database mutations that describes how stored data changed, usually without expressing why it changed in business terms. Both streams are internal contracts, not public integration contracts.

A CDC record such as `OrderRow.status changed from 2 to 5` forces consumers to know that status `5` means "fulfilled" and to infer a business fact from a database mutation. Exposing an event-sourced stream creates similar coupling at a higher semantic level: consumers become dependent on records that the producer may need to split, merge, or reshape as its model evolves. Treat both streams as internal sources from which integration events may be derived.

## When the consumer needs more data

A consumer often needs data owned by the producing context to act on the fact. There are three common approaches, each with a different tradeoff between autonomy and coupling:

- **Event-carried state transfer:** each event includes the producer-owned data needed to react to that fact. The consumer can act without calling the producer, at the cost of larger payloads and duplicated data.
- **Thin notification plus callback:** the event carries identifiers, and the consumer queries the producer for details. This keeps payloads small but creates a runtime dependency and may return data that has changed since the event occurred.
- **Local replica:** the consumer builds a non-authoritative read model by processing the producer's event stream. It can query that data without depending on the producer's availability, but must operate and synchronize additional storage and tolerate replication lag. (A fuller treatment belongs with eventual consistency.)

Whichever you choose, one principle holds: **notification does not transfer ownership.** The consumer must not become authoritative for the producer's data. When the consumer needs values as they existed at the time of the fact, and carrying them is practical and appropriate, prefer including them in the event over calling back for potentially newer data.

## Translate at the edge

Translation from internal fact to public contract belongs at the boundary, not inside the aggregate.

- Map the domain event to an integration event in the application or infrastructure layer.
- Keep the aggregate unaware of the wire contract; it raises a domain event and nothing more.
- Not every domain event needs to leave the bounded context: one domain event may produce no integration event, or a contract tailored to a specific boundary.

<img src="/images/posts/ddd-integration-events/2.svg" alt="Concentric architecture layers showing a domain event translated into an integration event at the edge of a bounded context" />

```text
on OrderPlaced event:
  message = OrderReadyForFulfillmentV1(
    messageId = newId(),
    orderId = event.orderId,
    items = event.items.map(item -> {
      productCode = productCodeFor(item.product),
      quantity = item.quantity.value
    }),
    occurredAt = event.occurredAt)
```

## Publish reliably: the outbox

> _Skeleton — to write._

The naive sequence is unsafe:

```text
save(order)
publish(OrderReadyForFulfillmentV1)
```

- Explain both failure directions: the order commits but publishing never happens; or publishing happens but the order transaction rolls back.
- State why a database transaction cannot usually include an external broker atomically. The requirement: commit the domain change and the intent to publish together.
- **The transactional outbox:** store the integration event in an outbox table in the **same database transaction** as the order. Two writes, one transaction.

```text
transaction:
  save(order)
  saveOutboxMessage(OrderReadyForFulfillmentV1)
commit
```

- A separate publisher reads pending messages, sends them to the broker, and marks them published only after the broker acknowledges. Mention polling vs. change-data-capture in one line.
- State what the outbox guarantees (a committed change never loses its publication intent) and what it does not (immediate delivery, exactly-once processing).

_Evolve contracts deliberately (folded paragraph):_ treat integration events as public APIs. Prefer additive, backward-compatible changes; never silently change the meaning of an existing field; introduce a new version for breaking structural or semantic changes and support old versions for an explicit migration period. Back this with producer and consumer contract tests.

## Consume reliably: duplicates and order

> _Skeleton — to write._

- **At-least-once delivery:** the broker accepts a message but the publisher crashes before marking it complete; retries then redeliver. Make the consumer idempotent using `messageId` and an inbox / processed-message record, kept in the same transaction as the business change.

```text
transaction:
  if messageId was already processed:
    return
  prepareOrder(message)
  recordProcessed(messageId)
commit
```

- Mention naturally idempotent operations where they apply.
- **Ordering is contextual:** global ordering is rarely available or necessary. Identify the stream that actually requires order (events for one `orderId`) and partition by aggregate identifier. Since retries can reorder messages, prefer version checks or explicit state transitions over assumptions about broker timing. Defer a deeper treatment to the eventual-consistency article.

## Know when not to use an integration event

> _Skeleton — to write._

- Prefer a direct API when the caller needs an immediate answer to continue. Do not use events to disguise synchronous coupling.
- **Pub/sub vs. async request-response:** an event is "this fact happened; whoever cares can react" (one-to-many, fire-and-forget, producer-owned contract). An async request-response is a **command or query with a reply** — one-to-one, expects an answer, reintroduces a logical dependency on the responder. The callback from the earlier section is exactly this: an async query, not part of the event contract. Mixing the two is how synchronous coupling gets disguised behind a broker.
- **WebSockets / SSE** push to end clients, not between contexts: no durability, outbox semantics, ordering, or replay. They sit downstream — a context consumes an integration event, updates state, then pushes a UI notification — and are not a substitute for durable messaging between contexts.
- Avoid asynchronous messaging when the operational cost outweighs the independence it provides.

## Closing

> _Skeleton — to write._

- Return to the two concerns from the start: contract and delivery.
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
- _Observability (folded paragraph):_ track outbox age, publication lag, retries, and failed messages; provide dead-letter handling with a deliberate replay process; use correlation and causation identifiers to trace a business flow across contexts; be careful replaying messages whose side effects are not idempotent.
- Point to related topics beyond this article: eventual consistency, sagas / process managers, failure recovery, and integrating many systems (hub-and-spoke, contract governance, and why the hub should not own a data model).
