# OpenTelemetry Integration

TickerQ.Instrumentation.OpenTelemetry provides distributed tracing and structured logging for TickerQ jobs, enabling comprehensive observability in distributed systems.

## Sections


### [Integration](./opentelemetry/integration)
Integrate with observability platforms, APM tools, and monitoring systems.

## Quick Start

```csharp
using TickerQ.DependencyInjection;
using TickerQ.Instrumentation.OpenTelemetry;
using OpenTelemetry.Trace;

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddConsoleExporter()
               .AddJaegerExporter();
    });

// Add TickerQ with OpenTelemetry instrumentation
builder.Services.AddTickerQ(options =>
{
    // Your TickerQ configuration
    options.AddOpenTelemetryInstrumentation(); // 👈 Enable tracing
});

var app = builder.Build();
app.UseTickerQ();
app.Run();
```

## Trace Structure

### Activity Hierarchy

TickerQ creates the following activity structure:

```
tickerq.job.execute.timeticker (main job execution span)
├── tickerq.job.enqueued (when job starts execution)
├── tickerq.job.completed (on successful completion)
├── tickerq.job.failed (on failure)
├── tickerq.job.cancelled (on cancellation)
├── tickerq.job.skipped (when skipped)
├── tickerq.seeding.started (for data seeding)
└── tickerq.seeding.completed (seeding completion)
```

### Activity Names

- `tickerq.job.execute.timeticker` - Main TimeTicker execution
- `tickerq.job.execute.crontickeroccurrence` - CronTicker occurrence execution
- `tickerq.job.enqueued` - Job enqueued event
- `tickerq.job.completed` - Job completed event
- `tickerq.job.failed` - Job failed event
- `tickerq.job.cancelled` - Job cancelled event
- `tickerq.job.skipped` - Job skipped event

## Activity Tags

TickerQ adds comprehensive tags to activities:

| Tag | Description | Example |
|-----|-------------|---------|
| `tickerq.job.id` | Unique job identifier | `123e4567-e89b-12d3-a456-426614174000` |
| `tickerq.job.type` | Type of ticker | `TimeTicker`, `CronTicker` |
| `tickerq.job.function` | Function name | `ProcessEmails` |
| `tickerq.job.priority` | Job priority | `Normal`, `High`, `LongRunning` |
| `tickerq.job.machine` | Machine executing job | `web-server-01` |
| `tickerq.job.parent_id` | Parent job ID | `parent-job-guid` |
| `tickerq.job.enqueued_from` | Where job was enqueued | `UserController.CreateUser (Program.cs:42)` |
| `tickerq.job.is_due` | Whether job is due | `true`, `false` |
| `tickerq.job.is_child` | Whether this is a child job | `true`, `false` |
| `tickerq.job.retries` | Maximum retry attempts | `3` |
| `tickerq.job.current_attempt` | Current retry attempt | `1`, `2`, `3` |
| `tickerq.job.final_status` | Final execution status | `Done`, `Failed`, `Cancelled` |
| `tickerq.job.final_retry_count` | Final retry count | `2` |
| `tickerq.job.execution_time_ms` | Execution time in milliseconds | `1250` |
| `tickerq.job.success` | Whether execution was successful | `true`, `false` |
| `tickerq.job.error_type` | Exception type | `SqlException`, `TimeoutException` |
| `tickerq.job.error_message` | Error message | `Connection timeout` |
| `tickerq.job.error_stack_trace` | Full stack trace | `at MyService.ProcessData()...` |
| `tickerq.job.cancellation_reason` | Reason for cancellation | `Task was cancelled` |
| `tickerq.job.skip_reason` | Reason for skipping | `Another instance is already running` |

## Integration Examples

### With Jaeger

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddJaegerExporter(options =>
               {
                   options.Endpoint = new Uri("http://localhost:14268/api/traces");
               });
    });

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation();
});
```

### With Azure Application Insights

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddAzureMonitorTraceExporter(options =>
               {
                   options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
               });
    });

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation();
});
```

### With Zipkin

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddZipkinExporter(options =>
               {
                   options.Endpoint = new Uri("http://localhost:9411/api/v2/spans");
               });
    });

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation();
});
```

### With OTLP (OpenTelemetry Protocol)

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddOtlpExporter(options =>
               {
                   options.Endpoint = new Uri("http://localhost:4317");
               });
    });

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation();
});
```

