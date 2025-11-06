# DbContext Setup

Configure which DbContext TickerQ uses for persistence. Choose between the built-in `TickerQDbContext` or your application's existing DbContext.

## Options

### [Use Built-in TickerQDbContext](./dbcontext-setup#using-built-in-tickerqdbcontext-recommended)
Lightweight, optimized DbContext designed specifically for TickerQ (recommended).

### [Use Application DbContext](./dbcontext-setup#using-application-dbcontext)
Integrate TickerQ entities into your existing application DbContext.

## Using Built-in TickerQDbContext (Recommended)

The built-in `TickerQDbContext` is lightweight, optimized for TickerQ, and keeps job persistence separate from your application entities.

### UseTickerQDbContext

Configure the built-in or custom TickerQ DbContext.

**Method:**
```csharp
TickerQEfCoreOptionBuilder<TTimeTicker, TCronTicker> UseTickerQDbContext<TDbContext>(
    Action<DbContextOptionsBuilder> optionsAction,
    string schema = null)
    where TDbContext : TickerQDbContext<TTimeTicker, TCronTicker>;
```

**Using Built-in Context:**
```csharp
using TickerQ.EntityFrameworkCore.DbContextFactory;

options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        // PostgreSQL example
        optionsBuilder.UseNpgsql("Server=localhost;Port=5432;Database=TickerQ;User Id=postgres;Password=postgres;", 
            cfg =>
            {
                cfg.EnableRetryOnFailure(3, TimeSpan.FromSeconds(5), ["40P01"]);
            });
        
        // SQL Server example
        // optionsBuilder.UseSqlServer("Server=localhost;Database=TickerQ;Integrated Security=true;");
    }, schema: "ticker");
});
```

**Benefits:**
- Lightweight and optimized for TickerQ
- No mixing with application entities
- Simpler configuration
- Connection string configured directly in TickerQ options
- Easier to manage separately

## Using Custom Entity Types

When using custom entity types, create your own TickerQ DbContext:

```csharp
using TickerQ.EntityFrameworkCore.DbContextFactory;

public class CustomTimeTicker : TimeTickerEntity<CustomTimeTicker>
{
    public string TenantId { get; set; }
}

public class CustomCronTicker : CronTickerEntity
{
    public string Category { get; set; }
}

public class MyTickerQDbContext : TickerQDbContext<CustomTimeTicker, CustomCronTicker>
{
    public MyTickerQDbContext(DbContextOptions<MyTickerQDbContext> options) 
        : base(options) { }
}

// Configuration
builder.Services.AddTickerQ<CustomTimeTicker, CustomCronTicker>(options =>
{
    options.AddOperationalStore(efOptions =>
    {
        efOptions.UseTickerQDbContext<MyTickerQDbContext>(optionsBuilder =>
        {
            optionsBuilder.UseNpgsql(connectionString);
        });
    });
});
```

## Using Application DbContext

Integrate TickerQ entities into your existing application DbContext.

### UseApplicationDbContext

Configure TickerQ to use your application's DbContext.

**Method:**
```csharp
TickerQEfCoreOptionBuilder<TTimeTicker, TCronTicker> UseApplicationDbContext<TDbContext>(
    ConfigurationType configurationType)
    where TDbContext : DbContext;
```

**Configuration Types:**
- `ConfigurationType.UseModelCustomizer` - Automatically apply TickerQ entity configurations (recommended)
- `ConfigurationType.IgnoreModelCustomizer` - Manual configuration in `OnModelCreating`

**Example:**
```csharp
options.AddOperationalStore<MyApplicationDbContext>(efOptions =>
{
    efOptions.UseApplicationDbContext<MyApplicationDbContext>(ConfigurationType.UseModelCustomizer);
    efOptions.SetDbContextPoolSize(128);
});
```

**When to Use:**
- You want to share the same database connection pool
- You need to join TickerQ entities with your application entities
- You have existing migrations and prefer to manage everything together
- Your application already has a well-established DbContext

**Example Application DbContext:**
```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }
    
    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }
    // TickerQ entities added automatically via UseModelCustomizer
}
```

**With Manual Configuration:**
```csharp
options.AddOperationalStore<MyApplicationDbContext>(efOptions =>
{
    efOptions.UseApplicationDbContext<MyApplicationDbContext>(ConfigurationType.IgnoreModelCustomizer);
});

// In your DbContext
public class MyApplicationDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        // Manually apply TickerQ configurations
        builder.ApplyConfiguration(new TimeTickerConfigurations());
        builder.ApplyConfiguration(new CronTickerConfigurations());
        builder.ApplyConfiguration(new CronTickerOccurrenceConfigurations());
    }
}
```

## Comparison

| Feature | TickerQDbContext | Application DbContext |
|---------|------------------|----------------------|
| **Setup Complexity** | Simple | More complex |
| **Connection Sharing** | Separate | Shared pool |
| **Entity Mixing** | Isolated | Mixed |
| **Migrations** | Separate | Combined |
| **Querying** | Separate context | Single context |
| **Recommended For** | Most scenarios | When integration needed |

## Database Providers

Both approaches support all Entity Framework Core providers:

### PostgreSQL
```csharp
optionsBuilder.UseNpgsql(connectionString, cfg =>
{
    cfg.EnableRetryOnFailure(3, TimeSpan.FromSeconds(5), ["40P01"]);
});
```

### SQL Server
```csharp
optionsBuilder.UseSqlServer(connectionString);
```

### MySQL
```csharp
optionsBuilder.UseMySql(connectionString, ServerVersion.AutoDetect(connectionString));
```

### SQLite
```csharp
optionsBuilder.UseSqlite(connectionString);
```

## Schema Configuration

When using `UseTickerQDbContext`, specify schema as a parameter:

```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseNpgsql(connectionString);
}, schema: "ticker"); // Schema parameter
```

## See Also

- [Connection & Pooling](./connection-pooling) - Database connection settings
- [Seeding Guide](../seeding/index) - Initialize jobs on startup

