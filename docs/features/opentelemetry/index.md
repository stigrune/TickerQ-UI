# OpenTelemetry Integration

TickerQ.Instrumentation.OpenTelemetry provides distributed tracing and structured logging for TickerQ jobs, enabling comprehensive observability in distributed systems.

## Sections

### [Installation](./installation)
Install the OpenTelemetry package and configure instrumentation.

### [Setup](./setup)
Basic OpenTelemetry configuration and integration with TickerQ.

### [Tracing](./tracing)
Distributed tracing, spans, and trace correlation across services.

### [Metrics](./metrics)
Job execution metrics, performance counters, and custom metrics.

### [Logging](./logging)
Structured logging, log correlation, and centralized log management.

### [Exporters](./exporters)
Configure exporters for Jaeger, Zipkin, OTLP, and other observability platforms.

### [Integration](./integration)
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

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation();
});
```

## Key Features

- **Distributed Tracing**: Track job execution across multiple services
- **Performance Metrics**: Monitor job execution times and success rates
- **Structured Logging**: Correlated logs with trace and span IDs
- **Custom Instrumentation**: Add custom spans and metrics to your jobs
- **Multiple Exporters**: Support for Jaeger, Zipkin, OTLP, and more

## Observability Benefits

- **End-to-End Visibility**: Track jobs from trigger to completion
- **Performance Monitoring**: Identify bottlenecks and optimization opportunities
- **Error Tracking**: Detailed error traces and failure analysis
- **Dependency Mapping**: Visualize job dependencies and service interactions
- **SLA Monitoring**: Track job execution against performance targets

## Next Steps

- [Installation Guide](./installation) - Set up OpenTelemetry instrumentation
- [Tracing Configuration](./tracing) - Configure distributed tracing
- [Metrics Setup](./metrics) - Monitor job performance
