---
title: "Domain-Driven Design: Lean Aggregates"
date: 2026-04-04T17:39:14+03:00
draft: false
summary: "Design aggregates around true consistency boundaries, not around all related data, to keep domain logic clear and write paths performant as the system grows."
tags: ["software-architecture", "domain-driven-design", "csharp"]
editLink: "https://github.com/deniskyashif/blog/blob/main/content/posts/2026-04-04-ddd-lean-aggregates.md"
---

In this article we'll walk through a common DDD antipattern that's easy to fall into. We'll see how to refactor toward a better design and, more importantly, how to think differently to avoid it in the first place.

## Fat Aggregates

When modeling domain objects, we tend to think about *what they contain* before thinking about *what they do*. This is natural - but it's also where things can go wrong.

Say we're building a project management system. `Project` is our aggregate root - it governs the project's lifecycle and enforces its business rules. A project has tasks, team members, attached documents, and more - so we pull them all in:

```cs
public class Project
{
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public ProjectStatus Status { get; private set; }

    private readonly List<ProjectTask> _tasks = new();
    private readonly List<TeamMember> _members = new();
    private readonly List<Document> _documents = new();

    public IReadOnlyList<ProjectTask> Tasks => _tasks;
    public IReadOnlyList<TeamMember> Members => _members;
    public IReadOnlyList<Document> Documents => _documents;
}
```

Since we're modeling a rich Domain Model - not a procedural design where classes are just data holders and all logic lives in service classes - `Project` also encapsulates behavior:

```cs
public class Project
{
    // ...
    
    public void AssignTask(string title, TeamMember assignee)
    {
        if (_members.All(m => m.Id != assignee.Id))
            throw new DomainException("Assignee must be a project member.");

        if (Status == ProjectStatus.Completed)
            throw new DomainException("Cannot add tasks to a completed project.");

        _tasks.Add(new ProjectTask(title, assignee));
    }

    public void AttachDocument(Document document)
    {
        if (Status == ProjectStatus.Completed)
            throw new DomainException("Cannot attach documents to a completed project.");

        _documents.Add(document);
    }

    public void Complete()
    {
        if (_tasks.Any(t => !t.IsDone))
            throw new DomainException("Cannot complete project while there are open tasks.");

        Status = ProjectStatus.Completed;
    }
}
```

This looks correct: the aggregate protects its invariants and prevents invalid state. But problems start to surface as the system grows.

In DDD, repositories load the full aggregate before any write, so all invariants can be verified. If `Project` is the central aggregate, then every write operation - assigning a task, adding a member, attaching a document, adjusting the budget - must load the entire object graph. This causes table locks, performance bottlenecks, and contention between concurrent operations.

```cs
public class ProjectRepository
{
    // ...

    public async Task<Project> GetByIdAsync(Guid id)
    {
        return await _db.Projects
            .Include(p => p.Tasks)
            .Include(p => p.Members)
            .Include(p => p.Documents)
            .FirstOrDefaultAsync(p => p.Id == id)
            ?? throw new NotFoundException($"Project {id} not found.");
    }
}
```

Worse, every new business rule gets added to `Project`. The aggregate grows into a God class: bloated, hard to reason about, and increasingly risky to change.

## Trimming Down the Aggregate

> An aggregate should contain all the information about the business situation and not more.

The key question is simple: **what must be consistent in the same transaction?** If two pieces of data do not need to change together, they probably should not live in the same aggregate. For example, attaching a document does not need tasks and team members loaded into memory. That suggests a refactor: keep `Project` lean, and move documents and tasks into separate aggregates that reference `ProjectId`.

```cs
public class Project
{
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public ProjectStatus Status { get; private set; }

    public void Complete() => Status = ProjectStatus.Completed;
}

public class Document
{
    public Guid Id { get; private set; }
    public Guid ProjectId { get; private set; }
    public string Name { get; private set; }

    Document(Guid projectId, string name)
    {
        Id = Guid.NewGuid();
        ProjectId = projectId;
        Name = name;
    }

    public static Document Attach(Guid projectId, string name)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new DomainException("Document name is required.");

        return new Document(projectId, name.Trim());
    }
}
```

In this first version, `Document` is fully responsible for the attach operation because it has all the information needed for that rule.

```cs
var document = Document.Attach(projectId, fileName);
await _documentRepository.AddAsync(document);
```

Now assume we add a new rule: *we cannot attach a document if the project is completed*. This rule was easy to enforce in the fat aggregate - `Project` had all the data in memory, so a simple status check was enough. After the split, this rule spans two aggregates (`Project` and `Document`), so it should not live inside `Document`. Does that mean we made a mistake by splitting? No - we just need to be explicit about where cross-aggregate rules live.

