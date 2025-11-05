# Best Practices

Recommended practices for production deployments of TickerQ with Entity Framework Core.

## Setup Best Practices

### Use Built-in TickerQDbContext

For most scenarios, prefer the built-in `TickerQDbContext`:

```csharp
options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        optionsBuilder.UseNpgsql(connectionString);
    });
});
```

**Benefits:**
- Simpler configuration
- Isolated from application entities
- Easier to manage separately
- Optimized for TickerQ workloads

### Use Application DbContext When Needed

Only use application DbContext when you need:
- Shared connection pooling
- Entity joins across application and TickerQ entities
- Unified migration management

### Use Model Customizer

Always use `ConfigurationType.UseModelCustomizer` unless you have conflicts:

```csharp
efOptions.UseApplicationDbContext<MyApplicationDbContext>(
    ConfigurationType.UseModelCustomizer);
```

## Migration Best Practices

### Test Migrations First

Always test migrations in development before production:

```bash
# Development
dotnet ef migrations add TestMigration --context TickerQDbContext
dotnet ef database update --context TickerQDbContext

# Verify changes
# Then apply to production
```

### Version Control Migrations

Commit all migration files to version control:

```bash
git add Migrations/
git commit -m "Add TickerQ initial schema"
```

### Backup Before Migration

Always backup production database before applying migrations:

```bash
# PostgreSQL
pg_dump -h localhost -U postgres TickerQ > backup.sql

# SQL Server
sqlcmd -S localhost -d TickerQ -Q "BACKUP DATABASE TickerQ TO DISK='backup.bak'"
```

### Incremental Migrations

Create separate migrations for each schema change:

```bash
# Good: Separate migrations
dotnet ef migrations add AddCustomProperty --context TickerQDbContext
dotnet ef migrations add AddIndex --context TickerQDbContext

# Avoid: Combining multiple changes in one migration
```

## Performance Best Practices

### Configure Appropriate Pool Size

Match pool size to your workload:

```csharp
// High throughput
efOptions.SetDbContextPoolSize(100);

// Medium workload
efOptions.SetDbContextPoolSize(34);

// Low workload
efOptions.SetDbContextPoolSize(16);
```

### Enable Connection Retry

Always enable retry on transient failures:

```csharp
optionsBuilder.UseNpgsql(connectionString, cfg =>
{
    cfg.EnableRetryOnFailure(3, TimeSpan.FromSeconds(5), ["40P01"]);
});
```

### Monitor Database Performance

Regularly monitor:
- Connection pool usage
- Query performance
- Index usage
- Table sizes

## Data Management Best Practices

### Regular Cleanup

Schedule cleanup jobs to remove old data:

```csharp
[TickerFunction("CleanupOldJobs", "0 0 2 * * *")] // Daily at 2 AM
public async Task CleanupOldJobs(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    var cutoffDate = DateTime.UtcNow.AddDays(-90);
    
    await _context.Set<CronTickerOccurrenceEntity>()
        .Where(o => o.CreatedAt < cutoffDate && o.Status != TickerStatus.InProgress)
        .ExecuteDeleteAsync(cancellationToken);
    
    await _context.Set<TimeTickerEntity>()
        .Where(t => (t.Status == TickerStatus.Done || t.Status == TickerStatus.DueDone)
                    && t.ExecutedAt < cutoffDate)
        .ExecuteDeleteAsync(cancellationToken);
}
```

### Monitor Table Sizes

Track table growth and plan cleanup:

```sql
-- PostgreSQL
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE tablename LIKE '%ticker%'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Archive Old Data

Instead of deleting, consider archiving:

```csharp
public async Task ArchiveOldJobsAsync(DateTime olderThan)
{
    var oldJobs = await _context.Set<TimeTickerEntity>()
        .Where(t => t.ExecutedAt < olderThan && t.Status == TickerStatus.Done)
        .ToListAsync();
    
    // Archive to separate table or file
    await ArchiveJobsAsync(oldJobs);
    
    // Then delete
    _context.Set<TimeTickerEntity>().RemoveRange(oldJobs);
    await _context.SaveChangesAsync();
}
```

## Security Best Practices

### Secure Connection Strings

Never hardcode connection strings. Use:

```csharp
// From configuration
var connectionString = builder.Configuration.GetConnectionString("TickerQ");

// From environment variables
var connectionString = Environment.GetEnvironmentVariable("TICKERQ_CONNECTION_STRING");

