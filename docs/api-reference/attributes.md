# Attributes

TickerQ uses attributes to discover job handlers and control constructor selection for dependency injection.

## [TickerFunction]

Namespace: `TickerQ.Utilities.Base`

Marks a method as a TickerQ job handler.

**Usage:**

```csharp
[TickerFunction("FunctionName")]
public async Task FunctionName(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    // Job logic
}
```

**Overloads:**

```csharp
public TickerFunctionAttribute(
    string functionName,
    string cronExpression = null,
    TickerTaskPriority taskPriority = TickerTaskPriority.Normal);

public TickerFunctionAttribute(
    string functionName,
    TickerTaskPriority taskPriority);
```

- `functionName` – logical identifier used when scheduling jobs (`Function` property on ticker entities).
- `cronExpression` – optional 6-part cron expression for recurring jobs.
- `taskPriority` – optional `TickerTaskPriority` (see [Job Priorities](/concepts/job-priorities)).

> **Note:** The source generator enforces that methods with `[TickerFunction]` have valid signatures and accessibility. See [Diagnostics](./diagnostics) for details.

## [TickerQConstructor]

Namespace: `TickerQ.Utilities.Base`

Marks the preferred constructor to use for dependency injection when a class with `[TickerFunction]` methods has multiple constructors.

**Usage:**

```csharp
using TickerQ.Utilities.Base;

public class ReportJobs
{
    private readonly IReportService _reportService;
    private readonly ILogger<ReportJobs> _logger;

    public ReportJobs(IReportService reportService)
    {
        _reportService = reportService;
    }

    [TickerQConstructor]
    public ReportJobs(
        IReportService reportService,
        ILogger<ReportJobs> logger)
    {
        _reportService = reportService;
        _logger = logger;
    }

    [TickerFunction("GenerateDailyReport")]
    public async Task GenerateDailyReport(
        TickerFunctionContext<ReportRequest> context,
        CancellationToken cancellationToken)
    {
        await _reportService.GenerateAsync(context.Request, cancellationToken);
        _logger?.LogInformation("Generated report {ReportId}", context.Request.ReportId);
    }
}
```

Rules:

- If exactly one constructor has `[TickerQConstructor]`, that constructor is used.
- If multiple constructors have `[TickerQConstructor]`, the generator emits an error (`TQ010`).
- If the class has multiple constructors and none are marked, the generator emits a warning (`TQ006`) and uses the first constructor.

See [Constructor Injection](/concepts/constructor-injection) for a conceptual overview.

