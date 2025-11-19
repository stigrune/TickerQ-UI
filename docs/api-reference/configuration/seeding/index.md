# Seeding

TickerQ supports multiple seeding strategies for initializing jobs when the application starts. Seeding is a **core feature** of TickerQ; it works whether you use the in-memory provider only or plug in Entity Framework Core.

## Seeding Strategies

TickerQ provides three seeding mechanisms:

### 1. [Automatic Cron Seeding](./automatic-seeding)
Automatic seeding of cron jobs from `[TickerFunction]` attributes with cron expressions.

### 2. [Custom Seeding](./custom-seeding)
Programmatic seeding using custom logic for TimeTicker and CronTicker jobs.

### 3. [Disabling Seeding](./disable-seeding)
How to prevent automatic seeding when you manage jobs manually.

## Seeding Order

Regardless of the persistence provider (in-memory or EF Core), seeding follows the same logical order:

1. **Dead node resource cleanup** (when EF Core persistence is used) – releases resources from dead nodes.
2. **Automatic cron seeding** (if enabled) – seeds cron jobs from `[TickerFunction]` attributes.
3. **Custom time ticker seeding** (if configured) – executes `UseTickerSeeder` time ticker function.
4. **Custom cron ticker seeding** (if configured) – executes `UseTickerSeeder` cron ticker function.

## Seeding Lifecycle

Seeding is orchestrated by the TickerQ core and executed in one of two ways:

- **Core-only / In-memory path**  
  - You call `builder.Services.AddTickerQ(...)` and `app.UseTickerQ(...)` without `AddOperationalStore`.  
  - During `UseTickerQ`, the core runs automatic cron seeding and any custom seeders configured via `TickerOptionsBuilder` (unless you disable them).

- **Entity Framework Core path**  
  - You call `AddTickerQ(...).AddOperationalStore(...)` to configure EF persistence.  
  - The EF provider hooks into `IHostApplicationLifetime.ApplicationStarted`, performs dead node cleanup plus the same automatic + custom seeding, then restarts the scheduler.

In both cases:

- Auto seeding is controlled by core configuration (for example, `IgnoreSeedDefinedCronTickers()`).
- Custom seeders are registered via `UseTickerSeeder(...)` and executed using manager APIs (`ITimeTickerManager` / `ICronTickerManager`).
- All seeding operations are logged through the instrumentation system, allowing you to track seeding progress.

## See Also

- [Automatic Cron Seeding](./automatic-seeding) - Learn about attribute-based seeding
- [Custom Seeding](./custom-seeding) - Implement your own seeding logic
- [Disable Seeding](./disable-seeding) - Prevent automatic seeding
- [Entity Framework Configuration](../entity-framework-configuration) - Complete EF Core setup
