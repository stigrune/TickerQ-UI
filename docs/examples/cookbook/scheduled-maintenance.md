# Scheduled Maintenance

Automate database optimization and maintenance tasks.

## Database Optimization

Optimize database weekly on Sunday at 3 AM:

```csharp
[TickerFunction("OptimizeDatabase", cronExpression: "0 0 3 * * 0")]
public async Task OptimizeDatabase(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    // Runs every Sunday at 3 AM
    using var scope = context.ServiceScope.ServiceProvider.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    
    // Rebuild indexes, update statistics, etc.
    await dbContext.Database.ExecuteSqlRawAsync(
        "EXEC sp_updatestats", 
        cancellationToken);
    
    _logger.LogInformation("Database optimization completed");
}
```

## Index Maintenance

Rebuild fragmented indexes:

```csharp
[TickerFunction("MaintainIndexes", cronExpression: "0 0 2 * * 0")]
public async Task MaintainIndexes(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    using var scope = context.ServiceScope.ServiceProvider.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    
    var indexes = await GetFragmentedIndexesAsync(dbContext, cancellationToken);
    
    foreach (var index in indexes)
    {
        if (index.Fragmentation > 30)
        {
            await RebuildIndexAsync(index, dbContext, cancellationToken);
        }
        else
        {
            await ReorganizeIndexAsync(index, dbContext, cancellationToken);
        }
    }
}
```

## Backup Verification

Verify backups daily:

```csharp
[TickerFunction("VerifyBackups", cronExpression: "0 0 6 * * *")]
public async Task VerifyBackups(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    var backupService = _serviceProvider.GetRequiredService<IBackupService>();
    
    var backups = await backupService.GetRecentBackupsAsync(cancellationToken);
    
    foreach (var backup in backups)
    {
        var isValid = await backupService.VerifyBackupAsync(backup, cancellationToken);
        
        if (!isValid)
        {
            await _notificationService.SendAlertAsync(
                $"Backup {backup.Id} failed verification",
                cancellationToken
            );
        }
    }
}
```

## Health Checks

Run system health checks periodically:

```csharp
[TickerFunction("HealthCheck", cronExpression: "0 */5 * * * *")]
public async Task HealthCheck(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    var healthChecks = new[]
    {
        CheckDatabaseHealthAsync(cancellationToken),
        CheckExternalServiceHealthAsync(cancellationToken),
        CheckDiskSpaceAsync(cancellationToken),
        CheckMemoryUsageAsync(cancellationToken)
    };
    
    var results = await Task.WhenAll(healthChecks);
    
    if (results.Any(r => !r.IsHealthy))
    {
        await _alertService.SendAlertAsync(
            "System health check failed",
            cancellationToken
        );
    }
}
```

## See Also

- [Database Cleanup](./database-cleanup) - Related cleanup tasks
- [Error Handling](../../concepts/error-handling) - Handling maintenance failures
- [Scheduled Reports](./scheduled-reports) - Maintenance reports

