# OpenTelemetry Integration

Integrate TickerQ OpenTelemetry instrumentation with various observability platforms, APM tools, and monitoring systems.

## Observability Platform Integrations

### Jaeger

```csharp
using OpenTelemetry.Trace;
using TickerQ.Instrumentation.OpenTelemetry;

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddJaegerExporter(jaegerOptions =>
               {
                   jaegerOptions.AgentHost = "jaeger-agent";
                   jaegerOptions.AgentPort = 6831;
                   jaegerOptions.Endpoint = new Uri("http://jaeger-collector:14268/api/traces");
               });
    });

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation(telemetryOptions =>
    {
        telemetryOptions.ServiceName = "TickerQ-Jobs";
        telemetryOptions.ServiceVersion = "1.0.0";
        telemetryOptions.EnableJobTracing = true;
        telemetryOptions.EnablePerformanceMetrics = true;
    });
});
```

### Zipkin

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddZipkinExporter(zipkinOptions =>
               {
                   zipkinOptions.Endpoint = new Uri("http://zipkin:9411/api/v2/spans");
               });
    });

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation(telemetryOptions =>
    {
        telemetryOptions.TracingConfiguration = tracingConfig =>
        {
            tracingConfig.SampleRate = 0.1; // 10% sampling
            tracingConfig.IncludeJobPayload = false; // Security
            tracingConfig.MaxSpanAttributes = 50;
        };
    });
});
```

### OTLP (OpenTelemetry Protocol)

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddOtlpExporter(otlpOptions =>
               {
                   otlpOptions.Endpoint = new Uri("http://otel-collector:4317");
                   otlpOptions.Headers = "api-key=your-api-key";
                   otlpOptions.Protocol = OtlpExportProtocol.Grpc;
               });
    })
    .WithMetrics(metrics =>
    {
        metrics.AddMeter("TickerQ")
               .AddOtlpExporter();
    });
```

## Cloud APM Services

### Azure Application Insights

```csharp
builder.Services.AddApplicationInsightsTelemetry(aiOptions =>
{
    aiOptions.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
});

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddApplicationInsightsTraceExporter();
    });

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation(telemetryOptions =>
    {
        telemetryOptions.EnableAzureIntegration = true;
        telemetryOptions.AzureConfiguration = azureConfig =>
        {
            azureConfig.CorrelateWithApplicationInsights = true;
            azureConfig.EnableCustomDimensions = true;
            azureConfig.TrackJobDependencies = true;
        };
    });
});
```

### AWS X-Ray

```csharp
builder.Services.AddAWSXRayTracing();

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddXRayTraceExporter();
    });

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation(telemetryOptions =>
    {
        telemetryOptions.EnableAwsIntegration = true;
        telemetryOptions.AwsConfiguration = awsConfig =>
        {
            awsConfig.ServiceMap = "TickerQ-Jobs";
            awsConfig.EnableSubsegments = true;
            awsConfig.TrackDownstreamServices = true;
        };
    });
});
```

### Google Cloud Trace

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddGoogleCloudTraceExporter(gcpOptions =>
               {
                   gcpOptions.ProjectId = "my-gcp-project";
               });
    });

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation(telemetryOptions =>
    {
        telemetryOptions.EnableGcpIntegration = true;
        telemetryOptions.GcpConfiguration = gcpConfig =>
        {
            gcpConfig.ProjectId = "my-gcp-project";
            gcpConfig.EnableCloudLogging = true;
            gcpConfig.TrackGcpResources = true;
        };
    });
});
```

## Metrics and Monitoring

### Prometheus

```csharp
using OpenTelemetry.Metrics;

builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics.AddMeter("TickerQ")
               .AddPrometheusExporter();
    });

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation(telemetryOptions =>
    {
        telemetryOptions.EnableMetrics = true;
        telemetryOptions.MetricsConfiguration = metricsConfig =>
        {
            metricsConfig.JobExecutionHistogram = true;
            metricsConfig.JobSuccessRate = true;
            metricsConfig.ActiveJobsGauge = true;
            metricsConfig.CustomMetrics = new[]
            {
                "job_queue_length",
                "job_processing_time",
                "job_retry_count"
            };
        };
    });
});

