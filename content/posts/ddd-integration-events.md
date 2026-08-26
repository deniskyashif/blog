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

## Designing the contract for the boundary

### A payload shaped for the boundary

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

Another possibility is to serialize the producer's object graph, letting the payload follow every association.

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

The nesting becomes part of the public contract, so a refactor such as replacing `OrderLine.product` with a product reference now affects every consumer. A smaller contract containing only the facts needed by the consumer avoids this coupling. How much data to carry, versus letting the consumer fetch it, is its own tradeoff, covered in the next section.

An integration event is a published interface. Once another context depends on it, changes carry the same compatibility concerns as changes to a versioned REST or gRPC API. Its schema, field meanings, guarantees, and versioning policy become part of that interface.

### Business facts rather than internal state changes

Names can leak the internal model just as easily as payloads. Suppose Fulfillment publishes the following event for Ordering:

```text
FulfillmentStatusChanged
  orderId
  oldStatus: PICKING
  newStatus: DISPATCHED
```

This contract makes Ordering understand Fulfillment's statuses and reproduce part of its state machine. It may expose every transition when Ordering cares about only a few. Changing a status can then affect every consumer, even when the business fact they need has not changed. The contract couples other contexts to Fulfillment's internal model and leaks implementation details.

Asynchronous messaging does not remove coupling by itself; the contract still needs to express the business meaning.

The same interaction can instead be expressed as the fact that matters to Ordering:

```text
OrderDispatched
  orderId
  dispatchedAt
```

`OrderDispatched` communicates what happened without requiring Ordering to interpret Fulfillment's internal lifecycle. `FulfillmentStatusChanged` is not inherently wrong when a status change is itself meaningful domain language; it becomes problematic when it merely synchronizes fields or exposes implementation details.

### Data events are not integration contracts

[Event sourcing](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing) and [Change Data Capture](https://en.wikipedia.org/wiki/Change_data_capture) produce their own streams of events, and it is tempting to treat them as ready-made integration events. They are not. An event-sourced event is a persistence record used to reconstruct internal state. It may express a genuine business fact such as `OrderPlaced`, but its name, schema, and evolution still serve the producer's model and persistence needs. A change-data-capture stream is lower-level: it is a row-level feed of database mutations that describes how stored data changed, usually without expressing why it changed in business terms. Both streams are internal contracts, not public integration contracts.

A CDC record such as `OrderRow.status changed from 2 to 5` forces consumers to know that status `5` means "fulfilled" and to infer a business fact from a database mutation. Exposing an event-sourced stream creates similar coupling at a higher semantic level: consumers become dependent on records that the producer may need to split, merge, or reshape as its model evolves. Both streams can instead serve as internal sources from which integration events are derived.

### When the consumer needs more data

A consumer often needs data owned by the producing context to act on the fact. There are three common approaches, each with a different tradeoff between autonomy and coupling:

- **Event-carried state transfer:** each event includes the producer-owned data needed to react to that fact. The consumer can act without calling the producer, at the cost of larger payloads and duplicated data.
- **Thin notification plus callback:** the event carries identifiers, and the consumer queries the producer for details via the producer's public API. This keeps payloads small but creates a runtime dependency and may return data that has changed since the event occurred.
- **Local replica:** the consumer builds a read model by processing a public integration-event stream from the producer. It can query that data without depending on the producer's availability, but must operate and synchronize additional storage and tolerate replication lag.

Whichever approach we choose, **notification does not transfer ownership**: the producing context remains authoritative for its data. When the consumer needs values as they existed at the time of the fact, including them in the event avoids a callback returning potentially newer data, provided carrying them is practical and appropriate.

### Translation at the edge

Translation from an internal fact to a public contract can happen at the boundary rather than inside the aggregate:

- The application or infrastructure layer maps the domain event to an integration event.
- The aggregate remains unaware of the integration contract and raises only its domain event.
- A domain event may produce no integration event when the fact does not need to leave the bounded context.
- A single domain event may also produce more than one integration event.

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

## Reliable publication: the outbox

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

## Reliable consumption: duplicates and order

The outbox prevents an event from being lost between the producer's database and the broker, but it does not guarantee that the event is published only once. The publisher can send a message successfully and then crash before marking the outbox record as published. On restart, it sends the same message again. A similar window exists on the consumer side: the handler can commit its work and lose the acknowledgement, causing the broker to redeliver the message. This is **at-least-once delivery**: a message is not silently discarded, but duplicates are possible.

This means the consumer needs **idempotent** processing: handling the same message repeatedly has the same business effect as handling it once. Some operations are naturally idempotent. Setting a shipment address to a particular value, for example, has the same result each time. Incrementing a counter, reserving inventory, or charging a payment usually does not.

For operations that are not naturally idempotent, Fulfillment can use the event's `messageId` as a deduplication key. It keeps an inbox, or simply a table of processed message identifiers, and records the identifier in the same transaction as the business change:

```text
on OrderReadyForFulfillment message:
transaction:
  if messageId was not already processed:
    prepareOrder(message)
    recordProcessed(messageId)
commit
acknowledge message
```

If processing fails, the transaction rolls back and the message can be retried. If it succeeds but the acknowledgement is lost, the next delivery finds the recorded identifier and does nothing. The business change and the processed-message record must be atomic; otherwise, a crash between them recreates the dual-write problem that the producer solved with its outbox. A unique constraint on the processed `messageId` also protects against two consumer instances receiving the duplicate concurrently.

This inbox pattern covers changes made in the consumer's database. It cannot make a database update and an external side effect, such as calling a payment provider, atomic. Such a handler needs the same boundary treatment again: record an outgoing command in a local outbox, or call an external operation that accepts a stable idempotency key. Deduplicating only before the call is unsafe because the process may crash after the provider succeeds but before the message is recorded as processed.

Messages can also arrive out of order. Fulfillment does not care whether unrelated orders arrive in sequence, but order matters for events about the same `orderId`. For example, `OrderCancelled` with `orderEventSequence: 5` might arrive before a delayed `OrderReadyForFulfillment` with `orderEventSequence: 4`. Processing the older event would prepare a cancelled order.

Using `orderId` as the broker's partition or session key helps preserve per-order delivery order. The producer-assigned `orderEventSequence` provides a second safeguard: after recording sequence 5, Fulfillment can ignore sequence 4 as stale. Deduplication prevents one message from taking effect twice; sequence checks prevent older messages from undoing newer facts.

## When an integration event is not a good fit

An integration event fits when a context can publish a fact and finish its work without waiting for a response. When it needs an immediate answer to continue, a direct API usually makes the dependency clearer.

Sending a request through a broker does not turn it into an event. A message asking a particular recipient to act is a command; one asking for information is a query. If the sender expects a reply, the interaction is still request-response. The callback described earlier is such a query: it is triggered by the event, but is not part of the event contract.

Messaging also adds operational cost. It becomes useful when contexts need to work independently or several consumers need the same fact. Otherwise, a direct call may be simpler.
