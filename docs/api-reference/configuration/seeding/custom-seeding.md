# Custom Seeding

Use custom seeding logic to programmatically create initial jobs with full control over job properties, request data, and business logic.

## UseTickerSeeder

Configure custom seeding functions for TimeTicker and CronTicker jobs. Under the hood, EF Core and the in-memory provider both forward to the same core seeding pipeline.

### EF Core (operational store)

**Method:**
```csharp
TickerQEfCoreOptionBuilder<TTimeTicker, TCronTicker> UseTickerSeeder(
    Func<ITimeTickerManager<TTimeTicker>, Task> timeTickerAsync,
    Func<ICronTickerManager<TCronTicker>, Task> cronTickerAsync);
```

### Core / In-Memory (no EF)

You can also use `UseTickerSeeder` directly on the core `TickerOptionsBuilder`. This works:

- With the in-memory provider only (no EF Core).
- With EF Core, when you want seeding logic that applies regardless of which persistence provider is active.

**Methods (core):**
```csharp
TickerOptionsBuilder<TTimeTicker, TCronTicker> UseTickerSeeder(
    Func<ITimeTickerManager<TTimeTicker>, Task> timeSeeder);

TickerOptionsBuilder<TTimeTicker, TCronTicker> UseTickerSeeder(
    Func<ICronTickerManager<TCronTicker>, Task> cronSeeder);

TickerOptionsBuilder<TTimeTicker, TCronTicker> UseTickerSeeder(
    Func<ITimeTickerManager<TTimeTicker>, Task> timeSeeder,
    Func<ICronTickerManager<TCronTicker>, Task> cronSeeder);
```

## Basic Example (EF Core)

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddOperationalStore(efOptions =>
    {
        efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
        {
            optionsBuilder.UseNpgsql(connectionString);
        });
        
        // Custom seeding
        efOptions.UseTickerSeeder(
            async timeManager =>
            {
                // Seed TimeTicker jobs
                await timeManager.AddAsync(new TimeTickerEntity
                {
                    Function = "SendWelcomeEmail",
                    Description = "Welcome email for new users",
                    ExecutionTime = DateTime.UtcNow.AddDays(1),
                    Request = TickerHelper.CreateTickerRequest(new { Message = "Welcome!" })
                });
            },
            async cronManager =>
            {
                // Seed CronTicker jobs
                await cronManager.AddAsync(new CronTickerEntity
                {
                    Function = "GenerateMonthlyReport",
                    Description = "Monthly financial report",
                    Expression = "0 0 1 1 * *", // First day of each month at midnight
                    Request = TickerHelper.CreateTickerRequest(new { ReportType = "Monthly" })
                });
            }
        );
    });
});
```

## Seeding with Request Data (EF Core)

Include serialized request data in seeded jobs:

```csharp
efOptions.UseTickerSeeder(
    async timeManager =>
    {
        var welcomeEmailRequest = new EmailRequest
        {
            TemplateId = "welcome-template-001",
            SendTo = "user@example.com"
        };
        
        await timeManager.AddAsync(new TimeTickerEntity
        {
            Function = "SendWelcomeEmail",
            ExecutionTime = DateTime.UtcNow.AddHours(1),
            Request = TickerHelper.CreateTickerRequest(welcomeEmailRequest),
            Retries = 3,
            RetryIntervals = new[] { 60, 300, 900 }
        });
    },
    async cronManager =>
    {
        var reportRequest = new ReportRequest
        {
            ReportType = "Daily",
            Recipients = new[] { "admin@example.com" }
        };
        
        await cronManager.AddAsync(new CronTickerEntity
        {
            Function = "GenerateDailyReport",
            Expression = "0 0 9 * * *", // Daily at 9 AM
            Request = TickerHelper.CreateTickerRequest(reportRequest),
            Retries = 2,
            RetryIntervals = new[] { 300, 900 }
        });
    }
);
```

## Seeding Multiple Jobs (EF Core)

Seed multiple jobs in a single seeding function:

```csharp
efOptions.UseTickerSeeder(
    async timeManager =>
    {
        var jobs = new[]
        {
            new TimeTickerEntity
            {
                Function = "ProcessOrder",
                ExecutionTime = DateTime.UtcNow.AddMinutes(30),
                Request = TickerHelper.CreateTickerRequest(new { OrderId = 123 })
            },
            new TimeTickerEntity
            {
                Function = "SendConfirmation",
                ExecutionTime = DateTime.UtcNow.AddHours(1),
                Request = TickerHelper.CreateTickerRequest(new { OrderId = 123 })
            }
        };
        
        await timeManager.AddBatchAsync(jobs);
    },
    async cronManager =>
    {
        var cronJobs = new[]
        {
            new CronTickerEntity
            {
                Function = "CleanupLogs",
                Expression = "0 0 0 * * *" // Daily at midnight
            },
            new CronTickerEntity
            {
                Function = "BackupDatabase",
                Expression = "0 0 2 * * *" // Daily at 2 AM
            },
            new CronTickerEntity
            {
                Function = "GenerateReport",
                Expression = "0 0 9 * * 1" // Every Monday at 9 AM
            }
        };
        
        await cronManager.AddBatchAsync(cronJobs);
    }
);

## Core / In-Memory Seeding Example

Use `UseTickerSeeder` directly on `AddTickerQ` when you are not using `AddOperationalStore` (in-memory), or when you want seeding logic that applies regardless of persistence provider:

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

## Conditional Seeding

Seed jobs based on environment or configuration:

```csharp
efOptions.UseTickerSeeder(
    async timeManager =>
    {
        // Only seed in production
        if (builder.Environment.IsProduction())
        {
            await timeManager.AddAsync(new TimeTickerEntity
            {
                Function = "ProductionOnlyJob",
                ExecutionTime = DateTime.UtcNow.AddHours(24)
            });
        }
    },
    async cronManager =>
    {
        // Seed based on configuration
        var reportInterval = builder.Configuration["Reports:Interval"];
        
        if (reportInterval == "Daily")
        {
            await cronManager.AddAsync(new CronTickerEntity
            {
                Function = "GenerateDailyReport",
                Expression = "0 0 9 * * *"
            });
        }
        else if (reportInterval == "Weekly")
        {
            await cronManager.AddAsync(new CronTickerEntity
            {
                Function = "GenerateWeeklyReport",
                Expression = "0 0 9 * * 1"
            });
        }
    }
);
```

## Seeding with Database Checks

Check if jobs already exist before seeding:

```csharp
efOptions.UseTickerSeeder(
    null, // No TimeTicker seeding
    async cronManager =>
    {
        // Use persistence provider to check existing jobs
        // (This requires injecting the provider in a different way)
        // For now, use idempotent seeding - upsert logic in AddAsync
        
        await cronManager.AddAsync(new CronTickerEntity
        {
            Function = "UniqueJobName",
            Expression = "0 0 0 * * *"
        });
        
        // Note: Manager's AddAsync will handle duplicates based on validation
    }
);
```

## Accessing Services in Seeder

Since seeders run during application startup, you can't directly inject services. However, you can access configuration:

```csharp
var configuration = builder.Configuration;

efOptions.UseTickerSeeder(
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
```

## Error Handling

Handle errors in seeding functions:

```csharp
efOptions.UseTickerSeeder(
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
    options.AddOperationalStore(efOptions =>
    {
        efOptions.UseTickerSeeder(
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
