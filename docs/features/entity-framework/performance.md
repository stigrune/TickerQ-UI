# Performance Optimization

Optimize Entity Framework Core integration for production performance: connection pooling, indexing, and transaction management.

## DbContext Pooling

TickerQ uses `IDbContextFactory` with pooling for efficient database connection management.

### Pool Size Configuration

```csharp
options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        optionsBuilder.UseNpgsql(connectionString);
    });
    
    efOptions.SetDbContextPoolSize(34); // Configure pool size
});
```

### Pool Size Recommendations

**Small Applications** (< 100 jobs/minute)
- Pool size: 16-32

**Medium Applications** (100-1000 jobs/minute)
- Pool size: 34-64

**Large Applications** (1000+ jobs/minute)
- Pool size: 128-256

**Very High Throughput** (10000+ jobs/minute)
- Pool size: 256-512

### Matching Concurrency

Ensure pool size is at least as large as scheduler max concurrency:

```csharp
options.ConfigureScheduler(scheduler =>
{
    scheduler.MaxConcurrency = 20;
});

options.AddOperationalStore(efOptions =>
{
    efOptions.SetDbContextPoolSize(50); // At least 2x MaxConcurrency
});
```

## Indexes

TickerQ automatically creates indexes for optimal query performance.

### Automatic Indexes

TickerQ creates the following indexes automatically:

**TimeTickerEntity:**
- `IX_TimeTicker_ExecutionTime` - On `ExecutionTime`
- `IX_TimeTicker_Status_ExecutionTime` - Composite on `Status`, `ExecutionTime`
- Indexes on foreign keys (`ParentId`)

**CronTickerEntity:**
- Indexes on unique constraint fields
- Foreign key indexes

**CronTickerOccurrenceEntity:**
- `IX_CronTickerOccurrence_CronTickerId` - On `CronTickerId`
- `IX_CronTickerOccurrence_ExecutionTime` - On `ExecutionTime`
- `IX_CronTickerOccurrence_Status` - On `Status`

### Custom Indexes

Add custom indexes for application-specific queries:

```csharp
public class CustomTimeTickerConfiguration : IEntityTypeConfiguration<CustomTimeTicker>
{
    public void Configure(EntityTypeBuilder<CustomTimeTicker> builder)
    {
        // Custom indexes for frequently queried properties
        builder.HasIndex(t => t.TenantId);
        builder.HasIndex(t => new { t.TenantId, t.Status });
        builder.HasIndex(t => t.Category);
    }
}
```

### Index Maintenance

Monitor index usage and performance:

```sql
-- PostgreSQL: Check index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE tablename LIKE '%ticker%';

-- SQL Server: Check index fragmentation
SELECT OBJECT_NAME(OBJECT_ID) AS TableName,
       name AS IndexName,
       avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED')
WHERE OBJECT_NAME(OBJECT_ID) LIKE '%Ticker%';
```

## Transactions

TickerQ operations are executed within database transactions for consistency.

### Automatic Transactions

TickerQ automatically manages transactions for:
- Job creation and updates
- Parent-child job operations
- Retry handling
- Status updates

### Transaction Isolation

Default isolation level is used (typically Read Committed). For custom isolation:

```csharp
using var transaction = await _context.Database.BeginTransactionAsync(
    IsolationLevel.Serializable);

try
{
    // Your operations
    await _context.SaveChangesAsync();
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

## Connection String Optimization

### Connection Pooling

Configure connection pool settings in connection string:

```csharp
// PostgreSQL
var connectionString = "Server=localhost;Port=5432;Database=TickerQ;User Id=postgres;Password=postgres;" +
    "Pooling=true;MinPoolSize=10;MaxPoolSize=100;";

// SQL Server
var connectionString = "Server=localhost;Database=TickerQ;Integrated Security=true;" +
    "Pooling=true;MinPoolSize=10;MaxPoolSize=100;";