// Expose Prometheus endpoint
app.MapPrometheusScrapingEndpoint("/metrics");
```

### Grafana Integration

```json
{
  "dashboard": {
    "title": "TickerQ Job Monitoring",
    "panels": [
      {
        "title": "Job Execution Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(tickerq_jobs_executed_total[5m])",
            "legendFormat": "{{job_type}} - {{status}}"
          }
        ]
      },
      {
        "title": "Job Duration Distribution",
        "type": "heatmap",
        "targets": [
          {
            "expr": "tickerq_job_duration_seconds_bucket",
            "legendFormat": "{{le}}"
          }
        ]
      },
      {
        "title": "Active Jobs by Type",
        "type": "stat",
        "targets": [
          {
            "expr": "tickerq_active_jobs_total",
            "legendFormat": "{{job_type}}"
          }
        ]
      }
    ]
  }
}
```

### DataDog

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .AddOtlpExporter(otlpOptions =>
               {
                   otlpOptions.Endpoint = new Uri("https://trace.agent.datadoghq.com:4317");
                   otlpOptions.Headers = $"DD-API-KEY={builder.Configuration["DataDog:ApiKey"]}";
               });
    });

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation(telemetryOptions =>
    {
        telemetryOptions.EnableDataDogIntegration = true;
        telemetryOptions.DataDogConfiguration = ddConfig =>
        {
            ddConfig.Service = "tickerq-jobs";
            ddConfig.Environment = builder.Environment.EnvironmentName;
            ddConfig.Version = "1.0.0";
            ddConfig.EnableRumIntegration = true;
        };
    });
});
```

## Logging Integrations

### Structured Logging with Serilog

```csharp
builder.Host.UseSerilog((context, configuration) =>
{
    configuration
        .ReadFrom.Configuration(context.Configuration)
        .Enrich.WithOpenTelemetryTraceId()
        .Enrich.WithOpenTelemetrySpanId()
        .WriteTo.Console(new JsonFormatter())
        .WriteTo.OpenTelemetry(otlpOptions =>
        {
            otlpOptions.Endpoint = "http://otel-collector:4317";
            otlpOptions.IncludedData = IncludedData.TraceIdField | IncludedData.SpanIdField;
        });
});

builder.Services.AddTickerQ(options =>
{
    options.AddOpenTelemetryInstrumentation(telemetryOptions =>
    {
        telemetryOptions.EnableStructuredLogging = true;
        telemetryOptions.LoggingConfiguration = loggingConfig =>
        {
            loggingConfig.IncludeJobContext = true;
            loggingConfig.IncludeExecutionMetrics = true;
            loggingConfig.SanitizeSensitiveData = true;
        };
    });
});
```

### ELK Stack Integration

```csharp
builder.Services.AddOpenTelemetry()
    .WithLogging(logging =>
    {
        logging.AddOtlpExporter(otlpOptions =>
        {
            otlpOptions.Endpoint = new Uri("http://logstash:4317");
        });
    });

// Logstash configuration for TickerQ logs
/*
input {
  otlp {
    port => 4317
  }
}

filter {
  if [resource][service.name] == "TickerQ-Jobs" {
    mutate {
      add_field => { "[@metadata][index]" => "tickerq-jobs-%{+YYYY.MM.dd}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "%{[@metadata][index]}"
  }
}
*/
```

## Container and Orchestration

### Docker Compose with Observability Stack

```yaml
# docker-compose.observability.yml
version: '3.8'
services:
  # Application
  tickerq-app:
    build: .
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_SERVICE_NAME=tickerq-jobs
    depends_on:
      - otel-collector
    networks:
      - observability

  # OpenTelemetry Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yml"]
    volumes:
      - ./otel-collector-config.yml:/etc/otel-collector-config.yml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    networks:
      - observability

  # Jaeger
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    networks:
      - observability

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - observability

  # Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-storage:/var/lib/grafana
    networks:
      - observability

networks:
  observability:

volumes:
  grafana-storage:
```

### Kubernetes with OpenTelemetry Operator

