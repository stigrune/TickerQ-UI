# Custom Entity Types

Create custom entity types that extend base TickerQ entities to add application-specific properties, relationships, and behavior.

## Overview

TickerQ supports custom entity types through inheritance. You can extend `TimeTickerEntity` and `CronTickerEntity` to add custom properties while maintaining full TickerQ functionality.

## Creating Custom Entities

### Custom TimeTicker

```csharp
public class CustomTimeTicker : TimeTickerEntity<CustomTimeTicker>
{
    public string TenantId { get; set; }
    public string UserId { get; set; }
    public string Category { get; set; }
    public int PriorityLevel { get; set; }
}
```

### Custom CronTicker

```csharp
public class CustomCronTicker : CronTickerEntity
{
    public string Category { get; set; }
    public bool IsActive { get; set; }
    public string Environment { get; set; }
    public string Owner { get; set; }
}
```

## Creating Custom DbContext

Create a custom DbContext that extends `TickerQDbContext` with your custom entity types:

```csharp
using TickerQ.EntityFrameworkCore.DbContextFactory;

public class MyTickerQDbContext : TickerQDbContext<CustomTimeTicker, CustomCronTicker>
{
    public MyTickerQDbContext(DbContextOptions<MyTickerQDbContext> options) 
        : base(options) { }
    
    // Optionally add custom DbSets or configuration
}
```

## Configuration

Configure TickerQ to use your custom entity types:

```csharp
builder.Services.AddTickerQ<CustomTimeTicker, CustomCronTicker>(options =>
{
    options.AddOperationalStore(efOptions =>
    {
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

## Using Custom Properties

Schedule jobs with custom properties:

```csharp
public class JobService
{
    private readonly ITimeTickerManager<CustomTimeTicker> _timeTickerManager;
    
    public async Task ScheduleTenantJobAsync(string tenantId, string functionName)
    {
        await _timeTickerManager.AddAsync(new CustomTimeTicker
        {
            Function = functionName,
            ExecutionTime = DateTime.UtcNow.AddMinutes(5),
            TenantId = tenantId,
            Category = "tenant-specific",
            PriorityLevel = 10
        });
    }
}
```

## Querying Custom Properties

Query jobs using custom properties:

```csharp
public class CustomJobRepository
{
    private readonly MyTickerQDbContext _context;
    
    public async Task<List<CustomTimeTicker>> GetJobsByTenantAsync(string tenantId)
    {
        return await _context.Set<CustomTimeTicker>()
            .Where(t => t.TenantId == tenantId)
            .ToListAsync();
    }
    
    public async Task<List<CustomCronTicker>> GetActiveCronJobsAsync()
    {
        return await _context.Set<CustomCronTicker>()
            .Where(c => c.IsActive)
            .ToListAsync();
    }
}
```

## Adding Indexes

Create indexes for frequently queried custom properties:

```csharp
public class CustomTimeTickerConfiguration : IEntityTypeConfiguration<CustomTimeTicker>
{
    public void Configure(EntityTypeBuilder<CustomTimeTicker> builder)
    {
        builder.HasIndex(t => t.TenantId);
        builder.HasIndex(t => new { t.TenantId, t.Category });
        builder.HasIndex(t => t.PriorityLevel);
    }
}

// Apply in DbContext
public class MyTickerQDbContext : TickerQDbContext<CustomTimeTicker, CustomCronTicker>
{
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.ApplyConfiguration(new CustomTimeTickerConfiguration());
    }
}
```

## Migration Considerations

When creating migrations for custom entities:

```bash
dotnet ef migrations add CustomEntitiesInitialCreate --context MyTickerQDbContext
dotnet ef database update --context MyTickerQDbContext
```

Ensure migrations include:
- Base TickerQ entity properties
- Custom properties
- Indexes on custom properties

## Use Cases

### Multi-Tenancy

```csharp
public class TenantTimeTicker : TimeTickerEntity<TenantTimeTicker>
{
    public string TenantId { get; set; }
    public string OrganizationId { get; set; }
}

// Query tenant-specific jobs
var tenantJobs = await _context.Set<TenantTimeTicker>()
    .Where(t => t.TenantId == currentTenantId)
    .ToListAsync();
```

### Environment-Specific Cron Jobs

```csharp
public class EnvironmentCronTicker : CronTickerEntity
{
    public string Environment { get; set; }
    public bool ProductionOnly { get; set; }
}

// Only seed production jobs in production
if (environment.IsProduction())
{
    await _cronManager.AddAsync(new EnvironmentCronTicker
    {
        Function = "ProductionJob",
        Expression = "0 0 0 * * *",
        Environment = "Production",
        ProductionOnly = true
    });
}
```

### Category-Based Organization

```csharp
public class CategorizedTimeTicker : TimeTickerEntity<CategorizedTimeTicker>
{
    public string Category { get; set; }
    public string SubCategory { get; set; }
    public Dictionary<string, object> Metadata { get; set; }
}
```

## Best Practices

1. **Use Meaningful Names**: Custom properties should clearly indicate their purpose
2. **Add Indexes**: Create indexes for frequently queried custom properties
3. **Document Properties**: Document what each custom property is used for
4. **Consider Migrations**: Plan migration strategy for custom property changes
5. **Type Safety**: Prefer strongly-typed properties over string dictionaries when possible

## Limitations

- Custom properties are not used in TickerQ's internal job matching logic
- Base entity properties (Function, ExecutionTime, Expression) still govern job behavior
- Custom properties are stored but not validated by TickerQ core

## See Also

- [Built-in TickerQDbContext](./built-in-dbcontext) - Standard setup
- [Application DbContext](./application-dbcontext) - Integration setup
- [Custom Entities API Reference](/api-reference/entities/custom-entities) - Complete API documentation
- [Migrations](../migrations) - Database migration guide

