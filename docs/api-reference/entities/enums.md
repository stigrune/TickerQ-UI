# Enums

Reference for all TickerQ enumeration types.

## TickerStatus

Job execution status values.

```csharp
public enum TickerStatus
{
    Idle,           // Created but not yet queued
    Queued,         // In queue waiting to execute
    InProgress,     // Currently executing
    Done,           // Completed (executed after due time)
    DueDone,        // Completed (executed at due time)
    Failed,         // Failed after all retries
    Cancelled,      // Cancelled by user/system
    Skipped         // Skipped (e.g., duplicate prevented)
}
```

### Status Flow

```
Idle → Queued → InProgress → Done/Failed/Cancelled/Skipped
```

### Status Descriptions

| Status | Description | Can Update? | Can Delete? |
|--------|-------------|-------------|------------|
| `Idle` | Created but not yet queued | Yes | Yes |
| `Queued` | Waiting in queue to execute | No | Yes |
| `InProgress` | Currently executing | No | No |
| `Done` | Completed (executed after due time) | No | Yes |
| `DueDone` | Completed (executed exactly at due time) | No | Yes |
| `Failed` | Failed after all retries exhausted | No | Yes |
| `Cancelled` | Cancelled by user or system | No | Yes |
| `Skipped` | Skipped due to conditions (duplicate, etc.) | No | Yes |

## RunCondition

Condition for child job execution based on parent status.

```csharp
public enum RunCondition
{
    OnSuccess,              // Run if parent succeeds
    OnFailure,              // Run if parent fails
    OnCancelled,            // Run if parent is cancelled
    OnFailureOrCancelled,   // Run if parent fails or cancelled
    OnAnyCompletedStatus,   // Run after parent completes (any terminal status)
    InProgress              // Run in parallel with parent
}
```

### Usage Examples

```csharp
// Run child only if parent succeeds
var child = new TimeTickerEntity
{
    Function = "SendConfirmation",
    ParentId = parent.Id,
    RunCondition = RunCondition.OnSuccess,
    ExecutionTime = DateTime.UtcNow.AddMinutes(1)
};

// Run child in parallel with parent
var parallelChild = new TimeTickerEntity
{
    Function = "LogActivity",
    ParentId = parent.Id,
    RunCondition = RunCondition.InProgress,
    ExecutionTime = DateTime.UtcNow
};

// Run child if parent fails (error recovery)
var recoveryChild = new TimeTickerEntity
{
    Function = "NotifyFailure",
    ParentId = parent.Id,
    RunCondition = RunCondition.OnFailure,
    ExecutionTime = DateTime.UtcNow.AddMinutes(1)
};
```

### Condition Behavior

| Condition | When Child Runs | Parent Status |
|-----------|----------------|---------------|
| `OnSuccess` | Parent completes successfully | `DueDone` or `Done` |
| `OnFailure` | Parent fails after all retries | `Failed` |
| `OnCancelled` | Parent is cancelled | `Cancelled` |
| `OnFailureOrCancelled` | Parent fails or is cancelled | `Failed` or `Cancelled` |
| `OnAnyCompletedStatus` | Parent reaches any terminal status | `Done`, `DueDone`, `Failed`, `Cancelled`, `Skipped` |
| `InProgress` | Runs in parallel with parent | Any (runs immediately) |

## TickerTaskPriority

Execution priority for jobs.

```csharp
public enum TickerTaskPriority
{
    LongRunning,    // Executes in separate thread pool
    High,           // Highest priority
    Normal,         // Default priority
    Low             // Lowest priority
}
```

### Priority Usage

Set priority in the `[TickerFunction]` attribute:

```csharp
[TickerFunction(
    "MyJob",
    taskPriority: TickerTaskPriority.High
)]
public async Task MyJob(TickerFunctionContext context, CancellationToken cancellationToken)
{
    // High priority job
}
```

### Priority Behavior

- `High`: Executed before `Normal` and `Low` priority jobs
- `Normal`: Default priority (most common)
- `Low`: Executed after higher priority jobs
- `LongRunning`: Executes in separate thread pool (doesn't block other jobs)

## See Also

- [TimeTickerEntity](./time-ticker-entity) - Uses TickerStatus and RunCondition
- [CronTickerOccurrenceEntity](./cron-occurrence-entity) - Uses TickerStatus
- [Job Fundamentals](../../concepts/job-fundamentals) - Understanding job relationships and priorities

