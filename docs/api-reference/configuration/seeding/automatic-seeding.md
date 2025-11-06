# Automatic Cron Seeding

TickerQ can automatically seed cron jobs from `[TickerFunction]` attributes that include cron expressions.

## How It Works

When TickerQ starts, it scans all methods marked with `[TickerFunction]` attributes that have a `cronExpression` parameter. These cron jobs are automatically created (upserted) in the database.

## Example: Defining Cron Jobs with Attributes

```csharp
public class ScheduledJobs
{
    [TickerFunction("GenerateDailyReport", "0 0 9 * * *")] // Daily at 9 AM
    public async Task GenerateDailyReport(
        TickerFunctionContext context,
        CancellationToken cancellationToken)
    {
        // Your job logic
    }
    
    [TickerFunction("CleanupLogs", "0 0 0 * * *")] // Daily at midnight
    public async Task CleanupLogs(
        TickerFunctionContext context,
        CancellationToken cancellationToken)
    {
        // Your cleanup logic
    }
    
    [TickerFunction("SendWeeklyDigest", "0 0 8 * * 1")] // Every Monday at 8 AM
    public async Task SendWeeklyDigest(
        TickerFunctionContext context,
        CancellationToken cancellationToken)
    {
        // Your email digest logic
    }
}
```

## Automatic Behavior

By default, automatic seeding is **enabled**. When your application starts:

1. TickerQ scans for all `[TickerFunction]` attributes with cron expressions
2. Creates `CronTickerEntity` entries in the database
3. Uses upsert logic (inserts new, doesn't update existing)

## Seeding Logic

The seeding uses upsert logic based on:
- `Function` name
- `Expression` (cron expression)
- `Request` (must be empty for seeded jobs)

**Important:** If a cron ticker with the same `Function`, `Expression`, and empty `Request` already exists, it will **not** be updated. This prevents overwriting manually configured cron jobs.

## Configuration

Automatic seeding is enabled by default. No configuration is required:

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddOperationalStore(efOptions =>
    {
        efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
        {
            optionsBuilder.UseNpgsql(connectionString);
        });
        // Automatic seeding is enabled by default
    });
});
```

## When to Use

Use automatic seeding when:
- You want cron jobs defined in code via attributes
- You prefer code-based configuration over database configuration
- You want to version control your cron job definitions
- Your cron expressions are static and don't change at runtime

## Limitations

- Only works with cron expressions (not one-time jobs)
- Jobs are seeded with empty `Request` data
- Expression and Function must match exactly for upsert logic
- Cannot seed jobs with custom properties (use custom seeding instead)

## Disabling Automatic Seeding

To disable automatic seeding, use `IgnoreSeedDefinedCronTickers()`:

```csharp
options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        optionsBuilder.UseNpgsql(connectionString);
    });
    efOptions.IgnoreSeedDefinedCronTickers(); // Disable automatic seeding
});
```

See [Disable Seeding](./disable-seeding) for more details.

## Logging

Automatic seeding operations are logged:

```
[INF] TickerQ start seeding data: SeedDefinedCronTickers(CronTickerEntity)
[INF] TickerQ completed seeding data: SeedDefinedCronTickers(CronTickerEntity)
```

With OpenTelemetry instrumentation, you'll see:
- `tickerq.seeding.started` activity
- `tickerq.seeding.completed` activity

## Troubleshooting

**Jobs not being seeded?**
- Verify `[TickerFunction]` has a `cronExpression` parameter
- Check that Entity Framework Core persistence is configured
- Ensure cron expression is valid (6-part format)
- Review application logs for seeding errors

**Jobs being overwritten?**
- Seeding uses upsert logic and won't update existing jobs
- Only jobs with matching `Function`, `Expression`, and empty `Request` are skipped

**Multiple nodes seeding?**
- Each node will attempt to seed, but upsert logic prevents duplicates
- Safe to run in multi-node environments

## See Also

- [Custom Seeding](./custom-seeding) - Programmatic seeding for more control
- [Disable Seeding](./disable-seeding) - Prevent automatic seeding