That is the point where we split responsibilities explicitly:

1. A **domain service** that encapsulates the cross-aggregate business rule.
2. An **application service** that orchestrates repositories and persistence.

Domain service:

```cs
public class DocumentDomainService
{
    public void EnsureCanAttach(Project project)
    {
        if (project.Status == ProjectStatus.Completed)
            throw new DomainException("Cannot attach documents to a completed project.");
    }
}
```

Application service:

```cs
public class DocumentApplicationService
{
    readonly IProjectRepository _projectRepository;
    readonly IDocumentRepository _documentRepository;
    readonly DocumentDomainService _documentDomainService;

    public DocumentApplicationService(
        IProjectRepository projectRepository,
        IDocumentRepository documentRepository,
        DocumentDomainService documentDomainService)
    {
        _projectRepository = projectRepository;
        _documentRepository = documentRepository;
        _documentDomainService = documentDomainService;
    }

    public async Task AttachDocument(Guid projectId, string fileName)
    {
        var project = await _projectRepository.GetByIdAsync(projectId);
        _documentDomainService.EnsureCanAttach(project);

        var document = Document.Attach(projectId, fileName);
        await _documentRepository.AddAsync(document);
    }
}
```

Usage:

```cs
await _documentApplicationService.AttachDocument(projectId, fileName);
```

This keeps the model honest: local invariants stay in aggregates, cross-aggregate rules live in domain services, and orchestration lives in application services.

This approach is better for several reasons.

1. **Performance**: each write touches fewer tables and rows, so queries are faster and locks are smaller.
2. **Concurrency**: two users can update different parts of the same project (for example tasks and documents) with less contention.
3. **Maintainability**: business rules are grouped by true consistency boundary, which keeps each model smaller and easier to reason about.

There is one implication: some rules that used to live in one in-memory object now span aggregates. We enforce those rules explicitly during command handling (typically through a small application-service flow that invokes domain logic). In other words, we trade a little orchestration complexity for a much healthier model under real production load.

## Common Pitfalls

Refactoring toward lean aggregates helps a lot, but there are a few traps teams often fall into.

1. **Putting orchestration into domain services**

A domain service should express business rules. Loading repositories, managing transactions, and persisting changes is orchestration and belongs to an application service (or command handler).

2. **Splitting by tables, not by consistency boundaries**

If we split only because "this table is big", we may end up with models that still need to change together. A better signal is business consistency: if two things must always be valid together in one transaction, they likely belong to the same aggregate.

Example: imagine we split `ProjectTask` and `TaskAssignment` into separate aggregates, but the business rule says that whenever a task is moved to `InProgress`, an assignee must exist in the same transaction. If this must always happen together, the split is likely wrong (or the boundary needs to be revisited).

```cs
public class ProjectTask
{
    public Guid Id { get; private set; }
    public TaskStatus Status { get; private set; }

    public void Start()
    {
        Status = TaskStatus.InProgress;
    }
}

public class TaskAssignment
{
    public Guid TaskId { get; private set; }
    public Guid AssigneeId { get; private set; }
}
```

With this split, `ProjectTask.Start()` can be called even when there is no `TaskAssignment`. If the rule says that "InProgress" always requires an assignee, then these two concepts are likely part of the same consistency boundary.

Refactored version with an enforced consistency boundary:

```cs
public class ProjectTask
{
    public Guid Id { get; private set; }
    public Guid? AssigneeId { get; private set; }
    public TaskStatus Status { get; private set; }

    public void AssignTo(Guid assigneeId)
    {
        AssigneeId = assigneeId;
    }

    public void Start()
    {
        if (AssigneeId is null)
            throw new DomainException("A task must be assigned before it can start.");

        Status = TaskStatus.InProgress;
    }
}
```

Now the invariant is protected inside one aggregate: the task cannot enter `InProgress` without an assignee.

Common symptoms of a wrong split:

- We often load multiple aggregates just to enforce a single invariant.
- We keep adding cross-aggregate checks for operations that conceptually feel like one consistency rule.
- We see temporary invalid states that business users consider unacceptable.
- We frequently need distributed transactions for one user action.

## Conclusion

Lean aggregates start with a simple discipline: **model intrinsic behavior first, and include only the data required to protect that behavior**. An aggregate is not a container for all related data; it is a consistency boundary around rules that must hold together.

When we design boundaries this way, the model becomes smaller, clearer, and easier to evolve. We avoid God aggregates, reduce contention in write paths, and keep domain rules explicit where they belong.