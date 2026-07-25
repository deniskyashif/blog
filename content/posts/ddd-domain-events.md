---
title: "Making Things Happen: A Practical Guide to Domain Events"
date: 2026-07-25T08:14:19+03:00
draft: false
summary: "Events modeling in practice."
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

It also couples the customer-facing request to external systems. The order may be saved successfully while email delivery, fulfilment notification, or analytics fails. These operations cannot usually be made atomic with the order write, despite being grouped in the same script. Reporting the whole operation as failed can mislead the customer into placing a duplicate order; reporting it as successful can leave follow-up work unfinished.

Each consequence needs its own error handling, retries, and fault tolerance. There must be a better way to handle this. To get there, we must first ask: 

> What happened, and who needs to know?

## A domain event is a past-tense fact

An order being created is a fact in the domain. A domain event records such a fact in the language the business uses. It is not an instruction to perform work: `SendConfirmationEmail` describes a technical task, while `OrderCreated` describes something that has already happened.

Events are named in the past tense because they are historical records. No subscriber can reject or undo `OrderCreated`; it can only decide how to react to it. For the same reason, an event should be immutable. If the order later changes, that is a new fact and may warrant a new event rather than changing the record of what happened before.

An event carries the information a recipient needs to understand the fact without reaching back into the domain model. At a minimum, that usually means the identity of the affected entity, the relevant values at the time, and when the event occurred.

```text
OrderCreated
  orderId
  customerId
  createdAt
```

## Raise events where the decision is made

- Put the event at the aggregate or domain-model operation that enforces the
  invariant.
- Show the sequence: validate rule, change state, record event.
- Explain why application services should not infer domain events by observing
  persistence changes.
- Discuss events as part of the model's public language, not incidental logs.

## Handling events

- Separate raising an event from handling it.
- Use handlers for follow-up policies: allocate stock, start fulfilment, notify a
  customer, update a read model.
- Keep the command path understandable: a handler is a policy reacting to a fact.
- Call out that an event handler can issue a command, but should not bypass the
  receiving aggregate's rules.

## Commands express intent; events record facts

- A command asks the model to do something and can be rejected: `ConfirmOrder`.
- An event states something that already happened: `OrderConfirmed`.
- Commands are imperative and normally have one responsible handler; events are
  declarative, past tense, and may have zero or many subscribers.
- Commands enforce rules; events let policies react to their outcome.
- Explain why an event must not be used as an asynchronous command in disguise.

## Delivery is a separate problem

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

## Eventual Consistency and Event Sourcing

## When not to use them

- Do not create an event for every setter or database update.
- Prefer direct collaboration when there is no meaningful domain fact or policy
  boundary.
- Avoid using domain events to hide a dependency that should be explicit.

## Closing

- Recap: model meaningful facts first; choose handlers and transport second.
- Return to the example and show how its causal flow reads in domain language.
- Tease the next post: making cross-boundary reactions reliable.
