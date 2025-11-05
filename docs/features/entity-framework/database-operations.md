# Database Operations

Query, update, and manage TickerQ jobs through Entity Framework Core.

## Querying Jobs

### Get Failed Jobs

```csharp
public class JobService
{
    private readonly MyDbContext _context;
    
    public JobService(MyDbContext context)
    {
        _context = context;
    }
    
    public async Task<List<TimeTickerEntity>> GetFailedJobsAsync()
    {
        return await _context.Set<TimeTickerEntity>()
            .Where(t => t.Status == TickerStatus.Failed)
            .OrderByDescending(t => t.ExecutedAt)
            .ToListAsync();
    }
}
```

### Get Jobs by Status

```csharp
public async Task<List<TimeTickerEntity>> GetJobsByStatusAsync(TickerStatus status)
{
    return await _context.Set<TimeTickerEntity>()
        .Where(t => t.Status == status)
        .OrderBy(t => t.ExecutionTime)
        .ToListAsync();
}
```

### Get Recent Cron Occurrences

```csharp
public async Task<List<CronTickerOccurrenceEntity>> GetRecentOccurrencesAsync(
    Guid cronTickerId, int count = 10)
{
    return await _context.Set<CronTickerOccurrenceEntity>()
        .Where(o => o.CronTickerId == cronTickerId)
        .OrderByDescending(o => o.ExecutionTime)
        .Take(count)
        .ToListAsync();
}
```

### Get Jobs by Execution Time Range

```csharp
public async Task<List<TimeTickerEntity>> GetJobsInTimeRangeAsync(
    DateTime startTime, DateTime endTime)
{
    return await _context.Set<TimeTickerEntity>()
        .Where(t => t.ExecutionTime >= startTime && t.ExecutionTime <= endTime)
        .ToListAsync();
}
```

## Updating Jobs

### Update Execution Time

```csharp
public async Task<bool> UpdateJobExecutionTimeAsync(Guid jobId, DateTime newTime)
{
    var job = await _context.Set<TimeTickerEntity>()
        .FirstOrDefaultAsync(t => t.Id == jobId);
    
    if (job != null && job.Status == TickerStatus.Idle)
    {
        job.ExecutionTime = newTime;
        await _context.SaveChangesAsync();
        return true;
    }
    
    return false;
}
```

### Update Job Status

```csharp
public async Task MarkJobAsCancelledAsync(Guid jobId)
{
    var job = await _context.Set<TimeTickerEntity>()
        .FirstOrDefaultAsync(t => t.Id == jobId);
    
    if (job != null && job.Status == TickerStatus.InProgress)
    {
        job.Status = TickerStatus.Cancelled;
        job.UpdatedAt = DateTime.UtcNow;
        await _context.SaveChangesAsync();
    }
}
```

### Update Cron Expression

```csharp
public async Task UpdateCronExpressionAsync(Guid cronTickerId, string newExpression)
{
    var cronTicker = await _context.Set<CronTickerEntity>()
        .FirstOrDefaultAsync(c => c.Id == cronTickerId);
    
    if (cronTicker != null)
    {
        cronTicker.Expression = newExpression;
        cronTicker.UpdatedAt = DateTime.UtcNow;
        await _context.SaveChangesAsync();
        
        // Note: You may want to use manager.UpdateAsync() instead
        // to trigger recalculation of occurrences
    }
}
```

## Deleting Jobs

### Delete Completed Jobs

```csharp
public async Task<int> DeleteCompletedJobsAsync(DateTime olderThan)
{
    var completedJobs = await _context.Set<TimeTickerEntity>()
        .Where(t => (t.Status == TickerStatus.Done || t.Status == TickerStatus.DueDone)
                    && t.ExecutedAt < olderThan)
        .ToListAsync();
    
    _context.Set<TimeTickerEntity>().RemoveRange(completedJobs);
    return await _context.SaveChangesAsync();
}
```

### Delete Old Occurrences

```csharp
public async Task<int> CleanupOldOccurrencesAsync(int olderThanDays = 30)
{
    var cutoffDate = DateTime.UtcNow.AddDays(-olderThanDays);
    
    var oldOccurrences = await _context.Set<CronTickerOccurrenceEntity>()
        .Where(o => o.CreatedAt < cutoffDate 
                    && o.Status != TickerStatus.InProgress)
        .ToListAsync();
    
    _context.Set<CronTickerOccurrenceEntity>().RemoveRange(oldOccurrences);
    return await _context.SaveChangesAsync();
}
```

### Delete Failed Jobs

```csharp
public async Task<int> DeleteFailedJobsAsync(DateTime olderThan)
{
    var failedJobs = await _context.Set<TimeTickerEntity>()
        .Where(t => t.Status == TickerStatus.Failed 
                    && t.ExecutedAt < olderThan)
        .ToListAsync();
    
    _context.Set<TimeTickerEntity>().RemoveRange(failedJobs);
    return await _context.SaveChangesAsync();
}
```