```

### Connection Timeout

```csharp
var connectionString = "Server=localhost;Database=TickerQ;..." +
    "Connection Timeout=30;Command Timeout=60;";
```

### Retry Configuration

Enable automatic retry on transient failures:

```csharp
efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
{
    optionsBuilder.UseNpgsql(connectionString, cfg =>
    {
        cfg.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorCodesToAdd: ["40P01"] // PostgreSQL deadlock code
        );
    });
});
```

## Query Optimization

### Efficient Filtering

```csharp
// Good: Filter at database level
var jobs = await _context.Set<TimeTickerEntity>()
    .Where(t => t.Status == TickerStatus.Failed && t.ExecutedAt > cutoffDate)
    .ToListAsync();

// Avoid: Loading all then filtering in memory
var allJobs = await _context.Set<TimeTickerEntity>().ToListAsync();
var filtered = allJobs.Where(t => t.Status == TickerStatus.Failed);
```

### Projection

Select only needed fields:

```csharp
var jobSummary = await _context.Set<TimeTickerEntity>()
    .Where(t => t.Status == TickerStatus.Queued)
    .Select(t => new { t.Id, t.Function, t.ExecutionTime })
    .ToListAsync();
```

### Paging

Use paging for large result sets:

```csharp
var pageSize = 100;
var page = 1;

var jobs = await _context.Set<TimeTickerEntity>()
    .Where(t => t.Status == TickerStatus.Failed)
    .OrderByDescending(t => t.ExecutedAt)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

## Bulk Operations

Use bulk operations for efficiency:

```csharp
// Bulk delete
var oldOccurrences = await _context.Set<CronTickerOccurrenceEntity>()
    .Where(o => o.CreatedAt < cutoffDate)
    .ExecuteDeleteAsync(); // EF Core 7+

// Bulk update
await _context.Set<TimeTickerEntity>()
    .Where(t => t.Status == TickerStatus.Failed && t.ExecutedAt < cutoffDate)
    .ExecuteUpdateAsync(setter => setter.SetProperty(t => t.Status, TickerStatus.Cancelled));
```

## Monitoring Performance

### Query Performance

Enable query logging in development:

```csharp
optionsBuilder.UseNpgsql(connectionString)
    .LogTo(Console.WriteLine, LogLevel.Information)
    .EnableSensitiveDataLogging(); // Development only
```

### Connection Pool Monitoring

```csharp
// Get current pool statistics
var serviceProvider = builder.Services.BuildServiceProvider();
var dbContextFactory = serviceProvider.GetRequiredService<IDbContextFactory<TickerQDbContext>>();

// Monitor connection pool usage
// Check database connection pool statistics
```

## Production Tuning

### High-Throughput Configuration

```csharp
options.ConfigureScheduler(scheduler =>
{
    scheduler.MaxConcurrency = 20;
});

options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        var connectionString = "Server=localhost;Database=TickerQ;..." +
            "Pooling=true;MinPoolSize=20;MaxPoolSize=200;";
            
        optionsBuilder.UseNpgsql(connectionString, cfg =>
        {
            cfg.EnableRetryOnFailure(3, TimeSpan.FromSeconds(5), ["40P01"]);
            cfg.CommandTimeout(60);
        });
    });
    
    efOptions.SetDbContextPoolSize(100);
});
```

### Low-Resource Configuration

```csharp
options.ConfigureScheduler(scheduler =>
{
    scheduler.MaxConcurrency = 4;
});

options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        var connectionString = "Server=localhost;Database=TickerQ;..." +
            "Pooling=true;MinPoolSize=5;MaxPoolSize=20;";
            
        optionsBuilder.UseNpgsql(connectionString);
    });
    
    efOptions.SetDbContextPoolSize(16);
});
```

## See Also

- [Connection & Pooling](/api-reference/configuration/entity-framework-configuration/connection-pooling) - Detailed pooling configuration
- [Database Operations](./database-operations) - Querying and managing jobs
- [Best Practices](./best-practices) - Production recommendations
