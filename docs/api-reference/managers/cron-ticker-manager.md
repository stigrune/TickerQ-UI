# ICronTickerManager&lt;TCronTicker&gt;

Manages cron-based jobs (CronTicker) - recurring jobs using cron expressions.

## Methods

### AddAsync

Schedule a new CronTicker job.

**Signature:**
```csharp
Task<TickerResult<TCronTicker>> AddAsync(
    TCronTicker entity,
    CancellationToken cancellationToken = default);
```

**Example:**
```csharp
var result = await _cronTickerManager.AddAsync(new CronTickerEntity
{
    Function = "GenerateDailyReport",
    Expression = "0 0 0 * * *", // Daily at midnight (6-part format required)
    Request = TickerHelper.CreateTickerRequest(reportRequest),
    Retries = 2,
    RetryIntervals = new[] { 300, 900 }
}, cancellationToken);
```

**Validation:**
- Function name must exist (registered via `[TickerFunction]`)
- Cron expression must be valid 6-part format
- Expression must be parseable

### UpdateAsync

Update an existing CronTicker (e.g., change cron expression).

**Signature:**
```csharp
Task<TickerResult<TCronTicker>> UpdateAsync(
    TCronTicker cronTicker,
    CancellationToken cancellationToken = default);
```

**Example:**
```csharp
var cronTicker = await _persistenceProvider.GetCronTickerByIdAsync(cronJobId, cancellationToken);

if (cronTicker != null)
{
    cronTicker.Expression = "0 0 */6 * * *"; // Change to every 6 hours
    cronTicker.Description = "Updated to run every 6 hours";
    
    var result = await _cronTickerManager.UpdateAsync(cronTicker, cancellationToken);
}
```

**Note:** Updating a CronTicker automatically recalculates the next occurrence and updates any pending occurrences.

### DeleteAsync

Delete a CronTicker by ID.

**Signature:**
```csharp
Task<TickerResult<TCronTicker>> DeleteAsync(
    Guid id,
    CancellationToken cancellationToken = default);
```

### AddBatchAsync

Schedule multiple CronTicker jobs in a single operation.

**Signature:**
```csharp
Task<TickerResult<List<TCronTicker>>> AddBatchAsync(
    List<TCronTicker> entities,
    CancellationToken cancellationToken = default);
```

### UpdateBatchAsync

Update multiple CronTicker jobs.

**Signature:**
```csharp
Task<TickerResult<List<TCronTicker>>> UpdateBatchAsync(
    List<TCronTicker> cronTickers,
    CancellationToken cancellationToken = default);
```

### DeleteBatchAsync

Delete multiple CronTicker jobs by their IDs.

**Signature:**
```csharp
Task<TickerResult<TCronTicker>> DeleteBatchAsync(
    List<Guid> ids,
    CancellationToken cancellationToken = default);
```

## Cron Expression Format

CronTicker requires **6-part cron expressions** (with seconds):

```
second minute hour day month day-of-week
```

**Examples:**
- `"0 0 0 * * *"` - Daily at midnight
- `"0 0 9 * * *"` - Daily at 9:00 AM
- `"0 */5 * * * *"` - Every 5 minutes
- `"*/10 * * * * *"` - Every 10 seconds
- `"0 0 9,17 * * *"` - At 9 AM and 5 PM daily
- `"0 30 14 * * 1"` - Every Monday at 2:30 PM

## Common Patterns

### Schedule Multiple Cron Jobs

```csharp
var cronJobs = new List<CronTickerEntity>
{
    new CronTickerEntity
    {
        Function = "CleanupLogs",
        Expression = "0 0 0 * * *" // Daily at midnight
    },
    new CronTickerEntity
    {
        Function = "GenerateReport",
        Expression = "0 0 9 * * *" // Daily at 9 AM
    }
};

var result = await _cronTickerManager.AddBatchAsync(cronJobs, cancellationToken);
```

### Update Cron Schedule

```csharp
var cronTicker = await _persistenceProvider.GetCronTickerByIdAsync(cronJobId, cancellationToken);

if (cronTicker != null)
{
    // Change from daily to hourly
    cronTicker.Expression = "0 0 * * * *";
    var result = await _cronTickerManager.UpdateAsync(cronTicker, cancellationToken);
}
```

## Error Handling

Common errors when working with CronTicker:

```csharp
var result = await _cronTickerManager.AddAsync(cronTicker);

if (!result.IsSucceeded)
{
    switch (result.Exception)
    {
        case TickerValidatorException ex when ex.Message.Contains("Cannot parse expression"):
            _logger.LogError("Invalid cron expression: {Expression}", cronTicker.Expression);
            // Prompt user to fix expression
            break;
        case TickerValidatorException ex when ex.Message.Contains("Cannot find TickerFunction"):
            _logger.LogError("Function not found: {Function}", cronTicker.Function);
            // Verify function name matches [TickerFunction] attribute
            break;
        default:
            _logger.LogError(result.Exception, "Failed to schedule cron job");
            break;
    }
}
```

## See Also

- [ITimeTickerManager](./time-ticker-manager) - Time-based job management
- [TickerResult](./ticker-result) - Result type reference
- [CronTickerEntity](../entities/cron-ticker-entity) - CronTicker entity properties
- [Job Types](../../concepts/job-types) - Understanding cron jobs

