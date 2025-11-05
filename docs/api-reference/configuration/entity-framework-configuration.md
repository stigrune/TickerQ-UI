# Entity Framework Configuration

Configure Entity Framework Core persistence for TickerQ job storage and management.

## Configuration Sections

### [DbContext Setup](./entity-framework-configuration/dbcontext-setup)
Configure which DbContext to use: built-in `TickerQDbContext` or your application's DbContext.

### [Connection & Pooling](./entity-framework-configuration/connection-pooling)
Configure database connections, connection pooling, and schema settings.

### [Seeding](./seeding/index)
Initialize jobs in the database when the application starts.

## Quick Example

```csharp
using TickerQ.EntityFrameworkCore.DbContextFactory;

options.AddOperationalStore(efOptions =>
{
    // Use built-in TickerQDbContext
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        optionsBuilder.UseNpgsql(connectionString);
    });
    
    // Configure pool size
    efOptions.SetDbContextPoolSize(34);
});
```

## AddOperationalStore

The entry point for Entity Framework Core configuration.

**Method:**
```csharp
// For built-in TickerQDbContext (recommended)
TickerOptionsBuilder<TTimeTicker, TCronTicker> AddOperationalStore(
    Action<TickerQEfCoreOptionBuilder<TTimeTicker, TCronTicker>> efOptions);

// For application DbContext
TickerOptionsBuilder<TTimeTicker, TCronTicker> AddOperationalStore<TDbContext>(
    Action<TickerQEfCoreOptionBuilder<TTimeTicker, TCronTicker>> efOptions)
    where TDbContext : DbContext;
```

## See Also

- [DbContext Setup](./entity-framework-configuration/dbcontext-setup) - Choose and configure DbContext
- [Connection & Pooling](./entity-framework-configuration/connection-pooling) - Database connection settings
- [Seeding Guide](./seeding/index) - Initialize jobs on startup
- [Entity Framework Guide](../../features/entity-framework) - Complete setup guide
- [Configuration Overview](./index) - All configuration sections

