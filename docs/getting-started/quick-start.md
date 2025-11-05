# Quick Start

Get up and running with TickerQ in under 5 minutes.

## Prerequisites

- .NET 8.0 or later
- A .NET project (Console, Web API, or ASP.NET Core)

## Step 1: Install TickerQ

```bash
dotnet add package TickerQ
```

## Step 2: Register Services

Add TickerQ to your `Program.cs`:

```csharp
using TickerQ.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddTickerQ();

var app = builder.Build();
app.UseTickerQ(); // Activate job processor
app.Run();
```

## Step 3: Create a Job Function

Create a job function with the `[TickerFunction]` attribute:

```csharp
using TickerQ.Utilities.Base;

public class MyJobs
{
    [TickerFunction("HelloWorld")]
    public async Task HelloWorld(
        TickerFunctionContext context,
        CancellationToken cancellationToken)
    {
        Console.WriteLine($"Hello from TickerQ! Job ID: {context.Id}");
        Console.WriteLine($"Scheduled at: {DateTime.UtcNow:HH:mm:ss}");
    }
}
```

## Step 4: Schedule the Job

Inject the manager and schedule your job:

```csharp
using TickerQ.Utilities.Entities;
using TickerQ.Utilities.Interfaces.Managers;

public class MyService
{
    private readonly ITimeTickerManager<TimeTickerEntity> _timeTickerManager;
    
    public MyService(ITimeTickerManager<TimeTickerEntity> timeTickerManager)
    {
        _timeTickerManager = timeTickerManager;
    }
    
    public async Task ScheduleJob()
    {
        var result = await _timeTickerManager.AddAsync(new TimeTickerEntity
        {
            Function = "HelloWorld",
            ExecutionTime = DateTime.UtcNow.AddSeconds(10) // Run in 10 seconds
        });
        
        if (result.IsSucceeded)
        {
            Console.WriteLine($"Job scheduled! ID: {result.Result.Id}");
        }
    }
}
```

## Step 5: Run Your Application

```bash
dotnet run
```

Wait 10 seconds and you should see:
```
Job scheduled! ID: {guid}
Hello from TickerQ! Job ID: {guid}
Scheduled at: {time}
```

## Complete Minimal Example

Here's a complete, runnable example:

**Program.cs:**
```csharp
using TickerQ.DependencyInjection;
using TickerQ.Utilities.Entities;
using TickerQ.Utilities.Interfaces.Managers;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddTickerQ();

var app = builder.Build();

// Schedule a job immediately after startup
using (var scope = app.Services.CreateScope())
{
    var manager = scope.ServiceProvider
        .GetRequiredService<ITimeTickerManager<TimeTickerEntity>>();
    
    await manager.AddAsync(new TimeTickerEntity
    {
        Function = "HelloWorld",
        ExecutionTime = DateTime.UtcNow.AddSeconds(5)
    });
}

app.UseTickerQ();
app.Run();
```

**MyJobs.cs:**
```csharp
using TickerQ.Utilities.Base;

public class MyJobs
{
    [TickerFunction("HelloWorld")]
    public async Task HelloWorld(
        TickerFunctionContext context,
        CancellationToken cancellationToken)
    {
        Console.WriteLine($"TickerQ is working! Job executed at {DateTime.UtcNow:HH:mm:ss}");
    }
}
```

Run it and watch the job execute after 5 seconds!

## Verification Checklist

- Services registered without errors
- `UseTickerQ()` called without exceptions
- Job function exists with `[TickerFunction]` attribute
- Function name matches exactly between attribute and entity
- Job executes at scheduled time
- Console output appears

## Troubleshooting

**Job doesn't execute?**
- Check function name matches exactly
- Verify `UseTickerQ()` is called
- Check application logs for errors
- Ensure job execution time is in the future (or past for immediate execution)

**Function not found error?**
- Rebuild the project (source generator needs to run)
- Verify `[TickerFunction]` attribute is present
- Check function name spelling

## What's Next?

- [Job Types](/concepts/job-types) - Understanding TimeTicker vs CronTicker
- [Installation Guide](/getting-started/installation) - Complete setup options
- [Complete Example](/examples/complete-example) - See a full working example

