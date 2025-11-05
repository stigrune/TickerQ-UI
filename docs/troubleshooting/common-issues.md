# Common Issues and Troubleshooting

This guide helps you resolve common issues when using TickerQ.

## Installation Issues

### Package Version Mismatch

**Problem:** Packages show version conflicts or missing dependencies.

**Solution:**
Ensure all TickerQ packages are the same version:

```bash
dotnet add package TickerQ --version 8.0.0
dotnet add package TickerQ.EntityFrameworkCore --version 8.0.0
dotnet add package TickerQ.Dashboard --version 8.0.0
```

### Source Generator Not Working

**Problem:** `[TickerFunction]` methods not being recognized.

**Solution:**
1. Ensure you're using .NET 8.0 or later
2. Restart your IDE (Visual Studio/Rider/VS Code)
3. Clean and rebuild the solution:
   ```bash
   dotnet clean
   dotnet build
   ```
4. Verify source generator DLL is included in the package

## Configuration Issues

### Jobs Not Executing

**Problem:** Jobs are scheduled but never execute.

**Solution:**
1. Verify `UseTickerQ()` is called in your application pipeline:
   ```csharp
   app.UseTickerQ();
   ```
2. Check start mode:
   ```csharp
   app.UseTickerQ(TickerQStartMode.Immediate); // Not Manual
   ```
3. Verify job execution time is not in the future
4. Check application logs for errors
5. Ensure persistence provider is configured correctly

### Function Not Found

**Problem:** Error indicating function name not found.

**Solution:**
1. Ensure function name matches exactly:
   ```csharp
   [TickerFunction("MyFunction")] // Must match exactly
   public async Task MyFunction(...) { }
   
   // When scheduling
   Function = "MyFunction" // Same name
   ```
2. Verify source generator ran successfully (check build output)
3. Rebuild the project to regenerate function registry

### Entity Framework Configuration Issues

**Problem:** Migrations fail or entities not configured.

**Solution:**
1. If using built-in `TickerQDbContext` (recommended), migrations are straightforward:
   ```csharp
   options.AddOperationalStore(efOptions =>
   {
       efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
       {
           optionsBuilder.UseNpgsql(connectionString);
       });
   });
   
   // Then run migrations
   dotnet ef migrations add InitialCreate --context TickerQDbContext
   dotnet ef database update --context TickerQDbContext
   ```
2. If using application DbContext, use `UseApplicationDbContext`:
   ```csharp
   options.AddOperationalStore<MyApplicationDbContext>(efOptions =>
   {
       efOptions.UseApplicationDbContext<MyApplicationDbContext>(ConfigurationType.UseModelCustomizer);
   });
   ```
3. If conflicts with other `IModelCustomizer`, use manual configuration:
   ```csharp
   protected override void OnModelCreating(ModelBuilder builder)
   {
       base.OnModelCreating(builder);
       builder.ApplyConfiguration(new TimeTickerConfigurations());
       builder.ApplyConfiguration(new CronTickerConfigurations());
       builder.ApplyConfiguration(new CronTickerOccurrenceConfigurations());
   }
   ```

## Runtime Issues

### Jobs Stuck in InProgress

**Problem:** Jobs remain in `InProgress` status and never complete.

**Solution:**
1. Check for unhandled exceptions in job functions
2. Verify `CancellationToken` is being checked
3. Check if job is actually running (dashboard or logs)
4. Restart the application to release stuck jobs
5. With EF Core, manually update status if needed:
   ```csharp
   var job = await _context.Set<TimeTickerEntity>().FindAsync(jobId);
   if (job.Status == TickerStatus.InProgress 
       && job.LockedAt < DateTime.UtcNow.AddMinutes(-30))
   {
       job.Status = TickerStatus.Failed;
       job.ExceptionMessage = "Job timeout";
       await _context.SaveChangesAsync();
   }
   ```

### High Memory Usage

**Problem:** Application memory usage is high.

**Solution:**
1. Check for memory leaks in job functions
2. Ensure proper disposal of resources:
   ```csharp
   await using var scope = _serviceProvider.CreateAsyncScope();
   // Use scope.ServiceProvider
   ```
3. Configure concurrency limits:
   ```csharp
   options.ConfigureScheduler(scheduler =>
   {
       scheduler.MaxConcurrency = Environment.ProcessorCount;
   });
   ```
4. Clean up old jobs and occurrences regularly
5. Check for accumulating request data

### Database Connection Issues

**Problem:** Database connection errors with EF Core.

