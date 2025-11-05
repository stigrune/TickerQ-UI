# Retry Strategies

Different retry approaches for handling transient failures.

## Exponential Backoff

Retry with exponentially increasing delays:

```csharp
await _timeTickerManager.AddAsync(new TimeTickerEntity
{
    Function = "CallExternalApi",
    ExecutionTime = DateTime.UtcNow,
    Retries = 5,
    RetryIntervals = new[] { 1, 2, 4, 8, 16 } // Exponential: 1s, 2s, 4s, 8s, 16s
});
```

## Progressive Backoff

Retry with progressively longer delays:

```csharp
await _timeTickerManager.AddAsync(new TimeTickerEntity
{
    Function = "ProcessPayment",
    ExecutionTime = DateTime.UtcNow,
    Retries = 4,
    RetryIntervals = new[] { 30, 60, 300, 900 } // Progressive: 30s, 1m, 5m, 15m
});
```

## Fixed Interval

Retry with fixed delay between attempts:

```csharp
await _timeTickerManager.AddAsync(new TimeTickerEntity
{
    Function = "SyncData",
    ExecutionTime = DateTime.UtcNow,
    Retries = 3,
    RetryIntervals = new[] { 60, 60, 60 } // Fixed: always 1 minute
});
```

## Smart Retry Based on Error Type

Implement different retry strategies based on exception type:

```csharp
[TickerFunction("SmartRetryJob")]
public async Task SmartRetryJob(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    try
    {
        await ProcessAsync(cancellationToken);
    }
    catch (TransientException ex)
    {
        // Transient errors should retry
        _logger.LogWarning(ex, "Transient error, will retry");
        throw;
    }
    catch (PermanentException ex)
    {
        // Permanent errors should not retry
        _logger.LogError(ex, "Permanent error, skipping retry");
        // Don't throw - mark as handled
    }
    catch (RateLimitException ex)
    {
        // Rate limits need longer delays
        await Task.Delay(TimeSpan.FromMinutes(5), cancellationToken);
        throw; // Retry after delay
    }
}
```

## Conditional Retry

Retry only if certain conditions are met:

```csharp
[TickerFunction("ConditionalRetry")]
public async Task ConditionalRetry(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    var maxRetries = 3;
    
    if (context.RetryCount >= maxRetries)
    {
        _logger.LogError("Max retries reached, giving up");
        return; // Don't throw - stop retrying
    }
    
    try
    {
        await ProcessAsync(cancellationToken);
    }
    catch (Exception ex)
    {
        // Only retry if it's worth retrying
        if (ShouldRetry(ex))
        {
            throw; // Trigger retry
        }
        else
        {
            _logger.LogWarning(ex, "Not worth retrying");
        }
    }
}
```

## See Also

- [Error Handling](../../concepts/error-handling) - Comprehensive error handling guide
- [API Polling](./api-polling) - Retry strategies for API calls
- [Job Fundamentals](../../concepts/job-fundamentals) - Understanding retries

