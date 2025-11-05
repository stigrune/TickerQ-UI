# Job Types

TickerQ supports two main job types: **TimeTicker** and **CronTicker**. Understanding the differences helps you choose the right type for your use case.

## TimeTicker

TimeTicker is for jobs that run at a specific date and time. It's perfect for one-time tasks or jobs that need precise timing.

### Characteristics

- Executes once at a specified `ExecutionTime`
- Can be scheduled in the past (will execute immediately if due)
- Supports child jobs with conditional execution
- Can be cancelled, updated, or deleted
- Tracks execution status, elapsed time, and retry counts

### Use Cases

- Send email 5 minutes after user registration
- Process order 1 hour after purchase
- Cleanup job scheduled for next maintenance window
- Delayed notifications
- One-time batch processing

### Example

```csharp
await _timeTickerManager.AddAsync(new TimeTickerEntity
{
    Function = "SendWelcomeEmail",
    ExecutionTime = DateTime.UtcNow.AddMinutes(5),
    Request = TickerHelper.CreateTickerRequest(new EmailRequest 
    { 
        To = user.Email 
    }),
    Retries = 3,
    RetryIntervals = new[] { 60, 300, 900 }
});
```

### TimeTicker Properties

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Unique identifier |
| `Function` | `string` | Function name (must match `[TickerFunction]` attribute) |
| `ExecutionTime` | `DateTime?` | When to execute the job |
| `Request` | `byte[]` | Serialized request data (use `TickerHelper.CreateTickerRequest`) |
| `Description` | `string` | Optional description |
| `Retries` | `int` | Maximum number of retry attempts |
| `RetryIntervals` | `int[]` | Retry intervals in seconds |
| `ParentId` | `Guid?` | Parent job ID for child jobs |
| `RunCondition` | `RunCondition?` | Condition for child job execution |

### Status Tracking

TimeTicker tracks detailed execution information:

```csharp
// Properties available after execution
ticker.Status          // Current status (Idle, Queued, InProgress, Done, Failed, etc.)
ticker.ExecutedAt      // When the job was executed
ticker.ElapsedTime    // Execution time in milliseconds
ticker.RetryCount      // Current retry attempt
ticker.ExceptionMessage // Exception details if failed
ticker.LockedAt        // When the job was locked for execution
ticker.LockHolder      // Node/machine that locked the job
```

## CronTicker

CronTicker is for recurring jobs using cron expressions. It's ideal for periodic tasks that need to run on a schedule.

### Characteristics

- Executes repeatedly based on cron expression
- Requires 6-part cron format (with seconds)
- Automatically creates occurrences for tracking
- Can skip if previous occurrence is still running
- Persistent configuration - define once, runs forever

### Use Cases

- Daily reports at midnight
- Hourly data synchronization
- Weekly backups
- Periodic health checks
- Scheduled maintenance tasks

### Cron Expression Format

TickerQ uses 6-part cron expressions (with seconds) as the required format:

```
second minute hour day month day-of-week
```

**Examples:**
- `"0 0 0 * * *"` - Daily at midnight (00:00:00)
- `"0 0 9 * * *"` - Daily at 9:00 AM
- `"30 0 0 * * *"` - Daily at 00:00:30 (30 seconds past midnight)
- `"0 0 */6 * * *"` - Every 6 hours on the hour
- `"*/10 * * * * *"` - Every 10 seconds
- `"0 */5 * * * *"` - Every 5 minutes
- `"0 0 9,17 * * *"` - At 9 AM and 5 PM daily
- `"0 30 14 * * 1"` - Every Monday at 2:30 PM

> **Note:** All cron expressions must include the seconds field (6 parts). The format is: `second minute hour day month day-of-week`.

### Example

```csharp
await _cronTickerManager.AddAsync(new CronTickerEntity
{
    Function = "GenerateDailyReport",
    Expression = "0 0 0 * * *", // Daily at midnight (6-part format with seconds)
    Request = TickerHelper.CreateTickerRequest(new ReportRequest 
    { 
        ReportType = "Daily" 
    }),
    Retries = 2,
    RetryIntervals = new[] { 300, 900 }
});
```

### Defining Cron Jobs with Attributes

You can also define cron jobs directly in the function:

```csharp
[TickerFunction("GenerateDailyReport", cronExpression: "0 0 0 * * *")]
public async Task GenerateDailyReport(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    // This will be automatically registered and scheduled
    // Note: cron expression must be 6-part format with seconds
    await GenerateReportAsync(cancellationToken);
}
```

### CronTicker Properties

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Unique identifier |
| `Function` | `string` | Function name (must match `[TickerFunction]` attribute) |
| `Expression` | `string` | Cron expression (6-part with seconds, required) |
| `Request` | `byte[]` | Serialized request data |
| `Description` | `string` | Optional description |
| `Retries` | `int` | Maximum number of retry attempts |
| `RetryIntervals` | `int[]` | Retry intervals in seconds |

### CronTicker Occurrences

When a CronTicker runs, it creates a `CronTickerOccurrence` that tracks individual executions:

```csharp
// Each execution creates an occurrence with:
occurrence.Id              // Unique occurrence ID
occurrence.ExecutionTime  // Scheduled execution time
occurrence.Status         // Status of this occurrence
occurrence.ExecutedAt     // Actual execution time
occurrence.ElapsedTime    // Execution duration
occurrence.RetryCount     // Retry attempts
```

### Preventing Concurrent Execution

To prevent multiple occurrences from running simultaneously:

```csharp
[TickerFunction("LongRunningTask", cronExpression: "0 0 * * * *")]
public async Task LongRunningTask(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    // Skip if another occurrence is already running
    context.CronOccurrenceOperations.SkipIfAlreadyRunning();
    
    // Your long-running task
    await ProcessDataAsync(cancellationToken);
}
```

## Comparing TimeTicker vs CronTicker

| Feature | TimeTicker | CronTicker |
|---------|------------|------------|
| **Execution Type** | One-time | Recurring |
| **Scheduling** | Specific DateTime | Cron expression |
| **Child Jobs** | Supported | Not supported |
| **Occurrences** | Single execution | Multiple occurrences |
| **Use Case** | Delayed tasks | Periodic tasks |
| **Update** | Can update ExecutionTime | Can update Expression |
| **Past Scheduling** | Executes immediately if due | Always scheduled for future |

## Choosing the Right Type

**Use TimeTicker when:**
- Job runs once at a specific time
- You need conditional child jobs
- Job is triggered by a user action (e.g., send email after registration)
- Timing is dynamic based on business logic

**Use CronTicker when:**
- Job needs to run on a recurring schedule
- Schedule follows a pattern (daily, weekly, hourly)
- Job should run indefinitely
- Schedule is defined at development time

## Next Steps

- [Learn About Job Scheduling](/concepts/job-fundamentals)
- [Configure Retry Policies](/concepts/error-handling)
- [Explore Child Jobs](/concepts/job-fundamentals#job-chaining)