## Structured Logging

TickerQ OpenTelemetry integration provides structured logging through `ILogger`:

### Log Output Examples

```
[INF] TickerQ Job enqueued: TimeTicker - ProcessEmails (123e4567-e89b-12d3-a456-426614174000) from ExecutionTaskHandler
[INF] TickerQ Job completed: ProcessEmails (123e4567-e89b-12d3-a456-426614174000) in 1250ms - Success: True
[ERR] TickerQ Job failed: ProcessEmails (123e4567-e89b-12d3-a456-426614174000) - Retry 1 - Connection timeout
[INF] TickerQ Job completed: ProcessEmails (123e4567-e89b-12d3-a456-426614174000) in 2500ms - Success: False
[WRN] TickerQ Job cancelled: ProcessEmails (123e4567-e89b-12d3-a456-426614174000) - Task was cancelled
[INF] TickerQ Job skipped: ProcessEmails (123e4567-e89b-12d3-a456-426614174000) - Another CronOccurrence is already running!
[INF] TickerQ start seeding data: TimeTicker (production-node-01)
[INF] TickerQ completed seeding data: TimeTicker (production-node-01)
```

### Logging Frameworks

Works with any logging framework that integrates with `ILogger`:

#### Serilog

```csharp
builder.Host.UseSerilog((context, config) =>
{
    config.WriteTo.Console()
          .WriteTo.File("logs/tickerq-.txt", rollingInterval: RollingInterval.Day)
          .Enrich.FromLogContext();
});
```

#### NLog

```csharp
builder.Logging.ClearProviders();
builder.Logging.AddNLog();
```

#### Application Insights

```csharp
builder.Services.AddApplicationInsightsTelemetry();
```

## Parent-Child Relationships

TickerQ maintains trace relationships between parent and child jobs:

```
tickerq.job.execute.timeticker (parent)
└── tickerq.job.execute.timeticker (child)
    └── tickerq.job.execute.timeticker (grandchild)
```

Child job traces are linked to parent traces using the `tickerq.job.parent_id` tag.

## Performance Considerations

### Minimal Overhead

- Activities are only created when OpenTelemetry listeners are active
- Uses structured logging with minimal string allocations
- No performance impact when tracing is disabled

### Conditional Tracing

Tracer automatically handles cases where no listeners are registered:

```csharp
// No overhead if no listeners
using var activity = _instrumentation.StartJobActivity("my-job", context);
// Activity will be null if no listeners
```

## Filtering and Sampling

Configure sampling for TickerQ traces:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .SetSampler(new TraceIdRatioBasedSampler(0.1)) // Sample 10% of traces
               .AddJaegerExporter();
    });
```

Or filter specific jobs:

```csharp
tracing.AddSource("TickerQ")
       .AddProcessor(new SimpleActivityExportProcessor(new CustomExporter()))
       .AddJaegerExporter();
```

## Best Practices

### 1. Correlation with Application Traces

Ensure TickerQ traces are correlated with your application traces:

```csharp
// In your job function
using var activity = Activity.Current; // Get current activity
if (activity != null)
{
    activity.SetTag("custom.tag", "value");
}
```

### 2. Filter High-Volume Jobs

Consider sampling or filtering for high-frequency jobs:

```csharp
// Sample only 1% of high-frequency jobs
tracing.SetSampler(new TraceIdRatioBasedSampler(0.01));
```

### 3. Use Structured Logging

Leverage structured logging for better querying:

```csharp
_logger.LogInformation(
    "Job {JobId} completed in {ElapsedMs}ms with status {Status}",
    jobId, elapsedMs, status);
```

### 4. Monitor Trace Volume

Monitor trace volume in your observability platform to avoid overwhelming your tracing backend.

## Requirements

- .NET 8.0 or later
- OpenTelemetry 1.7.0 or later
- TickerQ.Utilities (automatically included)

## Next Steps

- [Learn About Dashboard](/features/dashboard)
- [Configure Entity Framework](/features/entity-framework)
- [Explore Redis Integration](/features/redis)

