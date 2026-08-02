---
title: "Making a Cross-System Process Correct"
date: 2026-07-25T16:00:00+03:00
draft: true
summary: "How a saga drives a multi-step process across decoupled, eventually consistent systems to a consistent end — completing it, or compensating it."
tags: ["software-architecture", "domain-driven-design", "distributed-systems", "sagas"]
editLink: "https://github.com/deniskyashif/blog/blob/main/content/posts/ddd-sagas.md"
---

This continues [Couple the Experience, Decouple the Systems](/posts/ddd-integration-topology/), which introduced a process spanning decoupled systems, named the saga as its owner, and stopped at *who owns the flow*. Here we answer *how that owner works*.

The systems are decoupled and eventually consistent. There is no distributed transaction across them. So how does a multi-step business process still reach a correct, complete outcome?

> A saga drives a process to a consistent end state: complete it, or compensate it.

## The setup, recapped

- Decoupled systems, eventually consistent — the world established across the series.
- A cross-system process is a **sequence of local transactions**, each committed independently in its own context.
- The saga owns that sequence. It is a **spoke, not the hub** — it makes decisions; the hub only moves messages. (Reinforce; do not re-derive.)
- The problem this creates: if a later step fails, earlier steps are already committed. There is nothing to roll back to.

## Compensation, not rollback

- A committed local transaction cannot be rolled back; it must be **semantically undone** by a new forward action.
- Contrast rollback (database, automatic, invisible) with compensation (business action, explicit, may itself fail).
- Example: `reserveStock` is compensated by `releaseStock`, not by "un-committing".
- Not every action has a clean inverse — some effects are irreversible (an email sent, a payment captured). Discuss:
  - designing steps so compensations exist,
  - ordering irreversible steps late,
  - compensation that mitigates rather than perfectly reverses.
- **Forward recovery vs. backward recovery:** sometimes the right answer is to retry to completion, not to unwind. When to choose each.

## Orchestration vs. choreography, in mechanism

- Deepen the axis introduced in the previous article — here, how each actually works.
- **Orchestration:** a process-manager spoke issues commands step by step, awaits results, and decides the next move (including compensation). Explicit flow, single place to see and reason about it, one coordinating dependency.
- **Choreography:** each spoke reacts to the prior event and emits the next; failure emits a "failed" event that upstream spokes listen for and compensate themselves. No coordinator; logic distributed.
- Tradeoffs under failure: visibility, coupling, where compensation logic lives, how hard it is to reason about the whole.
- Rule of thumb: orchestration when the flow is complex or must be explicit; choreography when steps are few and independence matters most.

## Process state and the stuck saga

- A saga is a **state machine**: it must persist where the process is, so it survives crashes and can resume.
- Where the state lives (the process-manager spoke's own store).
- **Timeouts:** a step that never responds must not hang forever. Model deadlines and timeout transitions.
- The **stuck saga**: partial completion with no progress. Detection, alerting, and deliberate intervention.
- Correlation: tying every step and message back to the one saga instance.

## Idempotency and why retry is not compensation

- Steps and compensations run over at-least-once delivery — both must be **idempotent**.
- A compensation is a forward action that can *itself* fail and be retried; design it to be safe to repeat.
- **Message retry ≠ compensation** (payoff of the distinction from the topology article): the hub redelivering a message drives one hop to success; it has no notion of the whole process and cannot undo a completed step. Only the saga compensates.
- Idempotent compensations + reliable delivery together are what let the saga converge.

## Know when not to reach for a saga

- If a process fits in one aggregate/one transaction, you don't need a saga — don't manufacture distributed steps.
- If steps are genuinely independent (no cross-step consistency requirement), plain event handlers suffice.
- A saga is for **multi-step processes that must complete or unwind as a unit** across boundaries.

## Closing

- Recap: local transactions + compensation, owned by a process-manager spoke, driven by state, made safe by idempotency and reliable delivery.
- This is how eventual consistency becomes *correctness*: the process converges to complete or compensated, never stuck silently.
- Close the series arc: fact (domain events) → contract (integration events) → topology and experience → process correctness under failure.