**Solution:**
1. Verify connection string is correct
2. Check database is accessible
3. Ensure connection pooling is configured (with built-in `TickerQDbContext`):
   ```csharp
   options.AddOperationalStore(efOptions =>
   {
       efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
       {
           optionsBuilder.UseNpgsql(connectionString, cfg =>
           {
               cfg.EnableRetryOnFailure(3, TimeSpan.FromSeconds(5), ["40P01"]);
           });
       });
       efOptions.SetDbContextPoolSize(34); // Adjust as needed
   });
   ```
4. Check database timeout settings
5. Monitor connection pool exhaustion
6. Enable retry on failure for transient errors (as shown above)

## Dashboard Issues

### Dashboard Not Loading

**Problem:** Dashboard UI doesn't load or shows errors.

**Solution:**
1. Verify `UseTickerQ()` is called
2. Check base path configuration:
   ```csharp
   dashboardOptions.SetBasePath("/tickerq/dashboard");
   ```
3. Ensure static files middleware is configured
4. Check browser console for errors
5. Verify authentication is configured correctly

### 403 Forbidden Errors

**Problem:** Random 403 responses from dashboard endpoints.

**Solution:**
1. Check authentication configuration
2. Verify no authorization policies are blocking requests
3. Check middleware ordering (dashboard should be after auth)
4. Verify CORS configuration if dashboard is on different domain
5. Check route constraints

### SignalR Connection Issues

**Problem:** Real-time updates not working.

**Solution:**
1. Verify SignalR is configured:
   ```csharp
   builder.Services.AddSignalR();
   ```
2. Check WebSocket support is enabled
3. Verify authentication for SignalR connections
4. Check firewall/proxy settings
5. Review browser console for connection errors

## Redis Issues

### Redis Connection Failures

**Problem:** Cannot connect to Redis.

**Solution:**
1. Verify Redis connection string:
   ```csharp
   redisOptions.Configuration = "localhost:6379";
   ```
2. Check Redis server is running
3. Verify network connectivity
4. Check authentication if required
5. Test connection with Redis CLI

### Dead Node Detection Not Working

**Problem:** Dead nodes are not detected.

**Solution:**
1. Verify heartbeat interval:
   ```csharp
   redisOptions.NodeHeartbeatInterval = TimeSpan.FromMinutes(1);
   ```
2. Check heartbeat TTL (interval + 20 seconds)
3. Verify `NodeHeartBeatBackgroundService` is running
4. Check Redis connectivity
5. Review application logs for heartbeat errors

## Performance Issues

### Slow Job Execution

**Problem:** Jobs take too long to execute.

**Solution:**
1. Profile job functions to identify bottlenecks
2. Check database query performance
3. Verify concurrency settings:
   ```csharp
   options.ConfigureScheduler(scheduler =>
   {
       scheduler.MaxConcurrency = Environment.ProcessorCount * 2;
   });
   ```
4. Use appropriate priority:
   ```csharp
   [TickerFunction("FastJob", taskPriority: TickerTaskPriority.High)]
   ```
5. Consider using `LongRunning` priority for CPU-bound tasks

### Too Many Concurrent Jobs

**Problem:** Too many jobs executing simultaneously.

**Solution:**
1. Reduce max concurrency:
   ```csharp
   options.ConfigureScheduler(scheduler =>
   {
       scheduler.MaxConcurrency = 5; // Limit to 5 concurrent jobs
   });
   ```
2. Use priority system to control execution order
3. Implement throttling in job functions
4. Schedule jobs with delays to spread load

## OpenTelemetry Issues

### Traces Not Appearing

**Problem:** OpenTelemetry traces are not being exported.

**Solution:**
1. Verify OpenTelemetry is configured:
   ```csharp
   builder.Services.AddOpenTelemetry()
       .WithTracing(tracing =>
       {
           tracing.AddSource("TickerQ");
       });
   ```
2. Ensure exporter is added:
   ```csharp
   tracing.AddConsoleExporter(); // Or other exporter
   ```
3. Verify instrumentation is enabled:
   ```csharp
   builder.Services.AddTickerQ(options =>
   {
       options.AddOpenTelemetryInstrumentation();
   });
   ```
4. Check exporter endpoint/configuration
5. Review application logs for exporter errors

## Best Practices for Troubleshooting

### 1. Enable Logging

```csharp
builder.Host.ConfigureLogging(logging =>
{
    logging.SetMinimumLevel(LogLevel.Debug);
    logging.AddConsole();
});
```

### 2. Use Dashboard

The dashboard provides real-time visibility into:
- Job status
- Active threads
- Next occurrences
- Error messages

### 3. Check Application Logs

Monitor logs for:
- Exception messages
- Job execution times
- Retry attempts
- Connection errors

