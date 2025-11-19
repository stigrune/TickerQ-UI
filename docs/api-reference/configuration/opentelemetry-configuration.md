# OpenTelemetry Configuration

Enable OpenTelemetry tracing and observability for TickerQ.

## AddOpenTelemetryInstrumentation

Enable OpenTelemetry tracing (requires `TickerQ.Instrumentation.OpenTelemetry` package).

**Method:**
```csharp
TickerOptionsBuilder<TTimeTicker, TCronTicker> AddOpenTelemetryInstrumentation<TTimeTicker, TCronTicker>(
    this TickerOptionsBuilder<TTimeTicker, TCronTicker> tickerConfiguration);
```

**Example:**
```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation();
});
```

## Complete Setup Example

```csharp
using OpenTelemetry;
using OpenTelemetry.Trace;

// Configure OpenTelemetry
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddConsoleExporter();
    });

// Add TickerQ with OpenTelemetry instrumentation
builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation();
});
```

## See Also

- [OpenTelemetry Integration](../../features/opentelemetry) - Complete setup guide
- [Configuration Overview](./index) - All configuration sections
