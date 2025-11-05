# Database Cleanup

Patterns for periodic cleanup and maintenance jobs.

## Periodic Cleanup Job

Clean up old records daily at 2 AM:

```csharp
[TickerFunction("CleanupOldRecords", cronExpression: "0 0 2 * * *")]
public async Task CleanupOldRecords(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    using var scope = context.ServiceScope.ServiceProvider.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    
    var cutoffDate = DateTime.UtcNow.AddDays(-90);
    
    // Clean up old logs
    var oldLogs = await dbContext.Logs
        .Where(l => l.CreatedAt < cutoffDate)
        .ToListAsync();
    
    dbContext.Logs.RemoveRange(oldLogs);
    await dbContext.SaveChangesAsync(cancellationToken);
    
    _logger.LogInformation("Cleaned up {Count} old log records", oldLogs.Count);
}
```

## Archive Before Delete

Archive records before deleting them:

```csharp
[TickerFunction("ArchiveAndCleanup")]
public async Task ArchiveAndCleanup(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    using var scope = context.ServiceScope.ServiceProvider.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    var archiveService = scope.ServiceProvider.GetRequiredService<IArchiveService>();
    
    var cutoffDate = DateTime.UtcNow.AddDays(-365);
    var oldRecords = await dbContext.Orders
        .Where(o => o.CreatedAt < cutoffDate && !o.Archived)
        .ToListAsync();
    
    foreach (var record in oldRecords)
    {
        // Archive first
        await archiveService.ArchiveAsync(record, cancellationToken);
        record.Archived = true;
    }
    
    await dbContext.SaveChangesAsync(cancellationToken);
    
    // Then delete archived records older than 2 years
    var veryOldRecords = await dbContext.Orders
        .Where(o => o.CreatedAt < DateTime.UtcNow.AddYears(-2) && o.Archived)
        .ToListAsync();
    
    dbContext.Orders.RemoveRange(veryOldRecords);
    await dbContext.SaveChangesAsync(cancellationToken);
}
```

## Incremental Cleanup

Process cleanup in batches to avoid long-running transactions:

```csharp
[TickerFunction("IncrementalCleanup", cronExpression: "0 */30 * * * *")]
public async Task IncrementalCleanup(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    using var scope = context.ServiceScope.ServiceProvider.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    
    var batchSize = 1000;
    var cutoffDate = DateTime.UtcNow.AddDays(-90);
    
    while (true)
    {
        var batch = await dbContext.TempRecords
            .Where(r => r.CreatedAt < cutoffDate)
            .Take(batchSize)
            .ToListAsync(cancellationToken);
        
        if (!batch.Any())
            break;
        
        dbContext.TempRecords.RemoveRange(batch);
        await dbContext.SaveChangesAsync(cancellationToken);
        
        // Check cancellation between batches
        cancellationToken.ThrowIfCancellationRequested();
    }
}
```

## See Also

- [Scheduled Maintenance](./scheduled-maintenance) - Database optimization
- [Batch Processing](./batch-processing) - Processing large datasets
- [Error Handling](../../concepts/error-handling) - Handling cleanup failures

