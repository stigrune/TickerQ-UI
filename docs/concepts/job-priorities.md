# Job Priorities

TickerQ lets you tag jobs with a `TickerTaskPriority` via the `[TickerFunction]` attribute. Priority is surfaced in the dashboard and metrics, and the scheduler treats **long-running** work specially.

## Priority Levels

```csharp
public enum TickerTaskPriority
{
    LongRunning,  // Executes on a separate long-running path
    High,         // High priority (metadata today)
    Normal,       // Default priority
    Low           // Lowest priority
}
```

Today, only `LongRunning` has special scheduling behavior. `High`, `Normal`, and `Low` all share the same worker queues; they are primarily used for:

- Observability in the dashboard and logs
- Filtering/analytics
- Future scheduling strategies

## Setting Priority

### In Function Attribute

```csharp
[TickerFunction("CriticalTask", taskPriority: TickerTaskPriority.High)]
public async Task CriticalTask(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    // Tagged as High priority for observability
    await DoWorkAsync(cancellationToken);
}
```

### Long-Running Work

Use `LongRunning` for CPU-bound or very long-lived tasks:

```csharp
[TickerFunction("RebuildSearchIndex", taskPriority: TickerTaskPriority.LongRunning)]
public async Task RebuildSearchIndex(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    // This job runs on a separate long-running path
    await RebuildIndexAsync(cancellationToken);
}
```

Under the hood, `LongRunning` jobs are dispatched via `TaskCreationOptions.LongRunning` and bypass the regular worker queues (`TickerQTaskScheduler.QueueAsync`), so they do not block short-lived jobs.

## When to Use Each Priority

- `LongRunning` – heavy CPU work, long IO-bound pipelines, index rebuilds.
- `High` – important business jobs you want to highlight in monitoring.
- `Normal` – default for most application jobs.
- `Low` – background/low-importance jobs you may deprioritize in dashboards.

## Notes

- Priority is defined on the function via `[TickerFunction]`; scheduled TimeTickers inherit the priority from their handler.
- Changing priority on the attribute affects future scheduling but does not retroactively change existing persisted jobs.

