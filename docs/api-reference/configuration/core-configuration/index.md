# Core Configuration

Core TickerQ configuration options for scheduler, exception handling, and startup behavior.

## Configuration Sections

### [Scheduler Configuration](./scheduler-configuration)
Configure scheduler behavior including concurrency, timeouts, and node identification.

### [Exception Handling](./exception-handling)
Set up global exception handlers for job execution errors.

### [Start Mode](./start-mode)
Control when TickerQ starts processing jobs.

## Quick Example

```csharp
builder.Services.AddTickerQ<TTimeTicker, TCronTicker>(options =>
{
    // Scheduler configuration
    options.ConfigureScheduler(scheduler =>
    {
        scheduler.MaxConcurrency = 10;
        scheduler.NodeIdentifier = "production-server-01";
    });
    
    // Exception handling
    options.SetExceptionHandler<MyExceptionHandler>();
});
```

## See Also

- [Scheduler Configuration](./scheduler-configuration) - Detailed scheduler options
- [Exception Handling](./exception-handling) - Global exception handler setup
- [Start Mode](./start-mode) - Application startup control
- [Configuration Overview](../index) - All configuration sections

