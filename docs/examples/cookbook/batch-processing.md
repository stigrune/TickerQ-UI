# Batch Processing

Process large datasets efficiently in chunks.

## Process Items in Batches

Process items in configurable batch sizes:

```csharp
[TickerFunction("ProcessBatch")]
public async Task ProcessBatch(
    TickerFunctionContext<BatchRequest> context,
    CancellationToken cancellationToken)
{
    var batch = context.Request;
    var batchSize = 100;
    
    for (int i = 0; i < batch.Items.Count; i += batchSize)
    {
        var chunk = batch.Items.Skip(i).Take(batchSize).ToList();
        
        await ProcessChunkAsync(chunk, cancellationToken);
        
        // Check cancellation
        cancellationToken.ThrowIfCancellationRequested();
    }
}
```

## Schedule Multiple Jobs for Batch

Schedule individual jobs for each item:

```csharp
public async Task ScheduleBatchProcessing(List<Item> items)
{
    var jobs = items.Select(item => new TimeTickerEntity
    {
        Function = "ProcessItem",
        ExecutionTime = DateTime.UtcNow,
        Request = TickerHelper.CreateTickerRequest(new ItemRequest
        {
            ItemId = item.Id
        }),
        Description = $"Process item {item.Id}"
    }).ToList();
    
    // Add all at once
    var result = await _timeTickerManager.AddBatchAsync(jobs, cancellationToken);
    
    if (result.IsSucceeded)
    {
        _logger.LogInformation("Scheduled {Count} batch jobs", result.Result.Count);
    }
}
```

## Process with Progress Tracking

Track progress through large batches:

```csharp
[TickerFunction("ProcessLargeBatch")]
public async Task ProcessLargeBatch(
    TickerFunctionContext<LargeBatchRequest> context,
    CancellationToken cancellationToken)
{
    var request = context.Request;
    var batchSize = 500;
    var totalItems = request.ItemIds.Count;
    var processed = 0;
    
    for (int i = 0; i < totalItems; i += batchSize)
    {
        var chunk = request.ItemIds.Skip(i).Take(batchSize).ToList();
        
        await ProcessChunkAsync(chunk, cancellationToken);
        
        processed += chunk.Count;
        _logger.LogInformation("Processed {Processed}/{Total} items", processed, totalItems);
        
        // Yield to allow other jobs to run
        await Task.Delay(100, cancellationToken);
    }
}
```

## Parallel Batch Processing

Process multiple batches in parallel:

```csharp
[TickerFunction("ProcessParallelBatches")]
public async Task ProcessParallelBatches(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    var batches = await GetBatchesAsync(cancellationToken);
    var maxConcurrency = 5;
    
    var semaphore = new SemaphoreSlim(maxConcurrency);
    var tasks = batches.Select(async batch =>
    {
        await semaphore.WaitAsync(cancellationToken);
        try
        {
            await ProcessBatchAsync(batch, cancellationToken);
        }
        finally
        {
            semaphore.Release();
        }
    });
    
    await Task.WhenAll(tasks);
}
```

## See Also

- [Workflow Orchestration](./workflow-orchestration) - Chaining batch operations
- [Error Handling](../../concepts/error-handling) - Handling batch failures
- [Database Cleanup](./database-cleanup) - Batch cleanup patterns

