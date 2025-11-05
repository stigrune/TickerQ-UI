# Job Fundamentals

This guide covers the fundamental concepts of TickerQ jobs, including job chaining, priorities, and advanced scheduling patterns.

## Job Function Basics

### Function Signature

All TickerQ job functions must follow this pattern:

```csharp
[TickerFunction("FunctionName")]
public async Task FunctionName(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    // Job logic here
}
```

### Function Context

The `TickerFunctionContext` provides access to job metadata:

```csharp
public class TickerFunctionContext
{
    public Guid Id { get; }                    // Job ID
    public TickerType Type { get; }            // TimeTicker or CronTicker
    public int RetryCount { get; }             // Current retry attempt
    public bool IsDue { get; }                 // Whether job is due
    public string FunctionName { get; }        // Function name
    public CronOccurrenceOperations CronOccurrenceOperations { get; } // Cron-specific operations
    
    public void CancelOperation()               // Cancel the current job
}
```

### Typed Request Context

For easier request handling, use the typed version:

```csharp
[TickerFunction("ProcessOrder")]
public async Task ProcessOrder(
    TickerFunctionContext<OrderRequest> context,
    CancellationToken cancellationToken)
{
    var order = context.Request; // Directly typed request
    await ProcessOrderAsync(order, cancellationToken);
}
```

## Job Priorities

Control execution order with priorities. Jobs with higher priority execute first.

### Priority Levels

```csharp
public enum TickerTaskPriority
{
    LongRunning,  // Executes in separate thread pool
    High,         // Highest priority
    Normal,       // Default priority
    Low           // Lowest priority
}
```

### Setting Priority

#### In Function Attribute

```csharp
[TickerFunction("CriticalTask", taskPriority: TickerTaskPriority.High)]
public async Task CriticalTask(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    // This job will execute before Normal and Low priority jobs
}
```

#### When Scheduling (TimeTicker Only)

Priority is determined by the function's attribute setting. TimeTickers inherit priority from their function definition.

### Priority Execution Order

1. **High Priority** jobs execute first
2. **Normal Priority** jobs execute next
3. **Low Priority** jobs execute last
4. **LongRunning** jobs execute in a separate thread pool

## Job Chaining

TimeTicker supports parent-child job relationships, allowing you to create complex workflows.

### Child Job Conditions

Child jobs can run based on parent job status:

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

### Creating Child Jobs

#### Method 1: Direct Child Assignment

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
    ExecutionTime = DateTime.UtcNow.AddMinutes(1),
    ParentId = parent.Id,
    RunCondition = RunCondition.OnSuccess,
    Request = TickerHelper.CreateTickerRequest(new EmailRequest { /* ... */ })
};

parent.Children.Add(child);

await _timeTickerManager.AddAsync(parent);
```

#### Method 2: Using Fluent Chain Builder

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
             .SetRunCondition(RunCondition.OnSuccess)
             .SetExecutionTime(DateTime.UtcNow.AddMinutes(1));
    })
    .WithSecondChild(child =>
    {
        child.SetFunction("UpdateInventory")
             .SetRunCondition(RunCondition.OnSuccess)
             .SetExecutionTime(DateTime.UtcNow.AddMinutes(2));
    })
    .Build();

await _timeTickerManager.AddAsync(job);
```

### Child Job Execution

Child jobs execute based on their `RunCondition`:

- **InProgress**: Runs in parallel with parent (when parent status is `InProgress`)
- **OnSuccess**: Runs after parent succeeds (status `Done` or `DueDone`)
- **OnFailure**: Runs after parent fails
- **OnCancelled**: Runs if parent is cancelled
- **OnFailureOrCancelled**: Runs if parent fails or is cancelled
- **OnAnyCompletedStatus**: Runs after parent reaches any terminal state

### Nested Children (Grandchildren)

The fluent builder supports up to 3 levels:
- Root (parent)
- Children (max 5)
- Grandchildren (max 5 per child)

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

## Job Status Lifecycle

Understanding job status helps with monitoring and debugging:

```
Idle → Queued → InProgress → Done/Failed/Cancelled/Skipped
```

### Status Definitions

| Status | Description |
|--------|-------------|
| `Idle` | Job created but not yet queued |
| `Queued` | Job is in queue waiting to execute |
| `InProgress` | Job is currently executing |
| `Done` | Job completed (executed after due time) |
| `DueDone` | Job completed (executed at due time) |
| `Failed` | Job failed after all retries |
| `Cancelled` | Job was cancelled |
| `Skipped` | Job was skipped (e.g., duplicate prevented) |

## Job Cancellation

Jobs can be cancelled from within the function:

```csharp
[TickerFunction("LongRunningTask")]
public async Task LongRunningTask(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    // Check cancellation token
    if (cancellationToken.IsCancellationRequested)
        return;
    
    // Or cancel programmatically
    if (shouldCancel)
        context.CancelOperation();
}
```

## Batch Operations

TimeTickers can be grouped as batches with a parent:

```csharp
var batchParent = new TimeTickerEntity
{
    Function = "ProcessBatch",
    ExecutionTime = DateTime.UtcNow,
    // ... properties
};

var batchChild1 = new TimeTickerEntity
{
    Function = "ProcessItem1",
    ParentId = batchParent.Id,
    BatchParent = batchParent.Id,
    RunCondition = RunCondition.OnSuccess
};

var batchChild2 = new TimeTickerEntity
{
    Function = "ProcessItem2",
    ParentId = batchParent.Id,
    BatchParent = batchParent.Id,
    RunCondition = RunCondition.OnSuccess
};
```

## Request Data Handling

### Creating Request Data

```csharp
// Simple type
var request = TickerHelper.CreateTickerRequest(new EmailRequest
{
    To = "user@example.com",
    Subject = "Welcome",
    Body = "Thanks for joining!"
});

// Complex type
var request = TickerHelper.CreateTickerRequest(new OrderProcessingRequest
{
    OrderId = 12345,
    Items = new[] { /* ... */ },
    CustomerId = 67890
});
```

### Reading Request Data

```csharp
// In job function
var request = await TickerRequestProvider.GetRequestAsync<EmailRequest>(
    context, 
    cancellationToken
);

// Or use typed context
[TickerFunction("ProcessOrder")]
public async Task ProcessOrder(
    TickerFunctionContext<OrderRequest> context,
    CancellationToken cancellationToken)
{
    var order = context.Request; // Already typed
}
```

### Request Serialization

TickerQ uses GZip-compressed JSON serialization for request data. The `TickerHelper` handles all serialization automatically.

## Job Identification

### Function Name

The function name must match exactly between:
- `[TickerFunction("FunctionName")]` attribute
- `Function` property when scheduling

### Job ID

Each job gets a unique `Guid` identifier:

```csharp
var ticker = new TimeTickerEntity
{
    Id = Guid.NewGuid(), // Optional - auto-generated if not set
    Function = "MyJob",
    // ...
};
```

## Next Steps

- [Configure Retry Policies](/concepts/error-handling)
- [Learn About Entity Framework Integration](/features/entity-framework)
- [Explore Dashboard Features](/features/dashboard)
