---
title: "Making Things Happen with Domain Events"
date: 2026-07-25T08:14:19+03:00
draft: false
summary: "A practical guide to working with events in domain modeling."
tags: ["software-architecture", "domain-driven-design"]
---

In this article we'll explore techniques to better react to changes in our domain, namely:

> When something happens, something else should happen.

## The Problem

Consider this simple workflow.

```text
createOrder(...)
save(order)
sendConfirmationEmailToClient(order)
```

For a simple application, defining this in a [Transaction script](https://martinfowler.com/eaaCatalog/transactionScript.html) makes sense. Zoom in a little, however, and the code tells another story. The important domain fact is that an order was created. Sending a confirmation email is a consequence of that fact, not part of the fact itself.

Coupling facts and consequences creates several potential issues with our design. Let's update our requirements:

- We want the order creation to also notify our fulfilment department.
- We want this order to be recorded in analytics.

```text
createOrder(...)
save(order)
sendConfirmationEmailToClient(order)
notifyFulfilment(order)
updateAnalytics(order)
```

Now we have an ordered list of unrelated work that grows with the system. It obscures the domain fact that an order was created and makes it harder to see which consequences are essential.

It also couples the customer-facing request to external systems. The order may be saved successfully while email delivery, fulfilment notification, or analytics fails. These operations cannot usually be made atomic with the order write, despite being grouped in the same script. Reporting the whole operation as failed can mislead the customer into placing a duplicate order, whereas reporting it as successful can leave follow-up work unfinished.

Each consequence needs its own error handling, retries, and fault tolerance.

There is a subtler problem too. Order creation may not happen in just one place, so the same list of consequences has to be repeated at every call site, and the paths quietly drift apart when one of them is forgotten.

There must be a better way to handle this. To get there, we must first ask: 

> What happened, and who needs to know?

And then model this business logic in a single place within our domain.

## A domain event is a past-tense fact

An order being created is a fact in the domain. A domain event records such a fact in the language the business uses. It is not an instruction to perform work: `SendConfirmationEmail` describes a technical task, while `OrderCreated` describes something that has already happened.

Events are named in the **past tense** because they are historical records. No subscriber can reject or undo `OrderCreated`, it can only decide how to react to it. For the same reason, an event should be **immutable**. If the order later changes, that is a new fact and may warrant a new event rather than changing the record of what happened before.

An event carries the information a recipient needs to understand the fact without reaching back into the domain model. At a minimum, that usually means the identity of the affected entity, the relevant values at the time, and when the event occurred.

```text
OrderCreated
  orderId
  customerId
  createdAt
```

## Events and Commands are not the same

A **command** expresses intent: it asks the model to do something and can be refused. Think of `CreateOrder` as the `POST /orders` request behind it, handled in one place, enforcing the rules, and free to reject the call. An **event** records a fact that follows from a command that succeeded: once `Order.create` accepts the request and commits, `OrderCreated` states that it happened and cannot be undone.

## Raise events where the decision is made

If an event records a fact, then the place to record it is where that fact becomes true. In a rich domain model, that is the aggregate or domain operation that enforces the rule. `OrderCreated` is not simply "a row was written to the orders table", it is the moment the model accepted a request, checked its invariants, and committed to a new state. The event belongs to that decision, so it should be raised there.

The sequence inside the operation reads the same way every time: validate the rule, change the state, then record the event. The aggregate keeps its own list of events and appends to it as part of the state change, so the event cannot exist unless the change that justifies it has already happened.

```text
class Order:
  events = []

  static create(customerId, items):
    if items is empty: reject
    order = new Order(customerId, items)          // state change
    order.events.add(OrderCreated(order.id, customerId, now))
    return order
```

The order matters. An event is a past-tense fact, so it can only be recorded once the state change it describes has actually happened. Appending the event before the invariants are checked would let us announce something that might never become true.

Notice that nothing outside the aggregate raises the event. The application service only asks the model to do the work and then persists the result. It never constructs an `OrderCreated` itself.

```text
order = Order.create(customerId, items)
repository.add(order)
unitOfWork.commit()   // the fact becomes durable here
```

The events ride along on the aggregate and are dispatched after the unit of work commits successfully. Recording the event inside the model keeps intent and fact together, and it makes the event part of the model's **public language**. `OrderCreated`, `OrderConfirmed`, and `OrderCancelled` describe what the domain can do in the same vocabulary the business uses to talk about it.

This also resolves the drift we saw earlier. It no longer matters whether order creation is triggered from a checkout endpoint, an admin tool, or a background import: every path goes through `Order.create`, and the fact is recorded in exactly one place. The consequences are no longer copied into each call site; they follow from the event. Adding analytics or fulfilment later means subscribing to `OrderCreated` once, not remembering to update every caller.

## Handling events

Recording a fact and reacting to it are two different things. The aggregate's job ends once it has raised `OrderCreated`, it should know nothing about emails, fulfilment, or analytics. The reactions live in **event handlers**: small, focused pieces of code that subscribe to an event and carry out one follow-up policy each.

```text
class SendConfirmationEmail:
  handle(OrderCreated event):
    email = buildConfirmation(event.orderId)
    mailer.send(email)

class NotifyFulfilment:
  handle(OrderCreated event):
    fulfilment.startFor(event.orderId)

class RecordInAnalytics:
  handle(OrderCreated event):
    analytics.track("order_created", event)
```

Each handler is a **policy** that reads as "when this happened, do that." One event can have many handlers, and none of them know about the others. This is exactly what untangles the ordered list of consequences we started with: the growing script of unrelated work becomes a set of independent reactions, each with its own error handling and retry behaviour, added or removed without touching the code that raised the event.

### Who calls the handlers?

The aggregate collects events but never dispatches them; something outside has to pick them up and route them to the handlers. That something is a **dispatcher**, and the natural place to invoke it is the unit of work, because it already knows which aggregates took part in the transaction and when that transaction succeeds.

Recall the earlier warning: an event is a past-tense fact, and delivering it before the state change is durable would let us announce something that might still roll back. So the dispatcher runs **after the commit**. Once the transaction succeeds, the unit of work drains the events that rode along on each aggregate and hands them to the matching handlers.

```text
unitOfWork.commit():
  transaction:
    persist(trackedAggregates)   // the facts become durable
  events = collectEvents(trackedAggregates)
  for event in events:
    for handler in handlersFor(event):
      handler.handle(event)
  clearEvents(trackedAggregates)
```

The application service stays unchanged, it still just asks the model to do the work and commits. Wiring the dispatch into the unit of work keeps the call site unaware of who reacts.

Dispatching after the commit has one consequence worth stating: the handlers run **outside** the transaction that produced the event. The order is already durable when `SendConfirmationEmail` runs, so if that handler fails, the order does not disappear with it. That is usually what we want, an email failure should not undo an accepted order, but it means each handler now owns its own reliability.

Keeping raising and handling separate also keeps the command path readable. `Order.create` expresses one decision; the handlers express what follows from it. A newcomer can understand order creation without wading through email templates, and can find every consequence of the fact by looking at who subscribes to `OrderCreated`.

### A handler that changes another aggregate

A handler often needs to change other state, and it should do so the same way any other caller would: by issuing a **command** against the receiving aggregate rather than reaching in and mutating it directly, so that aggregate can still enforce its own invariants. Say a handler must react to `OrderCreated` by reserving stock in a separate `Inventory` aggregate. Because the handler runs after the original commit, it cannot simply piggyback on the order's transaction. It loads the target aggregate, invokes its command, and commits its own unit of work:

```text
class ReserveStock:
  handle(OrderCreated event):
    inventory = inventoryRepo.forItems(event.orderId)
    inventory.reserve(event.orderId)     // enforces its own rules, may raise InventoryReserved
    unitOfWork.commit()
```

Two things follow from this, and it is important to be honest about them.

First, this is a **second transaction**, not an extension of the first. The order and the reservation are now committed separately, which means they are **eventually consistent**: for a brief window the order exists but stock is not yet reserved. Wrapping both aggregates in one transaction to avoid this is tempting, but it couples their lifecycles and breaks the [one-aggregate-per-transaction](https://www.dddcommunity.org/library/vernon_2011/) guideline that keeps aggregates independent. Prefer the two-transaction model and design for the gap rather than trying to close it.

Second, `Inventory` can raise its own event, `InventoryReserved`, which its own handlers then react to. This is the chaining from before, now spanning aggregates: one fact drives a policy, which produces the next fact. It is legitimate, but each hop widens the window of inconsistency and lengthens the causal chain, so keep the chains short and make each link idempotent.

Because handlers run after the commit, in their own transactions, and possibly in another process entirely, "just call the handler" stops being a safe assumption. If the reservation handler crashes before it commits, the order is already durable but the stock is never reserved. Getting that handoff right is a problem of its own.

This chaining is powerful and easy to abuse. A handler that issues a command can trigger another event, which triggers another handler, and the causal flow becomes hard to follow. Keep each handler small, let it do one thing, and let the resulting command decide on its own terms whether to raise a further event.

## Delivery is a separate problem

So far we've treated raising and handling as if they happen together, in memory, within the same unit of work. In practice they often don't. A handler may run in another process, on another machine, or long after the original operation, and any of the steps in between can fail. Once an event has to leave the process that raised it, delivery becomes a problem in its own right.

- Raising an in-process domain event does not guarantee that it will be delivered.
- Show the failure window in a naive implementation:

```text
save(order)
publish(OrderCreated)
```

- Explain the crash between these operations: the order exists, but email,
  fulfilment, and analytics may never react to it.
- Introduce a transactional outbox as the durable handoff:

```text
transaction:
  save(order)
  save(outboxMessage(OrderCreated))
```

- A separate publisher reads pending outbox messages and delivers them, retrying
  failures without losing the event.
- Explain that consumers must be idempotent because a message can be delivered
  more than once.
- State the limits: the outbox commits state and the event atomically, but does
  not provide immediate consistency or solve ordering in every context.
- Preview retries and observability.
- Link this discussion to the eventual-consistency follow-up post.

## Boundaries matter

- Domain events are primarily internal to a bounded context.
- Other contexts should receive an intentionally designed integration event, not
  the domain object's private representation.
- Explain translation, stable contracts, and why event names alone do not create
  decoupling.
- Domain and integration events
- Keeping the command and its consequences loosely coupled is also what later lets us turn these logical boundaries into physical ones, evolving toward a service-oriented architecture.

## Do not overdo it

- Do not create an event for every setter or database update.
- Prefer direct collaboration when there is no meaningful domain fact or policy
  boundary.
- Avoid using domain events to hide a dependency that should be explicit.

## Closing

- Recap: model meaningful facts first; choose handlers and transport second.
- Return to the example and show how its causal flow reads in domain language.
- Point to further areas worth exploring: eventual consistency as a first-class
  design concern, and event sourcing as a way to treat events as the source of truth.
- Tease the next post: making cross-boundary reactions reliable.