// From Azure Key Vault, AWS Secrets Manager, etc.
var connectionString = await _keyVault.GetSecretAsync("TickerQ-ConnectionString");
```

### Use Parameterized Queries

Always use Entity Framework Core queries (automatically parameterized):

```csharp
// Good: EF Core parameterizes automatically
var jobs = await _context.Set<TimeTickerEntity>()
    .Where(t => t.Function == functionName)
    .ToListAsync();

// Avoid: Raw SQL without parameters
var jobs = _context.Set<TimeTickerEntity>()
    .FromSqlRaw($"SELECT * FROM TimeTickers WHERE Function = '{functionName}'");
```

## Monitoring Best Practices

### Enable Query Logging (Development)

```csharp
optionsBuilder.UseNpgsql(connectionString)
    .LogTo(Console.WriteLine, LogLevel.Information);
```

### Use Application Insights

Integrate with monitoring platforms:

```csharp
// Azure Application Insights
builder.Services.AddApplicationInsightsTelemetry();

// Custom metrics
_telemetryClient.TrackMetric("TickerQ.Jobs.Processed", jobCount);
```

### Health Checks

Add health checks for database connectivity:

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<TickerQDbContext>("tickerq-db");

app.MapHealthChecks("/health/tickerq");
```

## Error Handling Best Practices

### Handle Transient Failures

```csharp
try
{
    var job = await _context.Set<TimeTickerEntity>().FindAsync(jobId);
    // Operations
    await _context.SaveChangesAsync();
}
catch (DbUpdateException ex) when (ex.InnerException is PostgresException pgEx && pgEx.SqlState == "40P01")
{
    // Retry on deadlock
    await Task.Delay(TimeSpan.FromMilliseconds(100));
    // Retry operation
}
```

### Logging Database Operations

```csharp
public class JobRepository
{
    private readonly ILogger<JobRepository> _logger;
    private readonly MyDbContext _context;
    
    public async Task<List<TimeTickerEntity>> GetFailedJobsAsync()
    {
        try
        {
            return await _context.Set<TimeTickerEntity>()
                .Where(t => t.Status == TickerStatus.Failed)
                .ToListAsync();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to retrieve failed jobs");
            throw;
        }
    }
}
```

## Configuration Best Practices

### Environment-Specific Settings

```csharp
var poolSize = builder.Environment.IsProduction() ? 100 : 16;
var connectionString = builder.Configuration.GetConnectionString("TickerQ");

options.AddOperationalStore(efOptions =>
{
    efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
    {
        optionsBuilder.UseNpgsql(connectionString);
    });
    efOptions.SetDbContextPoolSize(poolSize);
});
```

### Configuration Validation

Validate configuration at startup:

```csharp
var connectionString = builder.Configuration.GetConnectionString("TickerQ");
if (string.IsNullOrEmpty(connectionString))
{
    throw new InvalidOperationException("TickerQ connection string is required");
}

// Test connection
using var testContext = new TickerQDbContext(
    new DbContextOptionsBuilder<TickerQDbContext>()
        .UseNpgsql(connectionString)
        .Options);
        
await testContext.Database.CanConnectAsync();
```

## Migration from In-Memory

If you start with in-memory mode and later need persistence:

1. **Install Package**
   ```bash
   dotnet add package TickerQ.EntityFrameworkCore
   ```

2. **Add Configuration**
   ```csharp
   options.AddOperationalStore(efOptions =>
   {
       efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
       {
           optionsBuilder.UseNpgsql(connectionString);
       });
   });
   ```

3. **Create Migrations**
   ```bash
   dotnet ef migrations add InitialCreate --context TickerQDbContext
   dotnet ef database update --context TickerQDbContext
   ```

4. **Note**: Existing in-memory jobs will be lost (expected behavior)

## Troubleshooting

### High Connection Usage

- Reduce `MaxConcurrency` in scheduler
- Increase database connection pool limits
- Review and optimize queries

### Slow Queries

- Check index usage
- Review query execution plans
- Consider adding custom indexes

### Migration Failures

- Backup database before migration
- Test migrations in development
- Review generated SQL before applying

## See Also

- [Installation](./installation) - Package installation
- [Setup Guide](./setup/index) - Configuration options
- [Migrations](./migrations) - Database schema management
- [Database Operations](./database-operations) - Querying and updating
- [Performance Guide](./performance) - Optimization techniques

