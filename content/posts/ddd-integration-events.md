---
title: "Crossing Boundaries with Integration Events"
date: 2026-07-25T12:00:00+03:00
draft: false
summary: "How to turn internal domain facts into reliable contracts between bounded contexts."
tags: ["software-architecture", "domain-driven-design", "distributed-systems"]
editLink: "https://github.com/deniskyashif/blog/blob/main/content/posts/ddd-integration-events.md"
---

A domain event records a meaningful business fact inside the model where that fact became true. It can trigger local reactions, but its name, types, and payload belong to that domain model and are free to evolve with it. We can therefore think of domain events as **internal events**. For a fuller introduction, see [Modeling Facts and Reactions with Domain Events](/2026/07/25/modeling-facts-and-reactions-with-domain-events).

In this article, we cover what happens when that fact must cross into another bounded context: a model with its own language and responsibilities.

## Crossing the boundary

Suppose Ordering and Fulfillment are independently owned and deployed contexts, and Fulfillment needs to prepare a newly placed order. They need a stable public contract containing the business information that Fulfillment requires, without coupling it to Ordering's internal model. This contract is an **integration event**, which we can think of as an **external event**.

<img src="/images/posts/ddd-integration-events/1.svg" alt="Integration event between two contexts" />

Delivered through durable asynchronous messaging, using infrastructure such as Apache Kafka, RabbitMQ, or Azure Service Bus, integration events let Ordering finish without waiting for Fulfillment. Each context can process work at its own pace, temporary outages do not have to propagate back to the producer, and messages can remain available until a consumer is ready to handle them.

These benefits come with new challenges:  either context may be unavailable, messages may be delayed or duplicated, and the two contexts cannot share a database transaction. We therefore have two problems to solve:

1. Designing the contract
2. Delivering it reliably

## Design the contract for the boundary

### Shape the payload for the boundary, not the producer's model

Suppose that, in this model, placing an order makes it ready to enter Fulfillment, and Ordering raises this domain event when that happens:

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

This event is designed for Ordering's internal handlers. Its `ProductId`, `Quantity`, and other typed identifiers are value objects that enforce invariants and communicate intent: `Quantity` forbids negative values, and an `OrderId` cannot be passed where a `CustomerRef` is expected. These are classes with behavior, not a serialization format. Publishing the event directly would turn its internal types and payload into a public contract. Consumers would then be affected by changes made only for Ordering's own needs. The payload is also a poor fit for the Fulfillment boundary: it exposes `customer`, which Fulfillment does not need, and expresses products using Ordering's internal identifiers.

Instead, Ordering can translate the same fact into an integration event designed for the Fulfillment boundary:

```json
{
  "metadata": {
    "type": "order-ready-for-fulfillment",
    "version": 1,
    "messageId": "01JZ2Q5Y7M8K9N0P1R2S3T4V5W",
    "occurredAt": "2026-07-25T09:42:18Z"
  },
  "body": {
    "orderId": "ord-8472",
    "items": [
      { "productCode": "CHAIR-BLK", "quantity": 2 },
      { "productCode": "DESK-OAK", "quantity": 1 }
    ]
  }
}
```

The envelope separates transport and contract metadata from the business payload. `metadata` identifies the event contract and carries information used to route, deduplicate, and interpret the message; `body` contains the fact consumed by Fulfillment. Keeping a consistent envelope across event types also lets messaging infrastructure handle concerns such as tracing and deduplication without understanding each body. This exact JSON shape is not essential: some messaging platforms place metadata in headers, and standards such as CloudEvents define their own envelope fields. The important distinction is semantic and consistent across producers and consumers.

The two events describe the same occurrence without having the same name or shape. The integration event uses stable, serialized values rather than Ordering's value-object classes. It includes a message identifier for deduplication, the time of the fact, and a schema version so the contract can evolve deliberately. It omits `customer`, which Fulfillment does not need, and represents each product with a stable code agreed at this boundary. Fulfillment maps that code to its own product code and selects a warehouse according to inventory and picking rules that it owns.

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

An integration event is a published interface, so once another context depends on it, it deserves the same care as a versioned REST or gRPC API: document its schema, field meanings, and guarantees, and evolve it deliberately using an explicit compatibility and versioning policy.

