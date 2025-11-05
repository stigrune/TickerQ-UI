# Connection & Pooling

Configure database connections, connection pooling, and schema settings for TickerQ persistence.

## SetDbContextPoolSize

Configure the DbContext pool size for connection pooling.

**Method:**
```csharp
TickerQEfCoreOptionBuilder<TTimeTicker, TCronTicker> SetDbContextPoolSize(int poolSize);
```

**Type:** `int`  
**Default:** `1024`  
**Range:** Must be greater than 0

```csharp
options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        optionsBuilder.UseNpgsql(connectionString);
    });
    
    efOptions.SetDbContextPoolSize(34);
});
```

## Understanding Pool Size

The pool size determines how many DbContext instances are kept in the pool for reuse:

- **Higher values**: Better performance under load, more memory usage
- **Lower values**: Less memory, potential connection waits under load
- **Default (1024)**: Suitable for most applications

## Recommended Pool Sizes

### Small Applications
```csharp
efOptions.SetDbContextPoolSize(16);
```
- Low job volume (< 100 jobs/minute)
- Single node deployment
- Limited memory constraints

### Medium Applications
```csharp
efOptions.SetDbContextPoolSize(34);
```
- Moderate job volume (100-1000 jobs/minute)
- 1-3 node deployment
- Standard server resources

### Large Applications
```csharp
efOptions.SetDbContextPoolSize(128);
```
- High job volume (1000+ jobs/minute)
- Multi-node deployment
- High-performance servers

### Very High Throughput
```csharp
efOptions.SetDbContextPoolSize(256);
```
- Very high job volume (10000+ jobs/minute)
- Many nodes
- Dedicated database servers

## Schema Configuration

When using `UseTickerQDbContext`, you can specify a database schema:

```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseNpgsql(connectionString);
}, schema: "ticker"); // Schema name
```

**Default Schema:** `"ticker"`

**Benefits of Custom Schema:**
- Organizes TickerQ tables separately
- Easier to manage permissions
- Better for multi-tenant scenarios
- Cleaner database structure

**Example with Custom Schema:**
```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseNpgsql(connectionString);
}, schema: "job_scheduler"); // Custom schema
```

## Connection String Configuration

### PostgreSQL
```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseNpgsql("Server=localhost;Port=5432;Database=TickerQ;User Id=postgres;Password=postgres;", 
        cfg =>
        {
            // Enable retry on transient failures
            cfg.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorCodesToAdd: ["40P01"] // Deadlock detection code
            );
        });
});
```

### SQL Server
```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseSqlServer("Server=localhost;Database=TickerQ;Integrated Security=true;",
        cfg =>
        {
            cfg.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorCodesToAdd: null
            );
        });
});
```

### From Configuration
```csharp
var connectionString = builder.Configuration.GetConnectionString("TickerQ");

efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseNpgsql(connectionString);
});
```

## Connection Pooling Best Practices

1. **Start with default** - 1024 is usually sufficient
2. **Monitor performance** - Adjust based on actual usage
3. **Consider database limits** - Don't exceed database connection limits
4. **Match concurrency** - Pool size should be >= MaxConcurrency
5. **Use connection retry** - Enable retry on failure for transient errors

## Database Connection Limits

Ensure your pool size doesn't exceed database connection limits:

```csharp
// Example: PostgreSQL default max connections is 100
// For 3 nodes, don't use more than 33 per node
efOptions.SetDbContextPoolSize(30); // Leave room for other connections
```

## Performance Tuning

### High Concurrency Scenario
```csharp
options.ConfigureScheduler(scheduler =>
{
    scheduler.MaxConcurrency = 20; // High concurrency
});

options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        optionsBuilder.UseNpgsql(connectionString);
    });
    
    // Pool size should be higher than max concurrency
    efOptions.SetDbContextPoolSize(50); // At least 2x MaxConcurrency
});
```

### Low Resource Scenario
```csharp
options.ConfigureScheduler(scheduler =>
{
    scheduler.MaxConcurrency = 4; // Low concurrency
});

options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        optionsBuilder.UseNpgsql(connectionString);
    });
    
    efOptions.SetDbContextPoolSize(16); // Reasonable for low load
});
```

## Troubleshooting

### Connection Pool Exhausted

If you see "connection pool exhausted" errors:
- Increase pool size
- Check for connection leaks
- Verify database connection limits

### High Memory Usage

If memory usage is high:
- Reduce pool size
- Monitor actual pool utilization
- Use appropriate size for your workload

## See Also

- [DbContext Setup](./dbcontext-setup) - Choose and configure DbContext
- [Seeding Guide](../seeding/index) - Initialize jobs on startup
- [Scheduler Configuration](../core-configuration/scheduler-configuration) - Concurrency settings
- [Entity Framework Guide](../../../features/entity-framework) - Complete setup guide

