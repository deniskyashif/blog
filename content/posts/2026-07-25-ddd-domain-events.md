---
title: "Modeling Facts and Reactions with Domain Events"
date: 2026-07-25T08:14:19+03:00
draft: false
summary: "How to decouple facts from their consequences."
tags: ["software-architecture", "domain-driven-design"]
editLink: "https://github.com/deniskyashif/blog/blob/main/content/posts/2026-07-25-ddd-domain-events.md"
---

Most business logic eventually falls down to the following shape:

> When something happens, something else should happen.

In this article we look at **domain events**: a way to record what happened, and let each consequence react to it independently, without the fact having to know about any of them.

## The problem

Consider this simple workflow.

```text
placeOrder(...)
save(order)
sendConfirmationEmailToClient(order)
```

For a basic application, defining this in a [transaction script](https://martinfowler.com/eaaCatalog/transactionScript.html) makes sense. Zoom in a little, however, and the code tells another story. The important **domain fact** is that an order was placed. Sending a confirmation email is a **consequence** of that fact, not part of the fact itself.

Coupling facts and consequences creates several potential issues with our design. Let's update our requirements:

- We want placing the order to also notify our fulfillment department.
- We want this order to be recorded in analytics.

```text
placeOrder(...)
save(order)
sendConfirmationEmailToClient(order)
notifyFulfillment(order)
updateAnalytics(order)
```

Now we have an ordered list of unrelated work that grows as our system gets more complex. It obscures the domain fact that an order was placed and makes it harder to see which consequences are essential.

It also couples the customer-facing request to external systems. The order may be saved successfully while email delivery, fulfillment notification, or analytics fails. These operations cannot usually be made atomic with the order write, despite being grouped in the same script. Reporting the whole operation as failed can mislead the customer into placing a duplicate order, whereas reporting it as successful can leave follow-up work unfinished. Each consequence needs its own error handling, retries, and fault tolerance.

There is a subtler problem too. Placing an order may not happen in just one place, so we have to remember to include the same set of subsequent actions at every call site.

There must be a better way to handle this. To get there, we must first ask:

> What happened, and who needs to know?

Then record the domain fact in one place, allowing each consequence to react to it independently.

## Domain events describe facts

An order being placed is a fact in the domain. A domain event records such a fact in the language the business uses. It is not an instruction to perform work, e.g. `SendConfirmationEmail` describes a technical task, while `OrderPlaced` describes something that has already happened.

Events are named in the **past tense** because they are historical records. A handler cannot reject `OrderPlaced`; it can only decide how to react. For the same reason, an event should be **immutable**. If the order later changes, that is a new fact and may warrant a new event rather than a revision of the old one.

An event needs enough information to identify and understand the fact. That usually includes the affected entity, relevant values at the time, and when the event occurred. The exact payload is a design choice: a handler may use identifiers to query a read model, while a self-contained event may carry a snapshot of the values handlers need. Avoid exposing the aggregate's internal representation merely for convenience. As a rule of thumb, prefer lean events, unless you have a clear justification to do so otherwise.

```text
OrderPlaced
  orderId
  customerId
  items
  placedAt
```

This is also what distinguishes an **event** from a **command**. A command expresses intent: `PlaceOrder` asks the model to do something and can be refused. An event records the outcome: once the model accepts that command, `OrderPlaced` records the decision. The event is raised in memory at that point. The fact becomes durable when the transaction commits, and then the event can be dispatched to interested handlers.

## Raise events where the decision is made

The place to record a fact is where the fact becomes true. In a rich domain model, that is the aggregate or domain operation that enforces the relevant rules. `OrderPlaced` does not mean merely that a row was inserted. It means the model accepted the request and satisfied its invariants, so the event belongs to that decision.

The sequence is: validate the rules, change the state, then record the event. The aggregate keeps the event alongside its state until the [unit of work](https://martinfowler.com/eaaCatalog/unitOfWork.html) collects it.

```text
class Order:
  events = []

  static place(customerId, items):
    if items is empty: reject
    order = new Order(customerId, items)
    order.events.add(OrderPlaced(order.id, customerId, items.ids, now))
    return order
```

Nothing outside the aggregate constructs `OrderPlaced`. The application service asks the model to do the work and persists the result:

```text
order = Order.place(customerId, items)
repository.add(order)
unitOfWork.commit()   // the domain fact becomes durable
```

This resolves another issue with our original transaction script. Whether an order comes from checkout, an admin tool, or a data import, every path through `Order.place` records the same fact. Adding analytics or fulfillment later no longer requires updating every caller.

## React to events with handlers

Recording a fact and reacting to it are separate responsibilities. The aggregate knows that an order was placed, but nothing about email, fulfillment, or analytics. Those reactions live in **event handlers**, with each handler implementing its own behavior with a single responsiblity:

```text
class SendConfirmationEmail:
  handle(OrderPlaced event):
    email = buildConfirmation(event.orderId)
    mailer.send(email)

class NotifyFulfillment:
  handle(OrderPlaced event):
    fulfillment.startFor(event.orderId)

class RecordInAnalytics:
  handle(OrderPlaced event):
    analytics.track("order_placed", event)
```

Each handler reads as _"when this happened, do that"_. Handlers do not know about one another and can be added without changing the model that raised the event. The ordered list of unrelated work becomes a set of independent actions, each able to have its own failure and retry behavior.

### Who calls the handlers?

<img src="/images/posts/2026-07-25-ddd-domain-events/1.svg" width="450px" alt="Order placed event being dispatched to its handlers" />

The aggregate collects events but never dispatches them. Infrastructure must route them to matching handlers. For an in-process implementation, the unit of work is a natural place to do this because it knows which aggregates participated and whether the transaction committed:

```text
unitOfWork.commit:
  transaction:
    persist(trackedAggregates)
  events = collectEvents(trackedAggregates)
  for event in events:
    for handler in handlersFor(event):
      handler.handle(event)   // error handling and retries omitted for brevity
  clearEvents(trackedAggregates)
```

> **A note on execution:** This refactoring improves the logical separation, but the dispatcher remains synchronous: the request waits for every handler, and delivery is not durable. The naive loop above also lets one handler's failure affect the others, so each reaction needs its own error handling and retries. For asynchronous, reliable delivery, look into the [transactional outbox](https://en.wikipedia.org/wiki/Inbox_and_outbox_pattern) pattern. Handlers should still be safe to repeat.

## Crossing aggregate boundaries

Some handlers call external systems, while others initiate behavior elsewhere in the domain. A handler that triggers a change in another aggregate should invoke that aggregate's behavior rather than mutate its state directly. The receiving aggregate must still enforce its own invariants. For example, within the same bounded context, reserving stock after an order is placed requires a handler to load `Inventory`, ask it to reserve the items, and commit a new unit of work:

```text
class ReserveStock:
  handle(OrderPlaced event):
    for itemId in event.itemIds:
      inventory = inventoryRepository.get(itemId)
      inventory.reserve(event.orderId)   // may raise InventoryReserved
    unitOfWork.commit()   // failure handling omitted for brevity
```

This is a **second transaction**, not an extension of the first. For a brief period the order exists but stock is not yet reserved: the two aggregates are **eventually consistent**. A single transaction across both may sometimes be justified, but it couples their lifecycles. As a rule of thumb, follow the one-aggregate-per-transaction guideline and model the gap explicitly instead. Having to update two aggregates within a single transaction might indicate a design problem within the domain model.

If these reactions are later processed asynchronously, this design also improves availability: Ordering can continue accepting orders while Inventory resolves shortages, because unavailable stock delays fulfillment rather than causing the order itself to be rejected.

<img src="/images/posts/2026-07-25-ddd-domain-events/2.svg" style="width: 600px; margin-top: 20px; margin-bottom: 15px" alt="An aggregate event triggers a change in another aggregate." />

The command against `Inventory` may produce `InventoryReserved`, which can trigger further behaviors. Such chains are legitimate, but every hop adds another failure point and makes the causal flow harder to follow. Hence, it's important to keep the event chains short, handlers focused, and operations [idempotent](https://en.wikipedia.org/wiki/Idempotence).

## Crossing context boundaries

The inventory example crossed an aggregate boundary while remaining inside one bounded context. `OrderPlaced` could therefore be handled using Ordering's own domain language and an in-process dispatcher. Now suppose the fulfillment department from our original workflow is represented by its own Fulfillment context.

Crossing into another bounded context is different. A domain event belongs to the model that produced it: its name and payload may expose assumptions that make sense inside Ordering but not within Fulfillment.

<img src="/images/posts/2026-07-25-ddd-domain-events/3.svg" style="margin-top: 20px; margin-bottom: 15px" alt="Integration event across bounded contexts" />

`OrderReadyForFulfillment` is an **integration event**: Fulfillment consumes it at the boundary and translates it into its own local command, such as `PrepareOrder`. It does not need to understand Ordering's aggregates or internal event model. This translation doesn't need to be one-to-one. A context may combine several domain facts and invoke different behaviors for different consumers.

Bounded contexts are often deployed as separate processes or services, although they don't have to be. When they are, integration events are typically delivered asynchronously through messaging infrastructure such as Kafka, RabbitMQ, or Azure Service Bus. Unlike the in-process dispatcher shown earlier, this introduces serialization, delivery failures, duplicates, and versioned contracts. Reliable publication usually requires a transactional outbox or an equivalent mechanism.

## Don't overdo it

Not every state change deserves an event. Events are useful when a change represents a meaningful domain fact and one or more consumers need to react to it. Creating an event for every setter or database update produces noise rather than a useful model.

Prefer a direct call, via domain or application services, when the dependency is part of one clear operation and gains nothing from being separated. Events should expose meaningful facts within our business domain and not hide dependencies that would be easier to understand if they were explicit.

## Conclusion

Domain events let us separate a fact from its consequences: the model records what happened, and each reaction decides independently how to respond. As a rule of thumb, start with the fact the business cares about. Raise the domain event where the decision is made, react through focused handlers, and translate it into an integration event when crossing context boundaries. Choose between synchronous and durable asynchronous delivery only after the transaction, consistency, and failure requirements are clear.

Used with restraint, this keeps the domain fact explicit while giving each consequence its own place to fail, retry, and evolve — and leaves the door open to distributing the work across processes or services when the system genuinely calls for it.
