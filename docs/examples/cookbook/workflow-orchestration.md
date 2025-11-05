# Workflow Orchestration

Patterns for chaining and orchestrating multiple jobs.

## Sequential Job Chain

Create a parent-child job chain:

```csharp
public async Task ProcessOrderAsync(Order order)
{
    // Create job chain using fluent builder
    var workflow = FluentChainTickerBuilder<TimeTickerEntity>
        .BeginWith(parent =>
        {
            parent.SetFunction("ValidateOrder")
                  .SetExecutionTime(DateTime.UtcNow)
                  .SetRequest(new OrderRequest { OrderId = order.Id });
        })
        .WithFirstChild(child =>
        {
            child.SetFunction("ProcessPayment")
                 .SetRunCondition(RunCondition.OnSuccess)
                 .SetExecutionTime(DateTime.UtcNow.AddMinutes(1));
        })
        .WithSecondChild(child =>
        {
            child.SetFunction("SendConfirmation")
                 .SetRunCondition(RunCondition.OnSuccess)
                 .SetExecutionTime(DateTime.UtcNow.AddMinutes(2));
        })
        .Build();
    
    await _timeTickerManager.AddAsync(workflow);
}
```

## Conditional Workflow

Schedule child jobs conditionally based on parent result:

```csharp
// Parent job decides which child to run
[TickerFunction("ProcessOrder")]
public async Task ProcessOrder(
    TickerFunctionContext<OrderRequest> context,
    CancellationToken cancellationToken)
{
    var order = context.Request;
    
    // Process order
    var result = await ProcessOrderAsync(order, cancellationToken);
    
    // Schedule child based on result
    if (result.RequiresShipping)
    {
        await _timeTickerManager.AddAsync(new TimeTickerEntity
        {
            Function = "CreateShipment",
            ParentId = context.Id,
            RunCondition = RunCondition.OnSuccess,
            ExecutionTime = DateTime.UtcNow.AddMinutes(5),
            Request = TickerHelper.CreateTickerRequest(new ShipmentRequest
            {
                OrderId = order.OrderId
            })
        });
    }
}
```

## Parallel Execution

Execute multiple independent jobs in parallel:

```csharp
[TickerFunction("ProcessOrder")]
public async Task ProcessOrder(
    TickerFunctionContext<OrderRequest> context,
    CancellationToken cancellationToken)
{
    var orderId = context.Request.OrderId;
    
    // Schedule parallel child jobs
    var tasks = new[]
    {
        _timeTickerManager.AddAsync(new TimeTickerEntity
        {
            Function = "SendNotification",
            ParentId = context.Id,
            RunCondition = RunCondition.InProgress, // Run in parallel
            ExecutionTime = DateTime.UtcNow
        }),
        _timeTickerManager.AddAsync(new TimeTickerEntity
        {
            Function = "UpdateInventory",
            ParentId = context.Id,
            RunCondition = RunCondition.InProgress,
            ExecutionTime = DateTime.UtcNow
        }),
        _timeTickerManager.AddAsync(new TimeTickerEntity
        {
            Function = "LogActivity",
            ParentId = context.Id,
            RunCondition = RunCondition.InProgress,
            ExecutionTime = DateTime.UtcNow
        })
    };
    
    await Task.WhenAll(tasks);
}
```

## Error Recovery Workflow

Schedule recovery job on failure:

```csharp
[TickerFunction("ProcessPayment")]
public async Task ProcessPayment(
    TickerFunctionContext<PaymentRequest> context,
    CancellationToken cancellationToken)
{
    try
    {
        await ProcessPaymentAsync(context.Request, cancellationToken);
    }
    catch (PaymentException ex)
    {
        // Schedule recovery job
        await _timeTickerManager.AddAsync(new TimeTickerEntity
        {
            Function = "NotifyPaymentFailure",
            ParentId = context.Id,
            RunCondition = RunCondition.OnFailure,
            ExecutionTime = DateTime.UtcNow.AddMinutes(1),
            Request = TickerHelper.CreateTickerRequest(new NotificationRequest
            {
                OrderId = context.Request.OrderId,
                Error = ex.Message
            })
        });
        
        throw; // Re-throw to mark parent as failed
    }
}
```

## See Also

- [Job Fundamentals](../../concepts/job-fundamentals#job-chaining) - Understanding job relationships
- [Error Handling](../../concepts/error-handling) - Error recovery patterns
- [Batch Processing](./batch-processing) - Processing multiple items

