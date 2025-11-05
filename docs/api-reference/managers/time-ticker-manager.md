# ITimeTickerManager&lt;TTimeTicker&gt;

Manages time-based jobs (TimeTicker) - jobs scheduled for specific execution times.

## Methods

### AddAsync

Schedule a new TimeTicker job.

**Signature:**
```csharp
Task<TickerResult<TTimeTicker>> AddAsync(
    TTimeTicker entity,
    CancellationToken cancellationToken = default);
```

**Example:**
```csharp
var result = await _timeTickerManager.AddAsync(new TimeTickerEntity
{
    Function = "SendEmail",
    ExecutionTime = DateTime.UtcNow.AddMinutes(5),
    Request = TickerHelper.CreateTickerRequest(emailRequest),
    Retries = 3,
    RetryIntervals = new[] { 60, 300, 900 }
}, cancellationToken);
```

### UpdateAsync

Update an existing TimeTicker. Only jobs with `Idle` status can be updated.

**Signature:**
```csharp
Task<TickerResult<TTimeTicker>> UpdateAsync(
    TTimeTicker timeTicker,
    CancellationToken cancellationToken = default);
```

### DeleteAsync

Delete a TimeTicker by ID.

**Signature:**
```csharp
Task<TickerResult<TTimeTicker>> DeleteAsync(
    Guid id,
    CancellationToken cancellationToken = default);
```

### AddBatchAsync

Schedule multiple TimeTicker jobs in a single operation.

**Signature:**
```csharp
Task<TickerResult<List<TTimeTicker>>> AddBatchAsync(
    List<TTimeTicker> entities,
    CancellationToken cancellationToken = default);
```

### UpdateBatchAsync

Update multiple TimeTicker jobs.

**Signature:**
```csharp
Task<TickerResult<List<TTimeTicker>>> UpdateBatchAsync(
    List<TTimeTicker> timeTickers,
    CancellationToken cancellationToken = default);
```

### DeleteBatchAsync

Delete multiple TimeTicker jobs by their IDs.

**Signature:**
```csharp
Task<TickerResult<TTimeTicker>> DeleteBatchAsync(
    List<Guid> ids,
    CancellationToken cancellationToken = default);
```

## Common Patterns

### Schedule Job After User Action

```csharp
public async Task<IActionResult> RegisterUser(UserRegistrationDto dto)
{
    var user = await CreateUserAsync(dto);
    
    var result = await _timeTickerManager.AddAsync(new TimeTickerEntity
    {
        Function = "SendWelcomeEmail",
        ExecutionTime = DateTime.UtcNow.AddMinutes(5),
        Request = TickerHelper.CreateTickerRequest(new EmailRequest
        {
            UserId = user.Id,
            Email = user.Email
        }),
        Retries = 3,
        RetryIntervals = new[] { 60, 300, 900 }
    });
    
    if (!result.IsSucceeded)
    {
        _logger.LogError(result.Exception, "Failed to schedule welcome email");
    }
    
    return Ok();
}
```

### Reschedule Job

```csharp
public async Task RescheduleJob(Guid jobId, DateTime newExecutionTime)
{
    // Get job via persistence provider (when using EF Core)
    var job = await _persistenceProvider.GetTimeTickerByIdAsync(jobId, cancellationToken);
    
    if (job != null && job.Status == TickerStatus.Idle)
    {
        job.ExecutionTime = newExecutionTime;
        var result = await _timeTickerManager.UpdateAsync(job, cancellationToken);
        
        if (result.IsSucceeded)
        {
            Console.WriteLine("Job rescheduled successfully");
        }
    }
}
```

## Error Handling

Always check the `IsSucceeded` property:

```csharp
var result = await _timeTickerManager.AddAsync(ticker);

if (!result.IsSucceeded)
{
    switch (result.Exception)
    {
        case TickerValidatorException ex:
            _logger.LogWarning("Validation error: {Message}", ex.Message);
            break;
        default:
            _logger.LogError(result.Exception, "Unexpected error scheduling job");
            break;
    }
}
```

## See Also

- [ICronTickerManager](./cron-ticker-manager) - Cron-based job management
- [TickerResult](./ticker-result) - Result type reference
- [TimeTickerEntity](../entities/time-ticker-entity) - TimeTicker entity properties

