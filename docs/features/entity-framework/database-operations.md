# Database Operations

Use Entity Framework Core to **query** TickerQ jobs, and use TickerQ manager APIs to **create, update, or delete** active jobs so the scheduler is properly notified.

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

## Updating Jobs (via managers)

TickerQ jobs should be updated through the manager APIs so the scheduler, caches, and dashboard are kept in sync. EF Core should only be used to **discover which jobs** you want to act on.

> **Tip:** See [Manager APIs](/api-reference/managers/index) for the full `ITimeTickerManager` / `ICronTickerManager` surface (add, update, delete, batch operations).

### Update Execution Time

```csharp
public class JobUpdateService
{
    private readonly MyDbContext _context;
    private readonly ITimeTickerManager<TimeTickerEntity> _timeTickerManager;

    public JobUpdateService(
        MyDbContext context,
        ITimeTickerManager<TimeTickerEntity> timeTickerManager)
    {
        _context = context;
        _timeTickerManager = timeTickerManager;
    }

    public async Task<bool> UpdateJobExecutionTimeAsync(Guid jobId, DateTime newTime)
    {
        // Read job with EF Core
        var job = await _context.Set<TimeTickerEntity>()
            .FirstOrDefaultAsync(t => t.Id == jobId && t.Status == TickerStatus.Idle);

        if (job is null)
        {
            return false;
        }

        // Mutate via manager so scheduler is notified
        job.ExecutionTime = newTime;
        var result = await _timeTickerManager.UpdateAsync(job);
        return result.IsSucceeded;
    }
}
```

### Cancel a Job

```csharp
public async Task CancelJobAsync(Guid jobId)
{
    var job = await _context.Set<TimeTickerEntity>()
        .FirstOrDefaultAsync(t => t.Id == jobId && t.Status == TickerStatus.InProgress);

    if (job is not null)
    {
        // Here "cancel" is implemented as delete before execution
        await _timeTickerManager.DeleteAsync(job.Id);
    }
}
```

### Update Cron Expression

```csharp
public class CronUpdateService
{
    private readonly MyDbContext _context;
    private readonly ICronTickerManager<CronTickerEntity> _cronTickerManager;

    public CronUpdateService(
        MyDbContext context,
        ICronTickerManager<CronTickerEntity> cronTickerManager)
    {
        _context = context;
        _cronTickerManager = cronTickerManager;
    }

    public async Task<bool> UpdateCronExpressionAsync(Guid cronTickerId, string newExpression)
    {
        var cron = await _context.Set<CronTickerEntity>()
            .FirstOrDefaultAsync(c => c.Id == cronTickerId);

        if (cron is null)
        {
            return false;
        }

        cron.Expression = newExpression;
        cron.UpdatedAt = DateTime.UtcNow;

        var result = await _cronTickerManager.UpdateAsync(cron);
        return result.IsSucceeded;
    }
}
```

## Deleting Jobs (via managers when possible)

For most application workflows, prefer deleting jobs via manager APIs so the scheduler and dashboard are aware of the removal. Direct EF deletes against TickerQ tables should be reserved for maintenance/cleanup tasks that operate on completed or historical data only.

### Delete Completed Jobs (historical cleanup)

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

### Delete Old Occurrences (historical cleanup)

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

### Delete Failed Jobs (active jobs via manager)

```csharp
public async Task<int> DeleteFailedJobsAsync(DateTime olderThan)
{
    var failedJobIds = await _context.Set<TimeTickerEntity>()
        .Where(t => t.Status == TickerStatus.Failed 
                    && t.ExecutedAt < olderThan)
        .Select(t => t.Id)
        .ToListAsync();

    foreach (var jobId in failedJobIds)
    {
        await _timeTickerManager.DeleteAsync(jobId);
    }

    return failedJobIds.Count;
}
```

## Bulk Operations

For bulk operations, follow the same pattern: discover job IDs via EF Core, then call manager APIs for the actual mutations. This keeps the scheduler state consistent and ensures dashboard updates are propagated.

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
