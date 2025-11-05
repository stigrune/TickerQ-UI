# Using Application DbContext

Integrate TickerQ entities into your existing application DbContext for shared database connections and unified migrations.

## When to Use

Use your application DbContext when:

- You want to share the same database connection pool
- You need to join TickerQ entities with your application entities in queries
- You have existing migrations and prefer to manage everything together
- Your application already has a well-established DbContext setup

## Basic Setup

```csharp
using TickerQ.DependencyInjection;
using TickerQ.EntityFrameworkCore.DependencyInjection;

builder.Services.AddTickerQ(options =>
{
    options.AddOperationalStore<MyApplicationDbContext>(efOptions =>
    {
        efOptions.UseApplicationDbContext<MyApplicationDbContext>(ConfigurationType.UseModelCustomizer);
        efOptions.SetDbContextPoolSize(128);
    });
});
```

## Configuration Types

### UseModelCustomizer (Recommended)

Automatically applies TickerQ entity configurations during model building:

```csharp
options.AddOperationalStore<MyApplicationDbContext>(efOptions =>
{
    efOptions.UseApplicationDbContext<MyApplicationDbContext>(ConfigurationType.UseModelCustomizer);
});
```

**Benefits:**
- Automatic configuration application
- No manual model configuration required
- Works seamlessly with migrations

### IgnoreModelCustomizer

Use manual configuration in your DbContext's `OnModelCreating` method:

```csharp
options.AddOperationalStore<MyApplicationDbContext>(efOptions =>
{
    efOptions.UseApplicationDbContext<MyApplicationDbContext>(ConfigurationType.IgnoreModelCustomizer);
});
```

## Application DbContext Example

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }
    
    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }
    // TickerQ entities added automatically via UseModelCustomizer
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        // TickerQ configurations applied automatically
    }
}
```

## Manual Configuration

If you choose `ConfigurationType.IgnoreModelCustomizer`, manually apply configurations:

```csharp
public class MyApplicationDbContext : DbContext
{
    public MyApplicationDbContext(DbContextOptions<MyApplicationDbContext> options)
        : base(options) { }
    
    public DbSet<User> Users { get; set; }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        // Apply TickerQ entity configurations manually
        builder.ApplyConfiguration(new TimeTickerConfigurations());
        builder.ApplyConfiguration(new CronTickerConfigurations());
        builder.ApplyConfiguration(new CronTickerOccurrenceConfigurations());
    }
}
```

## Handling Conflicts

If you have other libraries that override `IModelCustomizer` (e.g., OpenIddict), you may need to:

**Option 1: Merge Customizations**

Manually merge TickerQ configurations with other customizations in `OnModelCreating`:

```csharp
protected override void OnModelCreating(ModelBuilder builder)
{
    base.OnModelCreating(builder);
    
    // Apply other library configurations
    // ... 
    
    // Apply TickerQ configurations
    builder.ApplyConfiguration(new TimeTickerConfigurations());
    builder.ApplyConfiguration(new CronTickerConfigurations());
    builder.ApplyConfiguration(new CronTickerOccurrenceConfigurations());
}
```

**Option 2: Use Separate DbContext**

Consider using the built-in `TickerQDbContext` instead if conflicts persist.

## Querying Across Entities

With application DbContext, you can query TickerQ entities alongside your application entities:

```csharp
var userJobs = await _context.Users
    .Where(u => u.Id == userId)
    .SelectMany(u => 
        _context.Set<TimeTickerEntity>()
            .Where(t => t.Request != null && /* parse request for userId */))
    .ToListAsync();
```

## Migrations

Create migrations for your application DbContext (includes TickerQ entities):

```bash
dotnet ef migrations add AddTickerQSupport --context MyApplicationDbContext
dotnet ef database update --context MyApplicationDbContext
```

## Best Practices

1. **Use Model Customizer**: Prefer `ConfigurationType.UseModelCustomizer` unless you have conflicts
2. **Monitor Conflicts**: Watch for issues with other `IModelCustomizer` implementations
3. **Separate When Needed**: Don't hesitate to use `TickerQDbContext` if conflicts arise
4. **Test Migrations**: Always test migrations in development before production

## See Also

- [Built-in TickerQDbContext](./built-in-dbcontext) - Alternative setup approach
- [Custom Entities](./custom-entities) - Using custom entity types
- [Migrations](../migrations) - Database migration guide
- [Configuration Reference](/api-reference/configuration/entity-framework-configuration/dbcontext-setup) - Complete API documentation

