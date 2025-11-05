# Manager APIs

TickerQ provides manager interfaces for scheduling and managing jobs. This guide covers all available APIs in the manager interfaces.

::: tip Accessing Entities
To query or retrieve entities by ID, use the persistence provider methods directly or access them through Entity Framework if you're using EF Core persistence.
:::

## Manager Types

### [ITimeTickerManager](./time-ticker-manager)
Manages time-based jobs (TimeTicker) - jobs scheduled for specific execution times.

### [ICronTickerManager](./cron-ticker-manager)
Manages cron-based jobs (CronTicker) - recurring jobs using cron expressions.

## [TickerResult](./ticker-result)
Result type returned by all manager operations, providing success status and exception information.

## Common Patterns

### Dependency Injection

Managers are registered as singletons and can be injected:

```csharp
public class MyService
{
    private readonly ITimeTickerManager<TimeTickerEntity> _timeTickerManager;
    private readonly ICronTickerManager<CronTickerEntity> _cronTickerManager;
    
    public MyService(
        ITimeTickerManager<TimeTickerEntity> timeTickerManager,
        ICronTickerManager<CronTickerEntity> cronTickerManager)
    {
        _timeTickerManager = timeTickerManager;
        _cronTickerManager = cronTickerManager;
    }
}
```

### Generic Types

When using custom entity types:

```csharp
public class CustomTimeTicker : TimeTickerEntity<CustomTimeTicker> { }
public class CustomCronTicker : CronTickerEntity { }

// In service registration
builder.Services.AddTickerQ<CustomTimeTicker, CustomCronTicker>(options => { });

// In your service
public class MyService
{
    private readonly ITimeTickerManager<CustomTimeTicker> _timeTickerManager;
    private readonly ICronTickerManager<CustomCronTicker> _cronTickerManager;
}
```

## See Also

- [Entity Reference](../entities/index) - Complete entity properties documentation
- [Configuration Reference](../configuration/index) - All configuration options
- [Job Types](../../concepts/job-types) - Understanding TimeTicker vs CronTicker

