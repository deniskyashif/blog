---
title: "Modeling Facts and Reactions with Domain Events"
date: 2026-07-25T08:14:19+03:00
draft: false
summary: "Learn how to model domain events, raise them from aggregates, and react through focused handlers without coupling facts to their consequences."
tags: ["software-architecture", "domain-driven-design"]
---

In this article we'll explore techniques to better react to changes in our domain, namely:

> When something happens, something else should happen.

## The Problem

Consider this simple workflow.

```text
placeOrder(...)
save(order)
sendConfirmationEmailToClient(order)
```

For a simple application, defining this in a [Transaction script](https://martinfowler.com/eaaCatalog/transactionScript.html) makes sense. Zoom in a little, however, and the code tells another story. The important domain fact is that an order was placed. Sending a confirmation email is a consequence of that fact, not part of the fact itself.

Coupling facts and consequences creates several potential issues with our design. Let's update our requirements:

- We want placing the order to also notify our fulfilment department.
- We want this order to be recorded in analytics.

```text
placeOrder(...)
save(order)
sendConfirmationEmailToClient(order)
notifyFulfilment(order)
updateAnalytics(order)
```

Now we have an ordered list of unrelated work that grows with the system. It obscures the domain fact that an order was placed and makes it harder to see which consequences are essential.

It also couples the customer-facing request to external systems. The order may be saved successfully while email delivery, fulfilment notification, or analytics fails. These operations cannot usually be made atomic with the order write, despite being grouped in the same script. Reporting the whole operation as failed can mislead the customer into placing a duplicate order, whereas reporting it as successful can leave follow-up work unfinished.

Each consequence needs its own error handling, retries, and fault tolerance.

There is a subtler problem too. Placing an order may not happen in just one place, so the same list of consequences has to be repeated at every call site, and the paths quietly drift apart when one of them is forgotten.

There must be a better way to handle this. To get there, we must first ask:

> What happened, and who needs to know?

And then record the domain fact in one place, allowing each consequence to react to it independently.

## Domain events describe facts

An order being placed is a fact in the domain. A domain event records such a fact in the language the business uses. It is not an instruction to perform work: `SendConfirmationEmail` describes a technical task, while `OrderPlaced` describes something that has already happened.

Events are named in the **past tense** because they are historical records. A subscriber cannot reject `OrderPlaced`; it can only decide how to react. For the same reason, an event should be **immutable**. If the order later changes, that is a new fact and may warrant a new event rather than a revision of the old one.

An event needs enough information to identify and understand the fact. That usually includes the affected entity, relevant values at the time, and when the event occurred. The exact payload is a design choice: a consumer may use identifiers to query a read model, while a self-contained event may carry a snapshot of the values consumers need. Avoid exposing the aggregate's internal representation merely for convenience.

```text
OrderPlaced
  orderId
  customerId
  itemIds
  placedAt
```

This is also what distinguishes an event from a **command**. A command expresses intent: `PlaceOrder` asks the model to do something and can be refused. An event records the outcome: once the model accepts that command, `OrderPlaced` records the decision. The event is raised in the model at that point. The fact becomes durable when the transaction commits, after which the event can be published to interested consumers.

## Raise events where the decision is made

The place to record a fact is where it becomes true. In a rich domain model, that is the aggregate or domain operation that enforces the relevant rules. `OrderPlaced` does not mean merely that a row was inserted. It means the model accepted the request and satisfied its invariants, so the event belongs to that decision.

The sequence is: validate the rules, change the state, then record the event. The aggregate keeps the event alongside its state until the unit of work collects it.

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

This removes the drift from our original transaction script. Whether an order comes from checkout, an admin tool, or an import, every path through `Order.place` records the same fact. Adding analytics or fulfilment later no longer requires updating every caller.

## React to events with handlers

Recording a fact and reacting to it are separate responsibilities. The aggregate knows that an order was placed, but nothing about email, fulfilment, or analytics. Those reactions live in **event handlers**, with each handler implementing one policy:

```text
class SendConfirmationEmail:
  handle(OrderPlaced event):
    email = buildConfirmation(event.orderId)
    mailer.send(email)

class NotifyFulfilment:
  handle(OrderPlaced event):
    fulfilment.startFor(event.orderId)

class RecordInAnalytics:
  handle(OrderPlaced event):
    analytics.track("order_placed", event)
```

Each policy reads as "when this happened, do that." Handlers do not know about one another and can be added without changing the model that raised the event. The ordered list of unrelated work becomes a set of focused reactions, each able to have its own failure and retry behaviour.

### Who calls the handlers?

The aggregate collects events but never dispatches them. Infrastructure must route them to matching handlers. For an in-process implementation, the unit of work is a natural place to do this because it knows which aggregates participated and whether the transaction committed:

```text
unitOfWork.commit():
  transaction:
    persist(trackedAggregates)
  events = collectEvents(trackedAggregates)
  for event in events:
    for handler in handlersFor(event):
      handler.handle(event)
  clearEvents(trackedAggregates)
```

Handlers run **after** the producing transaction. An email failure should not undo an accepted order, but this also means the handler cannot rely on that transaction for reliability. If one handler succeeds and the next fails, retrying may invoke the first one again. Handlers should therefore be safe to repeat where practical. This simple in-process dispatcher separates responsibilities, but it does not provide durable delivery.

## Crossing aggregate boundaries

A handler that changes another aggregate should invoke that aggregate's behavior rather than mutate its state directly. The receiving aggregate must still enforce its own invariants. To reserve stock after an order is placed, a handler loads `Inventory`, asks it to reserve the items, and commits a new unit of work:

```text
class ReserveStock:
  handle(OrderPlaced event):
    inventory = inventoryRepo.forItems(event.itemIds)
    inventory.reserve(event.orderId)     // may raise InventoryReserved
    unitOfWork.commit()
```

This is a **second transaction**, not an extension of the first. For a brief period the order exists but stock is not yet reserved: the two aggregates are **eventually consistent**. A single transaction across both may sometimes be justified, but it couples their lifecycles. The [one-aggregate-per-transaction](https://www.dddcommunity.org/library/vernon_2011/) guideline encourages us to model the gap explicitly instead.

The command against `Inventory` may produce `InventoryReserved`, which can trigger further policies. Such chains are legitimate, but every hop adds another failure point and makes the causal flow harder to follow. Keep chains short, handlers focused, and operations idempotent.

## Boundaries matter

Aggregates are not the only boundaries that matter. Domain events also belong to a **bounded context** and may reflect its internal language and model. Other bounded contexts should not consume those events directly, because doing so turns internal details into shared contracts.

When a fact must cross that boundary, translate it into an intentionally designed **integration event** containing only what external consumers should depend on. Domain and integration events may describe the same occurrence, but they serve different audiences and should evolve independently.

## Do not overdo it

Not every state change deserves an event. Events are useful when a change represents a meaningful domain fact and one or more policies need to react to it. Creating an event for every setter or database update produces noise rather than a useful model.

Prefer a direct call when the dependency is part of one clear operation and gains nothing from being separated. Events should expose meaningful facts, not hide dependencies that would be easier to understand if they were explicit.

## Closing

Our original script mixed one decision with all of its consequences. Modeled with events, its causal flow becomes:

```text
PlaceOrder
  -> OrderPlaced
     -> Send confirmation
     -> Notify fulfilment
     -> Record in analytics
```

Start with the fact the domain cares about. Record it where the decision is made, react through focused policies, and choose the delivery mechanism only after the transaction and consistency requirements are clear.

Domain events do not make failures or coupling disappear. They make those concerns visible at the right boundaries.

Several related topics sit beyond the scope of this article. Reliable cross-process delivery requires patterns such as a transactional outbox, retries, idempotent consumers, and observability. Longer event chains raise questions about ordering, eventual consistency, and coordination through process managers or sagas. Event sourcing goes further still by using events as the source of state rather than as notifications about state changes. Each builds on the distinctions made here, but solves a different problem.
