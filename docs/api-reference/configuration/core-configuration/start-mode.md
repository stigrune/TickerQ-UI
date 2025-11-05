# Start Mode

Control when TickerQ starts processing jobs in your application.

## UseTickerQ

Activate the TickerQ job processor with optional start mode control.

**Method:**
```csharp
IApplicationBuilder UseTickerQ(TickerQStartMode startMode = TickerQStartMode.Immediate);
```

## Start Modes

### Immediate Start (Default)

TickerQ starts processing jobs immediately when `UseTickerQ()` is called.

```csharp
var app = builder.Build();

// Starts processing jobs immediately
app.UseTickerQ();

app.Run();
```

**Behavior:**
- Jobs start executing as soon as the application starts
- No manual start required
- Best for most scenarios

### Manual Start

TickerQ is configured but doesn't start processing until explicitly started.

```csharp
var app = builder.Build();

// Configure but don't start
app.UseTickerQ(TickerQStartMode.Manual);

// Later, start manually
var scheduler = app.Services.GetRequiredService<ITickerQHostScheduler>();
await scheduler.StartAsync();

app.Run();
```

**Behavior:**
- Jobs are queued but not executed
- Requires explicit start call
- Useful for controlled startup sequences

## When to Use Manual Start

Use manual start when:
- You need to perform initialization before jobs run
- Database migrations must complete first
- External services must be ready
- You want to coordinate startup with other systems
- Running in test environments where you control execution

## Manual Start Example

```csharp
var app = builder.Build();

app.UseTickerQ(TickerQStartMode.Manual);

// Run migrations
await MigrateDatabaseAsync(app.Services);

// Wait for external service
await WaitForExternalServiceAsync();

// Now start TickerQ
var scheduler = app.Services.GetRequiredService<ITickerQHostScheduler>();
await scheduler.StartAsync();

app.Run();
```

## Getting the Scheduler Service

```csharp
// In Program.cs or Startup
var scheduler = app.Services.GetRequiredService<ITickerQHostScheduler>();

// Start processing
await scheduler.StartAsync();

// Stop processing (if needed)
await scheduler.StopAsync();
```

## Checking Scheduler Status

```csharp
var scheduler = app.Services.GetRequiredService<ITickerQHostScheduler>();

if (scheduler.IsRunning)
{
    Console.WriteLine("TickerQ is processing jobs");
}
```

## Lifecycle

### Immediate Start Lifecycle

1. Application starts
2. `UseTickerQ()` called
3. TickerQ immediately begins processing
4. Jobs execute as scheduled

### Manual Start Lifecycle

1. Application starts
2. `UseTickerQ(TickerQStartMode.Manual)` called
3. TickerQ configured but idle
4. Your initialization code runs
5. `scheduler.StartAsync()` called
6. TickerQ begins processing
7. Jobs execute as scheduled

## Best Practices

1. **Use Immediate Start** for most applications
2. **Use Manual Start** when you need initialization control
3. **Start as early as possible** in manual mode - delays affect job scheduling
4. **Handle start failures** - check if start was successful

## Error Handling

```csharp
try
{
    var scheduler = app.Services.GetRequiredService<ITickerQHostScheduler>();
    await scheduler.StartAsync();
    
    if (!scheduler.IsRunning)
    {
        throw new Exception("TickerQ scheduler failed to start");
    }
}
catch (Exception ex)
{
    _logger.LogError(ex, "Failed to start TickerQ scheduler");
    // Handle startup failure
}
```

## See Also

- [Scheduler Configuration](./scheduler-configuration) - Scheduler behavior settings
- [Exception Handling](./exception-handling) - Error handling configuration
- [Core Configuration Overview](./index) - All core configuration options

