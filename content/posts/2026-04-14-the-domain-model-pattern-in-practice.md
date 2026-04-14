---
title: "The Domain Model Pattern in Practice"
date: 2026-04-04T20:11:06+03:00
draft: false
summary: "How a rich domain model keeps the business invariants consistent, testable, and easier to evolve."
tags: ["software-architecture", "domain-driven design", "csharp"]
editLink: "https://github.com/deniskyashif/blog/blob/main/content/posts/2026-04-14-the-domain-model-pattern-in-practice.md"
---

# Transaction Scripts

Most web applications follow a layered architecture: `Presentation → Service → Data`. The data layer holds plain entities — POCOs with public getters and setters — plus data access infrastructure. Business logic lives in the service layer, one method per use case. The presentation layer stays thin and simply delegates.

<img src="/images/posts/2026-04-14-the-domain-model-in-practice/3-tier-arch.png" alt="3 Tier Architecture" />

Let's look at this simplified example of a banking application.

```csharp
// Data layer — plain entities, no behavior
public class BankAccount
{
    public int Id { get; set; }
    public decimal Balance { get; set; }
    public bool IsFrozen { get; set; }
}

// Service layer — all business logic lives here
public class TransferService(IAccountRepository accounts, IUnitOfWork uow)
{
    public void Transfer(int fromId, int toId, decimal amount)
    {
        uow.ExecuteInTransaction(() =>
        {
            if (amount <= 0)
                throw new ArgumentException("Amount must be positive.");

            var from = accounts.GetById(fromId);
            var to = accounts.GetById(toId);

            if (from.IsFrozen || to.IsFrozen)
                throw new InvalidOperationException("Frozen accounts cannot transact.");

            if (from.Balance < amount)
                throw new InvalidOperationException("Insufficient funds.");

            from.Balance -= amount;
            to.Balance += amount;
        });
    }
}
```

This is also called a [Transaction Script](https://martinfowler.com/eaaCatalog/transactionScript.html): a single procedure per use case that reads data, runs the logic, and writes back the result. It looks clean and sensible — straightforward and easy to follow. But as business complexity grows, the cracks start to show.

One of the earliest signs of trouble is the scattering of cross-cutting business rules. Consider a rule like: *"a frozen account cannot perform any money operation."* In a transaction-script design, every service method that needs to enforce this rule must check it independently:

```csharp
public class TransferService(IAccountRepository accounts)
{
    public void Transfer(int fromId, int toId, decimal amount)
    {
        // Guard duplicated across services
        var from = accounts.GetById(fromId);
        var to = accounts.GetById(toId);
        if (from.IsFrozen || to.IsFrozen)
            throw new InvalidOperationException("Frozen accounts cannot transact.");

        // ... transfer logic
    }
}

public class AtmWithdrawalService(IAccountRepository accounts)
{
    public void WithdrawCash(int accountId, decimal amount)
    {
        // Same guard, again
        var account = accounts.GetById(accountId);
        if (account.IsFrozen)
            throw new InvalidOperationException("Frozen accounts cannot transact.");

        // ... ATM withdrawal logic
    }
}
```

This duplication is easy to miss and hard to fix. When the rule changes — say, *"frozen accounts may now receive money but not send it*" — we have to track down every place it was applied. Forget one, and we have a bug. A real application has hundreds of such rules; multiply that by the number of enforcement points and the codebase becomes fragile very fast.

This structure — objects that carry data but no behavior, with all logic pushed into a service layer — is called an [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html). The entities look like a domain model on the surface, but they lack the one thing that makes object-oriented design worthwhile: behavior.

Additionally, entities that expose public setters allow any part of the codebase to mutate state directly. That makes business invariants harder to enforce consistently. For example, any caller can set `Balance` to an arbitrary value without going through withdrawal/deposit rules (insufficient funds checks, positive amount checks, auditing, etc.).

# Implementing the Domain Model Pattern

A domain model encapsulates both data and behavior in the same objects — it is where business invariants live and are consistently enforced. Let's refactor the previous example:

```csharp
public sealed class BankAccount
{
    public int Id { get; }
    public decimal Balance { get; private set; }
    public bool IsFrozen { get; private set; }

    public BankAccount(int id, decimal balance)
    {
        Id = id;
        Balance = balance;
        IsFrozen = false;
    }

    public void FreezeForComplianceReview()
    {
        IsFrozen = true;
    }

    public void EnsureCanSendMoney()
    {
        if (IsFrozen)
            throw new InvalidOperationException("Frozen accounts cannot send money.");
    }

    public void Withdraw(decimal amount)
    {
        EnsureCanSendMoney();

        if (amount <= 0)
            throw new ArgumentException("Amount must be positive.");

        if (Balance < amount)
            throw new InvalidOperationException("Insufficient funds.");

        Balance -= amount;
    }

    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive.");

        Balance += amount;
    }
}

// Thin application service: orchestration only.
public sealed class TransferMoneyHandler(IAccountRepository accounts, IUnitOfWork uow)
{
    public void Handle(int fromId, int toId, decimal amount)
    {
        uow.ExecuteInTransaction(() =>
        {
            var from = accounts.GetById(fromId);
            var to = accounts.GetById(toId);

            from.Withdraw(amount);
            to.Deposit(amount);
        });
    }
}
```

The rules now live in the domain objects and are enforced in one place. The service layer is pure orchestration. A rich domain model makes business rules evident and explicit, which makes them easier to reason about, understand, and change.

Because business behavior changes often, the domain layer should be easy to evolve and test — which means keeping it free of dependencies on web frameworks, ORMs, or transport concerns.

## Dependency Inversion

In the transaction-script setup, the data layer often contains both repository contracts and concrete ORM implementations. In a domain-model design, ORM-specific code does not belong in the domain layer. The [Dependency Inversion](https://en.wikipedia.org/wiki/Dependency_inversion_principle) principle is to keep abstractions in the domain (i.e., repository interfaces) and place implementations in infrastructure, which references the domain, not the other way around.

## The Pipeline

The dependency flow becomes: `Presentation and Infrastructure → Service → Domain` — low-level concerns at the edges, business behavior at the base. The service layer loads aggregates, invokes domain behavior, and persists results.

<img src="/images/posts/2026-04-14-the-domain-model-in-practice/ddd-arch.png" alt="DDD Architecture" />


## A Pragmatic Approach

Not all business rules need to live in the domain. For example, an account number should be unique. There is no need to pre-check uniqueness in domain code before every create operation; this is a great fit for a database constraint. Some rules can live in the DB, and that is perfectly fine. Business rules that determine whether an action is allowed are better candidates for the domain model.

# When to avoid a Domain Model

A rich domain model is not about adding classes for the sake of architecture. It is about putting business behavior where it belongs so invariants are enforced once, close to the data they protect. As the system grows, this pays off with better testability, clearer change boundaries, and fewer hidden rule duplications.

That said, domain modeling is an investment. If our use cases are mostly CRUD and unlikely to evolve, the extra design surface can become an unnecessary overhead.

Avoid a full domain model when:

- The application is simple CRUD with minimal or rarely changing business rules.
- Speed of delivery matters more than long-term design.

Consider a domain model when:

- Business rules are numerous, cross-cutting, and change frequently.
- Incorrect behavior has high business cost (money movement, compliance, risk decisions).

The practical takeaway is to start with behavior, not the entity-relationship diagram. Spend more time on concrete use cases and the rules they require before modeling relationships between entities. Keep services thin, keep infrastructure at the edges, and let the model absorb complexity where it naturally belongs. We likely won't get the model right on the first try, and that is perfectly normal. Iterating on it with business stakeholders is what leads to a better design.