# Exception Handling

Configure global exception handlers for TickerQ job execution errors.

## SetExceptionHandler

Register a global exception handler that processes all job execution exceptions.

**Method:**
```csharp
TickerOptionsBuilder<TTimeTicker, TCronTicker> SetExceptionHandler<THandler>() 
    where THandler : ITickerExceptionHandler;
```

**Example:**
```csharp
options.SetExceptionHandler<MyExceptionHandler>();
```

## Requirements

- Handler must implement `ITickerExceptionHandler` interface
- Handler is registered as a singleton service
- Handler is invoked for all job execution exceptions

## ITickerExceptionHandler Interface

```csharp
public interface ITickerExceptionHandler
{
    Task HandleExceptionAsync(
        Exception exception,
        TickerFunctionContext context,
        CancellationToken cancellationToken);
}
```

## Example Implementation

```csharp
using TickerQ.Utilities.Interfaces;
using TickerQ.Utilities.Base;

public class MyExceptionHandler : ITickerExceptionHandler
{
    private readonly ILogger<MyExceptionHandler> _logger;
    private readonly IEmailService _emailService;
    
    public MyExceptionHandler(
        ILogger<MyExceptionHandler> logger,
        IEmailService emailService)
    {
        _logger = logger;
        _emailService = emailService;
    }
    
    public async Task HandleExceptionAsync(
        Exception exception,
        TickerFunctionContext context,
        CancellationToken cancellationToken)
    {
        // Log the error
        _logger.LogError(exception, 
            "Job {JobId} ({Function}) failed after {RetryCount} retries", 
            context.Id, context.FunctionName, context.RetryCount);
        
        // Send notification for critical errors
        if (exception is CriticalBusinessException)
        {
            await _emailService.SendAsync(new EmailMessage
            {
                To = "admin@example.com",
                Subject = $"Critical Job Failure: {context.FunctionName}",
                Body = $"Job ID: {context.Id}\nError: {exception.Message}"
            }, cancellationToken);
        }
        
        // Store error details in database
        await StoreErrorDetailsAsync(context.Id, exception, cancellationToken);
    }
    
    private async Task StoreErrorDetailsAsync(Guid jobId, Exception exception, CancellationToken cancellationToken)
    {
        // Your error storage logic
    }
}
```

## Configuration

```csharp
builder.Services.AddTickerQ(options =>
{
    options.SetExceptionHandler<MyExceptionHandler>();
    
    // Other configuration...
});
```

## Exception Handler Scenarios

### Logging Only

```csharp
public class LoggingExceptionHandler : ITickerExceptionHandler
{
    private readonly ILogger<LoggingExceptionHandler> _logger;
    
    public LoggingExceptionHandler(ILogger<LoggingExceptionHandler> logger)
    {
        _logger = logger;
    }
    
    public Task HandleExceptionAsync(
        Exception exception,
        TickerFunctionContext context,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, 
            "Job {JobId} ({Function}) failed", 
            context.Id, context.FunctionName);
        
        return Task.CompletedTask;
    }
}
```

### Notification Handler

```csharp
public class NotificationExceptionHandler : ITickerExceptionHandler
{
    private readonly IEmailService _emailService;
    private readonly ILogger<NotificationExceptionHandler> _logger;
    
    public NotificationExceptionHandler(
        IEmailService emailService,
        ILogger<NotificationExceptionHandler> logger)
    {
        _emailService = emailService;
        _logger = logger;
    }
    
    public async Task HandleExceptionAsync(
        Exception exception,
        TickerFunctionContext context,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "Job {JobId} failed", context.Id);
        
        // Only notify after all retries exhausted
        if (context.RetryCount >= context.Retries)
        {
            await _emailService.SendAlertAsync(new AlertMessage
            {
                Severity = AlertSeverity.High,
                Message = $"Job {context.FunctionName} failed permanently",
                Details = exception.ToString()
            }, cancellationToken);
        }
    }
}
```

### Metrics and Monitoring

```csharp
public class MetricsExceptionHandler : ITickerExceptionHandler
{
    private readonly IMetricsCollector _metrics;
    
    public MetricsExceptionHandler(IMetricsCollector metrics)
    {
        _metrics = metrics;
    }
    
    public Task HandleExceptionAsync(
        Exception exception,
        TickerFunctionContext context,
        CancellationToken cancellationToken)
    {
        _metrics.IncrementCounter("tickerq.job.failures", new Dictionary<string, string>
        {
            { "function", context.FunctionName },
            { "exception_type", exception.GetType().Name }
        });
        
        return Task.CompletedTask;
    }
}
```

## Error Handling Flow

1. **Job execution fails** → Exception thrown
2. **Retry logic** → Attempts retries (if configured)
3. **After retries exhausted** → `HandleExceptionAsync` is called
4. **Handler processes error** → Logging, notifications, etc.
5. **Job marked as Failed** → Status updated in database

## Best Practices

1. **Always log errors** - Include job context and exception details
2. **Handle exceptions gracefully** - Don't throw from handler
3. **Use cancellation token** - Respect cancellation requests
4. **Avoid blocking operations** - Use async/await properly
5. **Include job context** - Job ID, function name, retry count are valuable

## When No Handler Is Set

If no exception handler is configured:
- Exceptions are still logged through the default instrumentation
- Jobs are marked as `Failed` in the database
- No custom processing occurs

## See Also

- [Start Mode](./start-mode) - Application startup control
- [Scheduler Configuration](./scheduler-configuration) - Scheduler options
- [Core Configuration Overview](./index) - All core configuration options
- [Error Handling Guide](../../../../concepts/error-handling) - Complete error handling guide