## Bulk Operations

### Bulk Status Update

```csharp
public async Task<int> BulkUpdateStatusAsync(
    List<Guid> jobIds, TickerStatus newStatus)
{
    var jobs = await _context.Set<TimeTickerEntity>()
        .Where(t => jobIds.Contains(t.Id))
        .ToListAsync();
    
    foreach (var job in jobs)
    {
        job.Status = newStatus;
        job.UpdatedAt = DateTime.UtcNow;
    }
    
    return await _context.SaveChangesAsync();
}
```

### Bulk Delete

```csharp
public async Task<int> BulkDeleteJobsAsync(List<Guid> jobIds)
{
    var jobs = await _context.Set<TimeTickerEntity>()
        .Where(t => jobIds.Contains(t.Id))
        .ToListAsync();
    
    _context.Set<TimeTickerEntity>().RemoveRange(jobs);
    return await _context.SaveChangesAsync();
}
```

## Complex Queries

### Jobs with Children

```csharp
public async Task<List<TimeTickerEntity>> GetJobsWithChildrenAsync()
{
    return await _context.Set<TimeTickerEntity>()
        .Include(t => t.Children)
        .Where(t => t.Children.Any())
        .ToListAsync();
}
```

### Cron Jobs with Recent Failures

```csharp
public async Task<List<CronTickerEntity>> GetCronJobsWithRecentFailuresAsync(int days = 7)
{
    var cutoffDate = DateTime.UtcNow.AddDays(-days);
    
    return await _context.Set<CronTickerEntity>()
        .Where(c => c.Occurrences.Any(o => 
            o.Status == TickerStatus.Failed && o.ExecutedAt >= cutoffDate))
        .ToListAsync();
}
```

### Job Statistics

```csharp
public async Task<JobStatistics> GetJobStatisticsAsync()
{
    var stats = new JobStatistics
    {
        TotalJobs = await _context.Set<TimeTickerEntity>().CountAsync(),
        PendingJobs = await _context.Set<TimeTickerEntity>()
            .CountAsync(t => t.Status == TickerStatus.Idle || t.Status == TickerStatus.Queued),
        FailedJobs = await _context.Set<TimeTickerEntity>()
            .CountAsync(t => t.Status == TickerStatus.Failed),
        CompletedJobs = await _context.Set<TimeTickerEntity>()
            .CountAsync(t => t.Status == TickerStatus.Done || t.Status == TickerStatus.DueDone)
    };
    
    return stats;
}
```

## Transaction Management

### Transactional Operations

```csharp
using var transaction = await _context.Database.BeginTransactionAsync();

try
{
    // Update parent job
    var parent = await _context.Set<TimeTickerEntity>().FindAsync(parentId);
    parent.Status = TickerStatus.DueDone;
    
    // Create child jobs
    var child = new TimeTickerEntity
    {
        Function = "ChildJob",
        ExecutionTime = DateTime.UtcNow.AddMinutes(5),
        ParentId = parentId,
        RunCondition = RunCondition.OnSuccess
    };
    await _context.Set<TimeTickerEntity>().AddAsync(child);
    
    await _context.SaveChangesAsync();
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

## Best Practices

1. **Use Async Methods**: Always use `ToListAsync()`, `FirstOrDefaultAsync()`, etc.
2. **Filter Before Materializing**: Apply `.Where()` before `.ToListAsync()` to reduce data transfer
3. **Include Navigation Properties**: Use `.Include()` when accessing related entities
4. **Batch Operations**: Use bulk operations for deleting or updating multiple records
5. **Avoid Direct Updates**: Prefer manager APIs when possible for consistency

## Performance Considerations

### Efficient Queries

```csharp
// Good: Filter before materializing
var jobs = await _context.Set<TimeTickerEntity>()
    .Where(t => t.Status == TickerStatus.Failed)
    .Select(t => new { t.Id, t.Function, t.ExecutedAt })
    .ToListAsync();

// Avoid: Loading all then filtering
var allJobs = await _context.Set<TimeTickerEntity>().ToListAsync();
var failed = allJobs.Where(t => t.Status == TickerStatus.Failed);
```

### Index Usage

TickerQ creates indexes on commonly queried fields. Ensure your queries use indexed columns:

- `ExecutionTime` - For time-based queries
- `Status` - For status filtering
- `CronTickerId` - For occurrence queries
- `ParentId` - For child job queries

## See Also

- [Migrations](./migrations) - Database schema management
- [Performance](./performance) - Optimization guide
- [Best Practices](./best-practices) - Production recommendations
- [Manager APIs](/api-reference/managers/index) - Programmatic job management

