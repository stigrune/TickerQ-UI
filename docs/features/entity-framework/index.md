# Entity Framework Core Integration

`TickerQ.EntityFrameworkCore` is the persistence provider that powers **TickerQ's operational store**:

- Persists `TimeTicker`, `CronTicker`, and occurrence entities so the scheduler can resume work after restarts.
- Exposes the same entities to your application through EF Core so you can inspect jobs with LINQ (read-only access).
- Feeds the dashboard and Redis cache coordination on top of the TickerQ core.

Compared to generic EF Core docs, this guide focuses specifically on how TickerQ plugs into EF and which TickerQ-specific features it lights up (seeding, dead-node cleanup, dashboard backing store).

## What the provider adds

- **`AddOperationalStore` integration** – registers DbContexts, pooling, and provider services used by `TickerQHostScheduler`.
- **Automatic cron migration** – keeps EF-backed job definitions in sync with `[TickerFunction]` declarations so the scheduler can resume recurring work after restarts.
- **Dead node cleanup** – releases stuck jobs when the host boots, leveraging EF transactions to ensure only one node processes a job.
- **Dashboard backing store** – the Vue dashboard calls the EF-powered HTTP endpoints to query ticker data, history, and pagination metadata.

## Integration flow

1. **Install packages**: `TickerQ` + `TickerQ.EntityFrameworkCore` + your EF Core database provider.
2. **Configure the operational store** inside `AddTickerQ` by calling `options.AddOperationalStore(...)`.
3. **Pick a DbContext model**:
   - `TickerQDbContext` (isolated store),
   - Application DbContext (shared with your domain),
   - Custom entities (advanced scenarios).
4. **Apply migrations** for the DbContext you chose.
5. **Run the application** – on startup TickerQ automatically:
   - Cleans dead nodes (EF Core operational store only),
   - Runs the core seeding pipeline (automatic cron seeding + any configured `UseTickerSeeder`),
   - Restarts the scheduler so persisted work is picked up immediately.

## DbContext options

- **[Built-in `TickerQDbContext`](./setup/built-in-dbcontext)** – optimized schema with its own migrations; best default.
- **[Application DbContext](./setup/application-dbcontext)** – merges TickerQ entities into an existing EF model; perfect when you need cross-entity queries.
- **[Custom Entities](./setup/custom-entities)** – extend the entities with additional columns or multi-tenant discriminators.

## Operational store capabilities

| Capability | Description |
|------------|-------------|
| Pool sizing | `SetDbContextPoolSize` configures the EF Core pool used by background services. |
| Provider configuration | Configure provider-specific options via the `optionsBuilder` lambda (e.g., `UseNpgsql`, command timeouts). |
| Dashboard queries | Same context feeds the dashboard endpoints so UI gets consistent data. |
| Multi-node coordination | EF-backed `ReleaseDeadNodeResources` prevents duplicate executions when nodes crash. |

> **Important:** Use the DbContext for querying/reporting only. Creating, updating, or deleting active ticker entities directly through EF skips the scheduler's signaling pipeline (queues, caches, SignalR). Always use `ITimeTickerManager` / `ICronTickerManager` (or their interfaces via DI) for mutations. Direct EF deletes are only appropriate for maintenance on completed or historical data.

### Example: query with EF, mutate with managers

```csharp
public class JobService
{
    private readonly MyDbContext _context;
    private readonly ITimeTickerManager<TimeTickerEntity> _timeTickerManager;

    public JobService(
        MyDbContext context,
        ITimeTickerManager<TimeTickerEntity> timeTickerManager)
    {
        _context = context;
        _timeTickerManager = timeTickerManager;
    }

    public async Task CancelFailedJobsAsync()
    {
        // Use EF Core for querying
        var failedJobIds = await _context.Set<TimeTickerEntity>()
            .Where(t => t.Status == TickerStatus.Failed)
            .Select(t => t.Id)
            .ToListAsync();

        // Use manager APIs for mutations
        foreach (var jobId in failedJobIds)
        {
            // Delete scheduled jobs via manager so the scheduler and dashboard are notified
            await _timeTickerManager.DeleteAsync(jobId);
        }
    }
}
```

## Quick setup (TickerQDbContext)

```csharp
using TickerQ.EntityFrameworkCore.DbContextFactory;

builder.Services.AddTickerQ(options =>
{
    options.AddOperationalStore(efOptions =>
    {
        efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
        {
            optionsBuilder
                .UseNpgsql(builder.Configuration.GetConnectionString("TickerQ"))
                .EnableSensitiveDataLogging(false);
        });

        efOptions.SetDbContextPoolSize(64);
    });
});
```

Once configured, run migrations using the DbContext you're targeting, and the scheduler will start persisting work automatically.

Job seeding (for example, creating initial cron/time tickers from code) is configured at the TickerQ core level rather than in the EF provider. See [Seeding configuration](/api-reference/configuration/seeding/index) for details.

## Sections

- [Installation](./installation) – Package install commands and prerequisites.
- [Setup](./setup/index) – Pick your DbContext strategy and configure it.
- [Migrations](./migrations) – Generate and apply schema changes.
- [Database Operations](./database-operations) – Query TickerQ entities with EF and mutate via managers.
- [Performance](./performance) – Pooling, indexes, and transaction guidance.
- [Best Practices](./best-practices) – Production recommendations specific to TickerQ.

## See Also

- [Configuration Reference](/api-reference/configuration/entity-framework-configuration) - Complete API reference
- [Entity Reference](/api-reference/entities/index) - Entity properties and types
- [Dashboard](/features/dashboard) - Visual job management
- [Redis Integration](/features/redis) - Multi-node coordination
