# Custom Seeding

Use custom seeding logic to programmatically create initial jobs with full control over job properties, request data, and business logic.

## UseTickerSeeder

Configure custom seeding functions for TimeTicker and CronTicker jobs on the core `TickerOptionsBuilder`. The same seeding pipeline runs regardless of whether you use the in-memory provider or EF Core.

**Methods:**
```csharp
TickerOptionsBuilder<TTimeTicker, TCronTicker> UseTickerSeeder(
    Func<ITimeTickerManager<TTimeTicker>, Task> timeSeeder);

TickerOptionsBuilder<TTimeTicker, TCronTicker> UseTickerSeeder(
    Func<ICronTickerManager<TCronTicker>, Task> cronSeeder);

TickerOptionsBuilder<TTimeTicker, TCronTicker> UseTickerSeeder(
    Func<ITimeTickerManager<TTimeTicker>, Task> timeSeeder,
    Func<ICronTickerManager<TCronTicker>, Task> cronSeeder);
```

## Basic Example

```csharp
Use `UseTickerSeeder` directly on `AddTickerQ` so seeding logic applies regardless of persistence provider (in-memory or EF Core):

```csharp
builder.Services.AddTickerQ(options =>
{
    // Optional: disable auto seeding of code-defined cron tickers
    options.IgnoreSeedDefinedCronTickers();

    options.UseTickerSeeder(
        async timeManager =>
        {
            await timeManager.AddAsync(new TimeTickerEntity
            {
                Function = "WarmUpCache",
                ExecutionTime = DateTime.UtcNow.AddMinutes(1)
            });
        },
        async cronManager =>
        {
            await cronManager.AddAsync(new CronTickerEntity
            {
                Function = "HealthCheck",
                Expression = "0 */5 * * * *" // Every 5 minutes
            });
        });
});
```
```

## Accessing Services in Seeder

Since seeders run during application startup, you can't directly inject services. However, you can access configuration:

```csharp
var configuration = builder.Configuration;

builder.Services.AddTickerQ(options =>
{
    options.UseTickerSeeder(
        async timeManager =>
        {
            var adminEmail = configuration["Admin:Email"];
            var jobDelayMinutes = int.Parse(configuration["Jobs:InitialDelayMinutes"] ?? "60");
            
            await timeManager.AddAsync(new TimeTickerEntity
            {
                Function = "NotifyAdmin",
                ExecutionTime = DateTime.UtcNow.AddMinutes(jobDelayMinutes),
                Request = TickerHelper.CreateTickerRequest(new { Email = adminEmail })
            });
        },
        null
    );
});
```

## Error Handling

Handle errors in seeding functions:

```csharp
builder.Services.AddTickerQ(options =>
{
    options.UseTickerSeeder(
        async timeManager =>
        {
            try
            {
                var result = await timeManager.AddAsync(new TimeTickerEntity
                {
                    Function = "MyJob",
                    ExecutionTime = DateTime.UtcNow.AddHours(1)
                });
                
                if (!result.IsSucceeded)
                {
                    // Log error but don't throw - seeding continues
                    Console.WriteLine($"Seeding failed: {result.Exception?.Message}");
                }
            }
            catch (Exception ex)
            {
                // Log but don't throw - application startup shouldn't fail
                Console.WriteLine($"Seeding exception: {ex.Message}");
            }
        },
        null
    );
});
```

## Seeding Custom Entity Types

When using custom entity types:

```csharp
public class CustomTimeTicker : TimeTickerEntity<CustomTimeTicker>
{
    public string TenantId { get; set; }
}

builder.Services.AddTickerQ<CustomTimeTicker, CustomCronTicker>(options =>
{
    options.UseTickerSeeder(
        async timeManager =>
        {
            await timeManager.AddAsync(new CustomTimeTicker
            {
                Function = "TenantSpecificJob",
                TenantId = "tenant-001",
                ExecutionTime = DateTime.UtcNow.AddDays(1)
            });
        },
        null
    );
});
```

## Best Practices

1. **Use batch operations** when seeding multiple jobs
2. **Make seeding idempotent** - handle cases where jobs might already exist
3. **Use configuration** for environment-specific seeding
4. **Handle errors gracefully** - don't let seeding failures block application startup
5. **Log seeding operations** - track what was seeded for debugging
6. **Combine with automatic seeding** - use both automatic and custom seeding

## When to Use Custom Seeding

Use custom seeding when:
- You need to seed jobs with request data
- Jobs depend on configuration or environment
- You need conditional logic for seeding
- You want to seed one-time (TimeTicker) jobs
- Jobs require custom properties or relationships
- You need to seed jobs based on database state

## Execution Order

Custom seeding executes after automatic cron seeding:
1. Automatic cron seeding (from attributes)
2. Custom TimeTicker seeding
3. Custom CronTicker seeding

## Logging

Custom seeding operations are logged:

```
[INF] TickerQ start seeding data: CustomTimeTicker
[INF] TickerQ completed seeding data: CustomTimeTicker
```

With OpenTelemetry, you'll see separate activities for each seeding operation.

## See Also

- [Automatic Seeding](./automatic-seeding) - Attribute-based cron job seeding
- [Disable Seeding](./disable-seeding) - Prevent automatic seeding