```yaml
# otel-collector.yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: tickerq-collector
spec:
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    
    processors:
      batch:
      
    exporters:
      jaeger:
        endpoint: jaeger-collector:14250
        tls:
          insecure: true
      prometheus:
        endpoint: "0.0.0.0:8889"
        
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [jaeger]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [prometheus]
---
# Application deployment with auto-instrumentation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tickerq-app
  annotations:
    instrumentation.opentelemetry.io/inject-dotnet: "true"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tickerq-app
  template:
    metadata:
      labels:
        app: tickerq-app
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://tickerq-collector:4317"
        - name: OTEL_SERVICE_NAME
          value: "tickerq-jobs"
```

## Custom Instrumentation

### Custom Spans and Metrics

```csharp
public class CustomJobInstrumentation
{
    private static readonly ActivitySource ActivitySource = new("MyApp.Jobs");
    private static readonly Meter Meter = new("MyApp.Jobs");
    private static readonly Counter<long> JobCounter = Meter.CreateCounter<long>("custom_job_executions");
    private static readonly Histogram<double> JobDuration = Meter.CreateHistogram<double>("custom_job_duration");

    public async Task ExecuteJobWithInstrumentation<T>(T job) where T : IJob
    {
        using var activity = ActivitySource.StartActivity($"Job.{typeof(T).Name}");
        activity?.SetTag("job.type", typeof(T).Name);
        activity?.SetTag("job.id", job.Id);
        
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            await job.ExecuteAsync();
            
            activity?.SetStatus(ActivityStatusCode.Ok);
            JobCounter.Add(1, new("status", "success"), new("type", typeof(T).Name));
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            JobCounter.Add(1, new("status", "error"), new("type", typeof(T).Name));
            throw;
        }
        finally
        {
            stopwatch.Stop();
            JobDuration.Record(stopwatch.Elapsed.TotalSeconds, 
                new("type", typeof(T).Name));
        }
    }
}

// Register custom instrumentation
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("MyApp.Jobs")
               .AddSource("TickerQ");
    })
    .WithMetrics(metrics =>
    {
        metrics.AddMeter("MyApp.Jobs")
               .AddMeter("TickerQ");
    });
```

### Correlation with External Systems

```csharp
public class ExternalServiceInstrumentation
{
    public async Task<T> CallExternalServiceAsync<T>(string serviceName, Func<Task<T>> serviceCall)
    {
        using var activity = Activity.Current?.Source.StartActivity($"ExternalService.{serviceName}");
        
        // Propagate trace context
        var headers = new Dictionary<string, string>();
        DistributedContextPropagator.Current.Inject(
            new PropagationContext(Activity.Current?.Context ?? default, Baggage.Current),
            headers,
            (carrier, key, value) => carrier[key] = value);
        
        activity?.SetTag("service.name", serviceName);
        activity?.SetTag("service.external", true);
        
        try
        {
            var result = await serviceCall();
            activity?.SetStatus(ActivityStatusCode.Ok);
            return result;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

## Performance and Sampling

### Adaptive Sampling

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("TickerQ")
               .SetSampler(new TraceIdRatioBasedSampler(0.1)) // 10% sampling
               .AddProcessor(new BatchActivityExportProcessor(
                   new JaegerExporter(jaegerOptions),
                   maxQueueSize: 2048,
                   scheduledDelayMilliseconds: 5000,
                   exporterTimeoutMilliseconds: 30000,
                   maxExportBatchSize: 512));
    });

// Custom sampler for different job types
public class JobTypeSampler : Sampler
{
    public override SamplingResult ShouldSample(in SamplingParameters samplingParameters)
    {
        var jobType = samplingParameters.Tags.FirstOrDefault(t => t.Key == "job.type").Value;
        
        return jobType switch
        {
            "CriticalJob" => new SamplingResult(SamplingDecision.RecordAndSample),
            "BatchJob" => new SamplingResult(SamplingDecision.Drop),
            _ => new SamplingResult(SamplingDecision.RecordAndSample)
        };
    }
}
```

## Next Steps

- [Tracing Configuration](./tracing) - Configure distributed tracing
- [Metrics Setup](./metrics) - Monitor job performance
- [Exporters Guide](./exporters) - Configure observability platforms
