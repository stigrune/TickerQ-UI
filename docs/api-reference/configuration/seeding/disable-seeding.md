# Disable Seeding

Prevent automatic seeding of cron jobs from `[TickerFunction]` attributes when you want to manage jobs manually.

## IgnoreSeedDefinedCronTickers

Disable automatic seeding of cron jobs from attributes.

**Method:**
```csharp
TickerQEfCoreOptionBuilder<TTimeTicker, TCronTicker> IgnoreSeedDefinedCronTickers();
```

## Basic Usage

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddOperationalStore(efOptions =>
    {
        efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
        {
            optionsBuilder.UseNpgsql(connectionString);
        });
        
        // Disable automatic seeding
        efOptions.IgnoreSeedDefinedCronTickers();
    });
});
```

## When to Disable

Disable automatic seeding when:
- You manage cron jobs manually through the dashboard or API
- Jobs are created programmatically at runtime
- You want full control over job creation
- Jobs are managed by external systems
- You're migrating from automatic to manual job management

## What Gets Disabled

`IgnoreSeedDefinedCronTickers()` only disables:
- Automatic seeding from `[TickerFunction]` attributes with cron expressions

It does **not** disable:
- Custom seeding via `UseTickerSeeder()`
- Manual job creation through managers
- Jobs created at runtime

## Example: Manual Job Management

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddOperationalStore(efOptions =>
    {
        efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
        {
            optionsBuilder.UseNpgsql(connectionString);
        });
        
        // Disable automatic seeding - manage jobs manually
        efOptions.IgnoreSeedDefinedCronTickers();
    });
});

// Later in your application
public class JobService
{
    private readonly ICronTickerManager<CronTickerEntity> _cronTickerManager;
    
    public async Task SetupJobsAsync()
    {
        // Manually create cron jobs
        await _cronTickerManager.AddAsync(new CronTickerEntity
        {
            Function = "GenerateDailyReport",
            Expression = "0 0 9 * * *",
            Description = "Daily report generation"
        });
    }
}
```

## Combining with Custom Seeding

You can disable automatic seeding but still use custom seeding:

```csharp
options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        optionsBuilder.UseNpgsql(connectionString);
    });
    
    // Disable automatic seeding
    efOptions.IgnoreSeedDefinedCronTickers();
    
    // But still use custom seeding
    efOptions.UseTickerSeeder(
        async timeManager =>
        {
            // Custom TimeTicker seeding
            await timeManager.AddAsync(new TimeTickerEntity
            {
                Function = "InitialSetup",
                ExecutionTime = DateTime.UtcNow.AddHours(1)
            });
        },
        async cronManager =>
        {
            // Custom CronTicker seeding
            await cronManager.AddAsync(new CronTickerEntity
            {
                Function = "CustomCronJob",
                Expression = "0 0 */6 * * *" // Every 6 hours
            });
        }
    );
});
```

## Migration Scenario

When migrating from automatic to manual job management:

```csharp
// Step 1: Disable automatic seeding
options.AddOperationalStore(efOptions =>
{
    efOptions.IgnoreSeedDefinedCronTickers();
});

// Step 2: Existing jobs in database remain unchanged
// Step 3: New jobs must be created manually

// Step 4: Remove [TickerFunction] cron expressions if desired
// (Or keep them for documentation but they won't auto-seed)
```

## Re-enabling Automatic Seeding

To re-enable automatic seeding, simply remove or don't call `IgnoreSeedDefinedCronTickers()`:

```csharp
options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        optionsBuilder.UseNpgsql(connectionString);
    });
    // Don't call IgnoreSeedDefinedCronTickers()
    // Automatic seeding is enabled by default
});
```

## Impact on Existing Jobs

Disabling automatic seeding does **not**:
- Delete existing jobs from the database
- Modify existing jobs
- Prevent manual job creation
- Affect jobs created via custom seeding

Existing jobs continue to run normally.

## Best Practices

1. **Document your decision** - Note why automatic seeding is disabled
2. **Use custom seeding** - For initial setup jobs when disabling automatic seeding
3. **Version control** - Track which jobs should exist if managing manually
4. **Dashboard management** - Use the dashboard to create and manage jobs visually
5. **API management** - Create jobs programmatically via managers when needed

## See Also

- [Automatic Seeding](./automatic-seeding) - Understanding automatic seeding
- [Custom Seeding](./custom-seeding) - Programmatic seeding alternative
- [Seeding Overview](./index) - All seeding options