### 4. Verify Configuration

Double-check:
- Package versions match
- Configuration options are correct
- Middleware ordering
- Connection strings

### 5. Test Incrementally

Start with minimal configuration and add features incrementally:
1. Start with core TickerQ (in-memory)
2. Add Entity Framework
3. Add Dashboard
4. Add Redis
5. Add OpenTelemetry

## Debug Checklist

Use this checklist to systematically debug issues:

### Installation & Setup
- [ ] TickerQ package installed correctly
- [ ] `.NET 8.0+` target framework
- [ ] `AddTickerQ()` called in service registration
- [ ] `UseTickerQ()` called in application pipeline
- [ ] No conflicting package versions

### Job Definition
- [ ] `[TickerFunction]` attribute present
- [ ] Function is `public` method
- [ ] Returns `Task` or `Task<T>`
- [ ] Signature matches: `(TickerFunctionContext, CancellationToken)`
- [ ] Function name matches exactly (case-sensitive)

### Job Scheduling
- [ ] Entity created with valid properties
- [ ] `Function` name matches attribute
- [ ] `ExecutionTime` set (for TimeTicker)
- [ ] Valid cron expression (for CronTicker - 6-part format)
- [ ] Manager `AddAsync` called
- [ ] Result checked for `IsSucceeded`

### Execution
- [ ] Job execution time has passed (if scheduled in future)
- [ ] Application is running (not just built)
- [ ] No exceptions in application logs
- [ ] Job status checked (via dashboard or persistence provider)

### Persistence (if using EF Core)
- [ ] Database connection working
- [ ] Migrations applied
- [ ] Entity configurations applied
- [ ] DbContext pool configured correctly

## Diagnostic Commands

### Check Registered Functions

```csharp
// After UseTickerQ() is called
var functions = TickerFunctionProvider.TickerFunctions;
foreach (var func in functions)
{
    Console.WriteLine($"Registered: {func.Key}");
}
```

### Check Job Status

```csharp
// If using EF Core
var job = await _context.Set<TimeTickerEntity>()
    .FirstOrDefaultAsync(t => t.Id == jobId);

Console.WriteLine($"Status: {job?.Status}");
Console.WriteLine($"Execution Time: {job?.ExecutionTime}");
Console.WriteLine($"Executed At: {job?.ExecutedAt}");
```

### Enable Verbose Logging

```csharp
builder.Host.ConfigureLogging(logging =>
{
    logging.SetMinimumLevel(LogLevel.Debug);
    logging.AddConsole();
});
```

## FAQ

### Q: How do I know if TickerQ is running?

A: Check that:
1. No exceptions during `UseTickerQ()` call
2. Background services started (check logs)
3. Dashboard accessible (if enabled)
4. Jobs execute at scheduled times

### Q: Can I schedule jobs before `UseTickerQ()` is called?

A: Yes, but they won't execute until `UseTickerQ()` is called. Jobs scheduled in the past will execute immediately.

### Q: How do I debug why a job didn't execute?

A:
1. Check job status via persistence provider or dashboard
2. Verify execution time has passed
3. Check application logs for errors
4. Verify function name matches exactly
5. Check if job was cancelled or skipped

### Q: What happens if I schedule a job with execution time in the past?

A: The job executes immediately (if due) when the scheduler processes it.

### Q: Can I update a job that's already running?

A: No. Only jobs with `Idle` status can be updated. Jobs `InProgress` cannot be modified.

### Q: How do I find all jobs with a specific status?

A: Use the persistence provider (EF Core) to query:

```csharp
var failedJobs = await _context.Set<TimeTickerEntity>()
    .Where(t => t.Status == TickerStatus.Failed)
    .ToListAsync();
```

### Q: My cron job isn't creating occurrences. Why?

A: Check:
1. Cron expression is valid 6-part format
2. Expression is parseable (no syntax errors)
3. Function name matches `[TickerFunction]` attribute
4. Next occurrence is calculated correctly (check logs)

## Getting Help

If you're still experiencing issues:

1. **Check Documentation**: Review relevant sections
2. **Search Issues**: Check GitHub issues for similar problems
3. **Create Minimal Reproduction**: Create a minimal example that reproduces the issue
4. **Provide Details**: Include:
   - TickerQ version
   - .NET version
   - Error messages with full stack traces
   - Complete configuration code
   - Steps to reproduce
   - Application logs

## Next Steps

- [Review Installation Guide](/getting-started/installation)
- [Check Job Configuration](/getting-started/quick-start)
- [Explore Dashboard](/features/dashboard)
- [Read Debugging Guide](/troubleshooting/debugging)

