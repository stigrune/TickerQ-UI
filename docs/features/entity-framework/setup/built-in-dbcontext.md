# Using Built-in TickerQDbContext

Use the lightweight, optimized `TickerQDbContext` designed specifically for TickerQ. This is the recommended approach for most production scenarios.

## Why TickerQDbContext?

The built-in `TickerQDbContext` offers:

- **Lightweight**: Optimized specifically for TickerQ job storage
- **Isolated**: Keeps job entities separate from your application entities
- **Simple Configuration**: Connection string configured directly in TickerQ options
- **Easy Management**: Separate database or schema for job data

## Basic Setup

```csharp
using TickerQ.DependencyInjection;
using TickerQ.EntityFrameworkCore.DependencyInjection;
using TickerQ.EntityFrameworkCore.DbContextFactory;

builder.Services.AddTickerQ(options =>
{
    options.AddOperationalStore(efOptions =>
    {
        // Use built-in TickerQDbContext with connection string
        efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
        {
            optionsBuilder.UseNpgsql("Server=localhost;Port=5432;Database=TickerQ;User Id=postgres;Password=postgres;", 
                cfg =>
                {
                    cfg.EnableRetryOnFailure(3, TimeSpan.FromSeconds(5), ["40P01"]);
                });
        });
        
        // Optional: Configure pool size
        efOptions.SetDbContextPoolSize(34);
    });
});
```

## Database Provider Examples

### PostgreSQL

```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseNpgsql("Server=localhost;Port=5432;Database=TickerQ;User Id=postgres;Password=postgres;", 
        cfg =>
        {
            cfg.EnableRetryOnFailure(3, TimeSpan.FromSeconds(5), ["40P01"]);
        });
}, schema: "ticker");
```

### SQL Server

```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseSqlServer("Server=localhost;Database=TickerQ;Integrated Security=true;", 
        cfg =>
        {
            cfg.EnableRetryOnFailure(3, TimeSpan.FromSeconds(5));
        });
}, schema: "ticker");
```

### MySQL

```csharp
var connectionString = "Server=localhost;Database=TickerQ;User=root;Password=password;";
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseMySql(connectionString, ServerVersion.AutoDetect(connectionString));
}, schema: "ticker");
```

### SQLite

```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseSqlite("Data Source=tickerq.db");
});
```

## Connection String from Configuration

```csharp
var connectionString = builder.Configuration.GetConnectionString("TickerQ");

efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseNpgsql(connectionString);
});
```

## Schema Configuration

Specify a custom database schema as a parameter:

```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseNpgsql(connectionString);
}, schema: "job_scheduler"); // Custom schema name
```

**Default Schema:** `"ticker"`

## Pool Size Configuration

Configure the DbContext pool size for optimal performance:

```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseNpgsql(connectionString);
});

efOptions.SetDbContextPoolSize(34); // Adjust based on your workload
```

See [Performance Guide](../performance) for pool size recommendations.

## Connection Retry Configuration

Enable automatic retry on transient database failures:

```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseNpgsql(connectionString, 
        cfg =>
        {
            // PostgreSQL: Retry on deadlock detection
            cfg.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorCodesToAdd: ["40P01"]
            );
        });
});
```

## Complete Production Example

```csharp
builder.Services.AddTickerQ(options =>
{
    options.ConfigureScheduler(scheduler =>
    {
        scheduler.MaxConcurrency = 8;
        scheduler.NodeIdentifier = Environment.MachineName;
    });
    
    options.AddOperationalStore(efOptions =>
    {
        var connectionString = builder.Configuration.GetConnectionString("TickerQ");
        
        efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
        {
            optionsBuilder.UseNpgsql(connectionString, 
                cfg =>
                {
                    cfg.EnableRetryOnFailure(3, TimeSpan.FromSeconds(5), ["40P01"]);
                });
        }, schema: "ticker");
        
        efOptions.SetDbContextPoolSize(34);
    });
});
```

## Next Steps

- [Create Migrations](../migrations) - Set up database schema
- [Database Operations](../database-operations) - Query and manage jobs
- [Performance Guide](../performance) - Optimize performance
- [Application DbContext Setup](./application-dbcontext) - Alternative approach

