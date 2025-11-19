# Job Chaining

TickerQ allows you to build parent–child job relationships using `TimeTickerEntity`, enabling multi-step workflows and conditional execution.

## Run Conditions

Child jobs decide when to run based on the parent job's status:

```csharp
public enum RunCondition
{
    OnSuccess,              // Run if parent succeeds
    OnFailure,              // Run if parent fails
    OnCancelled,            // Run if parent is cancelled
    OnFailureOrCancelled,   // Run if parent fails or cancelled
    OnAnyCompletedStatus,   // Run after parent completes (any status)
    InProgress              // Run in parallel with parent
}
```

## Creating Child Jobs

### Direct Child Assignment

```csharp
var parent = new TimeTickerEntity
{
    Function = "ProcessOrder",
    ExecutionTime = DateTime.UtcNow,
    // ... other properties
};

var child = new TimeTickerEntity
{
    Function = "SendConfirmationEmail",
    ParentId = parent.Id,
    RunCondition = RunCondition.OnSuccess,
    Request = TickerHelper.CreateTickerRequest(new EmailRequest { /* ... */ })
};

parent.Children.Add(child);

await _timeTickerManager.AddAsync(parent);
```

### Fluent Chain Builder

For more complex graphs, use `FluentChainTickerBuilder<TTimeTicker>`:

```csharp
using TickerQ.Utilities.Managers;

var job = FluentChainTickerBuilder<TimeTickerEntity>.BeginWith(parent =>
    {
        parent.SetFunction("ProcessOrder")
              .SetExecutionTime(DateTime.UtcNow)
              .SetRequest(new OrderRequest { OrderId = 123 });
    })
    .WithFirstChild(child =>
    {
        child.SetFunction("SendEmail")
             .SetRunCondition(RunCondition.OnSuccess);
    })
    .WithSecondChild(child =>
    {
        child.SetFunction("UpdateInventory")
             .SetRunCondition(RunCondition.OnSuccess);
    })
    .Build();

await _timeTickerManager.AddAsync(job);
```

## Execution Rules

Child jobs execute based on their `RunCondition` and the parent status. You do **not** need to set `ExecutionTime` on children; the scheduler takes care of when they should run relative to the parent.

- `InProgress` – runs while the parent is still executing.
- `OnSuccess` – runs when the parent finishes successfully (`Done` or `DueDone`).
- `OnFailure` – runs when the parent reaches `Failed`.
- `OnCancelled` – runs when the parent is `Cancelled`.
- `OnFailureOrCancelled` – runs on either `Failed` or `Cancelled`.
- `OnAnyCompletedStatus` – runs after any terminal status (Done, DueDone, Failed, Cancelled, Skipped).

## Nested Chains (Grandchildren)

The fluent builder supports up to three levels:

- Root (parent)
- Children (up to 5)
- Grandchildren (up to 5 per child)

```csharp
var job = FluentChainTickerBuilder<TimeTickerEntity>
    .BeginWith(parent => { /* ... */ })
    .WithFirstChild(child => { /* ... */ })
        .WithFirstGrandChild(grandchild =>
        {
            grandchild.SetFunction("NotifyAdmin")
                     .SetRunCondition(RunCondition.OnFailure);
        })
    .Build();
```

## When to Use Chaining

Use job chaining when you need:

- Multi-step workflows where later jobs depend on earlier outcomes.
- Fan-out/fan-in patterns (one parent, multiple children).
- Automatic compensating actions (failure or cancellation branches).