### Publish business facts, not internal state changes

Names can leak the internal model just as easily as payloads. Suppose Fulfillment publishes the following event for Ordering:

```text
FulfillmentStatusChanged
  orderId
  oldStatus: PICKING
  newStatus: DISPATCHED
```

This contract makes Ordering understand Fulfillment's statuses and reproduce part of its state machine. It may expose every transition when Ordering cares about only a few. Changing a status can then affect every consumer, even when the business fact they need has not changed. The contract couples other contexts to Fulfillment's internal model and leaks implementation details.

Asynchronous messaging does not remove coupling by itself; the contract still needs to express the business meaning.

Prefer publishing the fact that matters:

```text
OrderDispatched
  orderId
  dispatchedAt
```

`OrderDispatched` communicates what happened without requiring Ordering to interpret Fulfillment's internal lifecycle. `FulfillmentStatusChanged` is not inherently wrong when a status change is itself meaningful domain language; it becomes problematic when it merely synchronizes fields or exposes implementation details.

### Data events are not integration contracts

[Event sourcing](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing) and [Change Data Capture](https://en.wikipedia.org/wiki/Change_data_capture) produce their own streams of events, and it is tempting to treat them as ready-made integration events. They are not. An event-sourced event is a persistence record used to reconstruct internal state. It may express a genuine business fact such as `OrderPlaced`, but its name, schema, and evolution still serve the producer's model and persistence needs. A change-data-capture stream is lower-level: it is a row-level feed of database mutations that describes how stored data changed, usually without expressing why it changed in business terms. Both streams are internal contracts, not public integration contracts.

A CDC record such as `OrderRow.status changed from 2 to 5` forces consumers to know that status `5` means "fulfilled" and to infer a business fact from a database mutation. Exposing an event-sourced stream creates similar coupling at a higher semantic level: consumers become dependent on records that the producer may need to split, merge, or reshape as its model evolves. Hence, we should treat both streams as internal sources from which integration events may be derived.

### When the consumer needs more data

A consumer often needs data owned by the producing context to act on the fact. There are three common approaches, each with a different tradeoff between autonomy and coupling:

- **Event-carried state transfer:** each event includes the producer-owned data needed to react to that fact. The consumer can act without calling the producer, at the cost of larger payloads and duplicated data.
- **Thin notification plus callback:** the event carries identifiers, and the consumer queries the producer for details via the producer's public API. This keeps payloads small but creates a runtime dependency and may return data that has changed since the event occurred.
- **Local replica:** the consumer builds a read model by processing a public integration-event stream from the producer. It can query that data without depending on the producer's availability, but must operate and synchronize additional storage and tolerate replication lag.

Whichever we choose, one principle holds: **notification does not transfer ownership.** The consumer must not become authoritative for the producer's data. When the consumer needs values as they existed at the time of the fact, and carrying them is practical and appropriate, prefer including them in the event over calling back for potentially newer data.

### Translate at the edge

Translation from internal fact to public contract belongs at the boundary, not inside the aggregate.

- Map the domain event to an integration event in the application or infrastructure layer.
- Keep the aggregate unaware of the integration contract; it raises a domain event and nothing more.
- Not every domain event needs to leave the bounded context: one domain event may produce no integration event.
- It is possible for a single domain event to produce more than one integration event.

<img src="/images/posts/ddd-integration-events/2.svg" alt="Concentric architecture layers showing a domain event translated into an integration event at the edge of a bounded context" />

```text
on OrderPlaced event:
  message = OrderReadyForFulfillmentV1(
    messageId = newId(),
    orderId = event.orderId,
    items = event.items.map(item -> {
      productCode = item.product.code,
      quantity = item.quantity.value
    }),
    occurredAt = event.occurredAt)
```

## Publish reliably: the outbox

Once we've designed the integration event, we still have to publish it. The naive sequence is unsafe:

```text
save(order)
publish(OrderReadyForFulfillmentV1)
```

Suppose `save(order)` commits and the process crashes before publishing the event. Ordering now considers the order placed, but Fulfillment never finds out about it. Reversing the two operations does not help. If we publish first and saving the order then fails, Fulfillment may prepare an order that does not exist.

The problem is that the database and the message broker are two independent systems. In most applications, we cannot wrap both in one atomic transaction. What we can do is commit the domain change and the **intent to publish** in the same database transaction. This is the idea behind the **transactional outbox** pattern.

Instead of sending the event directly, we serialize it into an outbox table stored in the same database as the order:

```text
transaction:
  save(order)
  saveOutboxMessage(OrderReadyForFulfillmentV1)
commit
```

<img src="/images/posts/ddd-integration-events/3.svg" alt="Transactional outbox flow: save the order and outbox message atomically, then publish and mark the message after acknowledgement" />

The order and its publication intent now commit or roll back together. A separate publisher can poll for pending messages, send them, and mark them as published after the broker acknowledges them.

## Consume reliably: duplicates and order

When a message may be delivered more than once, the consumer has to be **idempotent**: processing the same message repeatedly must have the same effect as processing it once. Some operations are naturally idempotent. Setting a known value, for example, may produce the same result every time. Incrementing a counter, reserving inventory, or charging a payment usually does not.

For operations that are not naturally idempotent, Fulfillment can use the event's `messageId` as a deduplication key. It keeps an inbox, or simply a table of processed message identifiers, and records the identifier in the same transaction as the business change:

```text
transaction:
  if messageId was already processed:
    return
  prepareOrder(message)
  recordProcessed(messageId)
commit
```

If processing fails, the transaction rolls back and the broker can deliver the message again. If it succeeds but the acknowledgement to the broker is lost, the next delivery finds the recorded identifier and does nothing. The business change and the processed-message record must be atomic; otherwise, a crash between them brings back the same inconsistency we solved with the outbox.

Delivery order requires a similar qualification. A global order across all messages is rarely available and, fortunately, rarely useful. Fulfillment does not need every order in the exact sequence in which Ordering placed them. It may, however, need events concerning one `orderId` to be processed in order. We should identify this smaller stream and partition messages by the aggregate identifier when the broker supports it.

Even then, retries and concurrent consumers can make ordering less straightforward than it appears. A consumer should prefer explicit state transitions or an aggregate version carried by the event over assumptions about broker timing. It can reject an invalid transition, ignore an older version, or postpone a message until the missing version arrives. These choices depend on the business process and are part of the broader problem of eventual consistency.

## Know when not to use an integration event

Use an integration event when a context can publish a fact and finish its work without waiting for a response. When it needs an immediate answer to continue, a direct API usually makes the dependency clearer.

Sending a request through a broker does not turn it into an event. A message asking a particular recipient to act is a command; one asking for information is a query. If the sender expects a reply, the interaction is still request-response. The callback described earlier is such a query: it is triggered by the event, but is not part of the event contract.

Messaging also adds operational cost. Use it when contexts need to work independently or several consumers need the same fact. Otherwise, a direct call may be simpler.

## Closing

We've started with two problems: designing a contract for the boundary and delivering it reliably. The complete path now looks as follows:

```text
domain decision
  -> domain event
  -> integration event in the outbox
  -> publisher
  -> broker
  -> idempotent consumer
  -> receiving domain decision
```

The domain event remains an internal fact. At the edge of the bounded context, we translate it into a stable contract and store it in the outbox together with the domain change. The publisher eventually sends it, and an idempotent consumer translates it into a decision in its own domain model.

Messaging gives these contexts **temporal decoupling**: they do not have to be available or working at the same time. It does not remove coupling. The producer and consumer still agree on the meaning and evolution of the contract, and they also agree on operational guarantees such as delivery, ordering, and retention.

We need to make those guarantees observable. In practice, this means tracking the age of pending outbox messages, publication lag, retries, and failed deliveries. Messages that cannot be processed need a dead-letter path and a deliberate replay procedure. Correlation and causation identifiers help us follow one business flow across contexts, while replay deserves particular care whenever a handler has side effects that are not idempotent.

Integration events are therefore not merely serialized domain events. They are public contracts combined with a delivery model. Once this foundation is in place, we can address longer-running concerns such as eventual consistency, sagas and process managers, and failure recovery. Integrating many systems introduces another set of questions around hub-and-spoke communication and contract governance, but the same boundary still matters: an integration hub may route and transform contracts; it should not become the owner of a shared domain model.
