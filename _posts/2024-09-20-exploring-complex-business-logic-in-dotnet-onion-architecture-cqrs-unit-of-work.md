---
layout: post
title:  "Exploring Complex Business Logic in .NET with Onion Architecture, CQRS, and Unit of Work"
tags: .net onion-architecture uni-of-Work
---

In my previous company, we primarily developed APIs in .NET, following the Onion Architecture pattern. I’ve continued using this architecture in my projects, applying the Command Query Responsibility Segregation (CQRS) pattern, where all database-related logic is encapsulated within commands. 

Until recently, I hadn’t encountered a situation where I needed to handle complex business logic involving the execution of multiple commands, followed by a single transaction commit. This post explores my solution to that problem.

> **Disclaimer**: Everything described here is for personal use and simplified for demonstration purposes. I’m continuously learning and experimenting, so this is not a tutorial or the "correct" way to do it.

## The Problem

In my **Application Layer**, I have various features that invoke appropriate commands. Each command creates its instance of the database context via a factory, executing either a simple save operation or a more complex transaction with a commit at the end. 

However, in scenarios where I needed to execute multiple commands before committing, I encountered challenges coordinating the commands and transaction handling. 

## Solution

Both solutions I explored leverage the **Unit of Work (UoW)** pattern, albeit in slightly different ways. 

### 1. Unit of Work without Repositories

The first solution involves creating a `UnitOfWork` class with `BeginTransaction()` and `SaveChanges()` methods. The interfaces are placed in the Application Layer, while the implementations reside in the Persistence Layer.

Here are simplified interfaces:

```csharp
public interface IUnitOfWork
{
    IDatabaseTransaction BeginTransaction();
    void SaveChanges();
}

public interface IDatabaseTransaction : IDisposable
{
    public void Commit();
    public void Rollback();
}
```

In this approach, each handler in the business logic passes the `IUnitOfWork` as a dependency, manually creating a transaction, committing it, and disposing of it correctly.

```csharp
public sealed class FirstHandler(
    IUnitOfWork unitOfWork,
    IDatabaseCommand firstCommand,
    IDatabaseCommand secondCommand) : IHandler
{
    public void Execute()
    {
        using var transaction = unitOfWork.BeginTransaction();

        firstCommand.Execute();
        unitOfWork.SaveChanges();

        secondCommand.Execute();
        unitOfWork.SaveChanges();

        transaction.Commit();
    }
}
```

   A sample command might look like this:

```csharp
public sealed class FirstCommand(AppDbContext appDbContext) : IDatabaseCommand
{
    public void Execute()
    {
        appDbContext.Books.Add(new Book { Title = "BookExample" });
    }
}
```

**Pros**:
- This approach reuses the database context across multiple commands, avoiding repeated context creation.
- There is no need for repositories.

**Cons**:
- The structure becomes repetitive, as each handler needs to manage the transaction lifecycle.
- There’s a risk of mishandling transaction disposal, leading to issues later.

Assuming the reader is familiar with Dependency Injection (DI) and its usage, I personally believe that `DbContext` inherently follows the Unit of Work pattern.

### 2. Solution with Database runner
Inspired by the book [Entity Framework Core in Action, Second Edition](https://www.manning.com/books/entity-framework-core-in-action-second-edition), I experimented with a cleaner solution using a **Database Runner**. This design encapsulates transaction handling and disposal in one place, keeping the Application Layer free of database-specific concerns. There is also a note that the saving changes should not be called in the application layer, which my first solution did by exposing the appropriate methods.

Here’s the interface:

```csharp
public interface IDatabaseRunner
{
    void Run();
}
```

And the implementation:

```csharp
public sealed class DatabaseRunner(
  AppDbContext appDbContext, 
  IDatabaseCommand firstCommand, 
  IDatabaseCommand secondCommand) : IDatabaseRunner
{
  public void Run()
  {
    using var transaction = appDbContext.Database.BeginTransaction();

    firstCommand.Execute();
    appDbContext.SaveChanges();

    secondCommand.Execute();
    appDbContext.SaveChanges();

    transaction.Commit();
  }
}
```
    
This allows the handler to be simplified:

```csharp
public sealed class SecondHandler(IDatabaseRunner databaseRunner) : IHandler
{
    public void Execute()
    {
        databaseRunner.Run();
    }
}
```

**Pros**:
- Transaction handling is encapsulated, simplifying the handlers.
- The Application Layer remains agnostic of database-specific logic, aligning better with Onion Architecture principles.

**Cons**:
- Although more elegant, this approach might need more flexibility for complex scenarios where commands depend on the results of previous ones.

## Conclusion

Both approaches achieve the goal of coordinating complex business logic and multiple command executions within a single transaction. However, I find the **Database Runner** pattern cleaner and more maintainable, especially in larger systems where separating concerns between layers is crucial. 

I’m now experimenting with extending this pattern to support a **command chain**, allowing each command to pass results to the next one for even more complex workflows.

**Note:** You can achieve the same result if you replace the database context dependency with the context factory and pass the instance down to the commands through method injection.

---

This was a simplified case study, and I’m constantly learning and refining my understanding of these patterns. If you’re working with similar architectures, I encourage you to experiment and adapt these ideas to suit your needs.

The blog post is on my [GitHub](https://github.com/DominikTher/dominikther.github.io), leave me a comment here. I used ChatGPT to align the document.
