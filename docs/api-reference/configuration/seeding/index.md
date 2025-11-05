# Seeding

TickerQ supports multiple seeding strategies for initializing jobs in the database when the application starts.

## Seeding Strategies

TickerQ provides three seeding mechanisms:

### 1. [Automatic Cron Seeding](./automatic-seeding)
Automatic seeding of cron jobs from `[TickerFunction]` attributes with cron expressions.

### 2. [Custom Seeding](./custom-seeding)
Programmatic seeding using custom logic for TimeTicker and CronTicker jobs.

### 3. [Disabling Seeding](./disable-seeding)
How to prevent automatic seeding when you manage jobs manually.

## Seeding Order

When using Entity Framework Core persistence, seeding occurs in the following order on application startup:

1. **Dead node resource cleanup** - Releases resources from dead nodes
2. **Automatic cron seeding** (if enabled) - Seeds cron jobs from `[TickerFunction]` attributes
3. **Custom time ticker seeding** (if configured) - Executes `UseTickerSeeder` time ticker function
4. **Custom cron ticker seeding** (if configured) - Executes `UseTickerSeeder` cron ticker function

## Seeding Lifecycle

Seeding happens automatically when:
- Application starts (via `IHostApplicationLifetime.ApplicationStarted`)
- Entity Framework Core persistence is configured
- `UseTickerQ()` is called

All seeding operations are logged through the instrumentation system, allowing you to track seeding progress.

## See Also

- [Automatic Cron Seeding](./automatic-seeding) - Learn about attribute-based seeding
- [Custom Seeding](./custom-seeding) - Implement your own seeding logic
- [Disable Seeding](./disable-seeding) - Prevent automatic seeding
- [Entity Framework Configuration](../entity-framework-configuration) - Complete EF Core setup

