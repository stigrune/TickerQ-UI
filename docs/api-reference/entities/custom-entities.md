# Custom Entity Types

Creating custom entity types that extend base TickerQ entities.

## Custom TimeTicker

Create a custom TimeTicker with additional properties:

```csharp
public class CustomTimeTicker : TimeTickerEntity<CustomTimeTicker>
{
    public string TenantId { get; set; }
    public string UserId { get; set; }
    public string Category { get; set; }
}
```

## Custom CronTicker

Create a custom CronTicker with additional properties:

```csharp
public class CustomCronTicker : CronTickerEntity
{
    public string Category { get; set; }
    public bool IsActive { get; set; }
    public string Environment { get; set; }
}
```

## Configuration

When using custom entities, you must:

### 1. Create Custom DbContext (Recommended)

Create a custom DbContext that extends `TickerQDbContext` with your custom entity types:

```csharp
using TickerQ.EntityFrameworkCore.DbContextFactory;

public class MyTickerQDbContext : TickerQDbContext<CustomTimeTicker, CustomCronTicker>
{
    public MyTickerQDbContext(DbContextOptions<MyTickerQDbContext> options) 
        : base(options) { }
}
```

### 2. Configure TickerQ with Custom Types

```csharp
builder.Services.AddTickerQ<CustomTimeTicker, CustomCronTicker>(options =>
{
    options.AddOperationalStore(efOptions =>
    {
        // Use custom DbContext with connection string
        efOptions.UseTickerQDbContext<MyTickerQDbContext>(optionsBuilder =>
        {
            optionsBuilder.UseNpgsql("Server=localhost;Port=5432;Database=TickerQ;User Id=postgres;Password=postgres;", 
                cfg =>
                {
                    cfg.EnableRetryOnFailure(3, TimeSpan.FromSeconds(5), ["40P01"]);
                });
        });
        efOptions.SetDbContextPoolSize(34);
    });
});
```

**Alternative: Use Application DbContext**

If you prefer to use your existing application DbContext:

```csharp
builder.Services.AddTickerQ<CustomTimeTicker, CustomCronTicker>(options =>
{
    options.AddOperationalStore<MyApplicationDbContext>(efOptions =>
    {
        efOptions.UseApplicationDbContext<MyApplicationDbContext>(ConfigurationType.UseModelCustomizer);
    });
});
```

### 3. Inject Correct Manager Types

```csharp
public class MyService
{
    private readonly ITimeTickerManager<CustomTimeTicker> _timeTickerManager;
    private readonly ICronTickerManager<CustomCronTicker> _cronTickerManager;
    
    public MyService(
        ITimeTickerManager<CustomTimeTicker> timeTickerManager,
        ICronTickerManager<CustomCronTicker> cronTickerManager)
    {
        _timeTickerManager = timeTickerManager;
        _cronTickerManager = cronTickerManager;
    }
    
    public async Task ScheduleJobAsync()
    {
        await _timeTickerManager.AddAsync(new CustomTimeTicker
        {
            Function = "MyJob",
            ExecutionTime = DateTime.UtcNow.AddMinutes(5),
            TenantId = "tenant-123",
            UserId = "user-456"
        });
    }
}
```

## Querying Custom Entities

When using Entity Framework Core:

```csharp
// Query by custom properties
var jobs = await _context.Set<CustomTimeTicker>()
    .Where(t => t.TenantId == "tenant-123")
    .Where(t => t.Category == "email")
    .ToListAsync();

// Query custom CronTickers
var activeCronJobs = await _context.Set<CustomCronTicker>()
    .Where(c => c.IsActive && c.Environment == "Production")
    .ToListAsync();
```

## Best Practices

1. **Use meaningful property names** that describe their purpose
2. **Add indexes** for frequently queried custom properties
3. **Consider multi-tenancy** when adding tenant-specific properties
4. **Document custom properties** in your codebase
5. **Use migrations** to manage schema changes

## See Also

- [BaseTickerEntity](./base-entity) - Base class reference
- [TimeTickerEntity](./time-ticker-entity) - Base TimeTicker entity
- [CronTickerEntity](./cron-ticker-entity) - Base CronTicker entity
- [Entity Framework Configuration](../configuration/entity-framework-configuration) - EF Core setup
- [Entity Framework Guide](../../features/entity-framework) - Complete setup guide

