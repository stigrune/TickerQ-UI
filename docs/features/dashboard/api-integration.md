# API Integration

Learn how to integrate the TickerQ Dashboard with external systems, APIs, and custom applications.

## REST API Overview

The TickerQ Dashboard provides a comprehensive REST API for programmatic access to all dashboard functionality.

### Base Configuration

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.EnableApi = true;
        dashboardOptions.ApiBasePath = "/api/tickerq";
        dashboardOptions.ApiVersion = "v1";
    });
});
```

### Authentication

```csharp
dashboardOptions.ApiOptions = apiOptions =>
{
    apiOptions.RequireAuthentication = true;
    apiOptions.AuthenticationScheme = "Bearer";
    apiOptions.RequiredScopes = new[] { "tickerq:read", "tickerq:write" };
};
```

## Job Management API

### Job Operations

#### Get All Jobs

```http
GET /api/tickerq/jobs
Authorization: Bearer {token}
Content-Type: application/json
```

**Query Parameters:**
- `status` - Filter by job status (pending, running, completed, failed)
- `type` - Filter by job type
- `limit` - Number of jobs to return (default: 50)
- `offset` - Pagination offset
- `from` - Start date filter (ISO 8601)
- `to` - End date filter (ISO 8601)

**Response:**
```json
{
    "jobs": [
        {
            "id": "job-123",
            "name": "Email Notification Job",
            "type": "EmailNotificationJob",
            "status": "completed",
            "createdAt": "2024-01-15T10:30:00Z",
            "executedAt": "2024-01-15T10:31:00Z",
            "completedAt": "2024-01-15T10:31:30Z",
            "duration": "00:00:30",
            "priority": "normal",
            "retryCount": 0,
            "maxRetries": 3
        }
    ],
    "totalCount": 150,
    "hasMore": true
}
```

#### Get Job Details

```http
GET /api/tickerq/jobs/{jobId}
Authorization: Bearer {token}
```

**Response:**
```json
{
    "id": "job-123",
    "name": "Email Notification Job",
    "type": "EmailNotificationJob",
    "status": "completed",
    "parameters": {
        "recipientEmail": "user@example.com",
        "templateId": "welcome-email"
    },
    "result": {
        "messageId": "msg-456",
        "deliveryStatus": "delivered"
    },
    "error": null,
    "executionHistory": [
        {
            "attempt": 1,
            "startedAt": "2024-01-15T10:31:00Z",
            "completedAt": "2024-01-15T10:31:30Z",
            "status": "completed",
            "duration": "00:00:30"
        }
    ]
}
```

#### Execute Job

```http
POST /api/tickerq/jobs/{jobId}/execute
Authorization: Bearer {token}
Content-Type: application/json

{
    "priority": "high",
    "delay": "00:05:00"
}
```

#### Cancel Job

```http
POST /api/tickerq/jobs/{jobId}/cancel
Authorization: Bearer {token}
```

#### Retry Failed Job

```http
POST /api/tickerq/jobs/{jobId}/retry
Authorization: Bearer {token}
Content-Type: application/json

{
    "resetRetryCount": false
}
```

### Job Creation

#### Schedule New Job

```http
POST /api/tickerq/jobs
Authorization: Bearer {token}
Content-Type: application/json

{
    "type": "EmailNotificationJob",
    "name": "Welcome Email",
    "parameters": {
        "recipientEmail": "newuser@example.com",
        "templateId": "welcome-email"
    },
    "priority": "normal",
    "delay": "00:00:00",
    "maxRetries": 3,
    "cronExpression": null
}
```

#### Schedule Recurring Job

```http
POST /api/tickerq/jobs/recurring
Authorization: Bearer {token}
Content-Type: application/json

{
    "type": "DataBackupJob",
    "name": "Daily Backup",
    "cronExpression": "0 2 * * *",
    "parameters": {
        "backupType": "incremental",
        "destination": "s3://backups/"
    },
    "enabled": true
}
```

## Queue Management API

### Queue Operations

#### Get Queue Status

```http
GET /api/tickerq/queues
Authorization: Bearer {token}
```

**Response:**
```json
{
    "queues": [
        {
            "name": "default",
            "status": "running",
            "pendingJobs": 25,
            "runningJobs": 5,
            "completedJobs": 1250,
            "failedJobs": 12,
            "workers": 10,
            "throughput": {
                "jobsPerMinute": 15.5,
                "jobsPerHour": 930
            }
        }
    ]
}
```

#### Pause Queue

```http
POST /api/tickerq/queues/{queueName}/pause
Authorization: Bearer {token}
```

#### Resume Queue

```http
POST /api/tickerq/queues/{queueName}/resume
Authorization: Bearer {token}
```

#### Clear Queue

```http
POST /api/tickerq/queues/{queueName}/clear
Authorization: Bearer {token}
Content-Type: application/json

{
    "clearPending": true,
    "clearFailed": false
}
```

## Metrics and Analytics API

### System Metrics

#### Get System Overview

```http
GET /api/tickerq/metrics/system
Authorization: Bearer {token}
```

**Response:**
```json
{
    "timestamp": "2024-01-15T10:30:00Z",
    "systemHealth": "healthy",
    "totalJobs": 15000,
    "activeWorkers": 25,
    "queueLength": 150,
    "memoryUsage": {
        "used": "512MB",
        "available": "2GB",
        "percentage": 25.6
    },
    "cpuUsage": {
        "percentage": 15.2,
        "cores": 8
    }
}
```

#### Get Performance Metrics

```http
GET /api/tickerq/metrics/performance?from=2024-01-01&to=2024-01-15
Authorization: Bearer {token}
```

**Response:**
```json
{
    "period": {
        "from": "2024-01-01T00:00:00Z",
        "to": "2024-01-15T23:59:59Z"
    },
    "summary": {
        "totalJobs": 50000,
        "successfulJobs": 48500,
        "failedJobs": 1500,
        "successRate": 97.0,
        "averageExecutionTime": "00:00:45",
        "throughput": {
            "jobsPerDay": 3333,
            "peakJobsPerHour": 250
        }
    },
    "trends": [
        {
            "date": "2024-01-01",
            "jobsExecuted": 3200,
            "successRate": 96.5,
            "averageExecutionTime": "00:00:42"
        }
    ]
}
```

### Job Type Analytics

```http
GET /api/tickerq/metrics/job-types
Authorization: Bearer {token}
```

**Response:**
```json
{
    "jobTypes": [
        {
            "type": "EmailNotificationJob",
            "totalExecutions": 15000,
            "successRate": 98.5,
            "averageExecutionTime": "00:00:15",
            "failureReasons": [
                {
                    "reason": "SMTP timeout",
                    "count": 150,
                    "percentage": 1.0
                }
            ]
        }
    ]
}
```

## Real-time Integration

### WebSocket API

#### Connection Setup

```javascript
const ws = new WebSocket('ws://localhost:5000/api/tickerq/ws');

ws.onopen = function() {
    // Subscribe to job updates
    ws.send(JSON.stringify({
        action: 'subscribe',
        topics: ['job-status', 'queue-metrics', 'system-health']
    }));
};
```

#### Message Handling

```javascript
ws.onmessage = function(event) {
    const message = JSON.parse(event.data);
    
    switch(message.type) {
        case 'job-status-changed':
            handleJobStatusChange(message.data);
            break;
        case 'queue-metrics-updated':
            updateQueueMetrics(message.data);
            break;
        case 'system-health-alert':
            showHealthAlert(message.data);
            break;
    }
};

function handleJobStatusChange(data) {
    console.log(`Job ${data.jobId} status changed to ${data.status}`);
    updateJobInUI(data.jobId, data.status);
}
```

### SignalR Integration

#### Hub Configuration

```csharp
// Configure SignalR hub
public class TickerQHub : Hub
{
    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
    }
    
    public async Task LeaveGroup(string groupName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
    }
}
```

#### Client Integration

```javascript
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/api/tickerq/hub")
    .build();

connection.start().then(function () {
    // Join specific groups for targeted updates
    connection.invoke("JoinGroup", "job-updates");
    connection.invoke("JoinGroup", "system-metrics");
});

// Handle job status updates
connection.on("JobStatusChanged", function (jobId, status, details) {
    updateJobStatus(jobId, status, details);
});

// Handle system metrics updates
connection.on("MetricsUpdated", function (metrics) {
    updateDashboardMetrics(metrics);
});
```

## Webhook Integration

### Webhook Configuration

```csharp
dashboardOptions.WebhookOptions = webhookOptions =>
{
    webhookOptions.EnableWebhooks = true;
    webhookOptions.WebhookEndpoints = new[]
    {
        new WebhookEndpoint
        {
            Url = "https://api.example.com/tickerq/webhook",
            Events = new[] { "job.completed", "job.failed", "system.alert" },
            Secret = "webhook-secret-key",
            RetryPolicy = new RetryPolicy
            {
                MaxRetries = 3,
                RetryDelay = TimeSpan.FromSeconds(30)
            }
        }
    };
};
```

### Webhook Payload

```json
{
    "event": "job.completed",
    "timestamp": "2024-01-15T10:31:30Z",
    "data": {
        "jobId": "job-123",
        "jobType": "EmailNotificationJob",
        "status": "completed",
        "duration": "00:00:30",
        "result": {
            "messageId": "msg-456",
            "deliveryStatus": "delivered"
        }
    },
    "signature": "sha256=abc123..."
}
```

### Webhook Verification

```csharp
public class WebhookController : ControllerBase
{
    [HttpPost("tickerq/webhook")]
    public async Task<IActionResult> HandleWebhook([FromBody] WebhookPayload payload)
    {
        var signature = Request.Headers["X-TickerQ-Signature"].FirstOrDefault();
        
        if (!VerifyWebhookSignature(payload, signature))
        {
            return Unauthorized();
        }
        
        await ProcessWebhookEvent(payload);
        return Ok();
    }
    
    private bool VerifyWebhookSignature(WebhookPayload payload, string signature)
    {
        var secret = _configuration["TickerQ:WebhookSecret"];
        var computedSignature = ComputeHmacSha256(payload, secret);
        return signature == $"sha256={computedSignature}";
    }
}
```

## External System Integration

### Database Integration

#### Custom Data Storage

```csharp
// Custom job data repository
public class CustomJobRepository : IJobRepository
{
    private readonly IDbContext _context;
    
    public async Task<Job> GetJobAsync(string jobId)
    {
        return await _context.Jobs
            .Include(j => j.ExecutionHistory)
            .FirstOrDefaultAsync(j => j.Id == jobId);
    }
    
    public async Task SaveJobAsync(Job job)
    {
        _context.Jobs.Update(job);
        await _context.SaveChangesAsync();
        
        // Trigger external system notification
        await NotifyExternalSystem(job);
    }
}
```

#### Data Synchronization

```csharp
// Sync job data with external systems
public class JobSyncService : IHostedService
{
    public async Task SyncJobData()
    {
        var recentJobs = await _jobRepository.GetRecentJobsAsync();
        
        foreach (var job in recentJobs)
        {
            await _externalApiClient.SyncJobAsync(new ExternalJobData
            {
                Id = job.Id,
                Status = job.Status,
                ExecutedAt = job.ExecutedAt,
                Duration = job.Duration
            });
        }
    }
}
```

### Monitoring Integration

#### Prometheus Metrics

```csharp
// Export metrics to Prometheus
dashboardOptions.MetricsOptions = metricsOptions =>
{
    metricsOptions.EnablePrometheusExport = true;
    metricsOptions.PrometheusEndpoint = "/metrics";
    metricsOptions.CustomMetrics = new[]
    {
        "tickerq_jobs_total",
        "tickerq_job_duration_seconds",
        "tickerq_queue_length"
    };
};
```

#### Application Insights

```csharp
// Integration with Application Insights
builder.Services.AddApplicationInsightsTelemetry();

dashboardOptions.TelemetryOptions = telemetryOptions =>
{
    telemetryOptions.EnableApplicationInsights = true;
    telemetryOptions.TrackJobExecutions = true;
    telemetryOptions.TrackPerformanceCounters = true;
};
```

## SDK and Client Libraries

### .NET Client SDK

```csharp
// TickerQ Dashboard Client SDK
public class TickerQDashboardClient
{
    private readonly HttpClient _httpClient;
    private readonly string _baseUrl;
    
    public async Task<JobDetails> GetJobAsync(string jobId)
    {
        var response = await _httpClient.GetAsync($"{_baseUrl}/api/tickerq/jobs/{jobId}");
        response.EnsureSuccessStatusCode();
        
        var json = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<JobDetails>(json);
    }
    
    public async Task<string> ScheduleJobAsync<T>(T jobData, JobOptions options = null)
    {
        var request = new ScheduleJobRequest
        {
            Type = typeof(T).Name,
            Parameters = jobData,
            Options = options ?? new JobOptions()
        };
        
        var response = await _httpClient.PostAsJsonAsync($"{_baseUrl}/api/tickerq/jobs", request);
        response.EnsureSuccessStatusCode();
        
        var result = await response.Content.ReadFromJsonAsync<ScheduleJobResponse>();
        return result.JobId;
    }
}
```

### JavaScript Client SDK

```javascript
// JavaScript SDK for TickerQ Dashboard
class TickerQClient {
    constructor(baseUrl, apiKey) {
        this.baseUrl = baseUrl;
        this.apiKey = apiKey;
    }
    
    async getJobs(filter = {}) {
        const params = new URLSearchParams(filter);
        const response = await fetch(`${this.baseUrl}/api/tickerq/jobs?${params}`, {
            headers: {
                'Authorization': `Bearer ${this.apiKey}`,
                'Content-Type': 'application/json'
            }
        });
        
        return await response.json();
    }
    
    async scheduleJob(jobType, parameters, options = {}) {
        const response = await fetch(`${this.baseUrl}/api/tickerq/jobs`, {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${this.apiKey}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                type: jobType,
                parameters: parameters,
                ...options
            })
        });
        
        return await response.json();
    }
}
```

## Next Steps

- [Dashboard Features](./features) - Explore all dashboard capabilities
- [Customization Guide](./customization) - Customize dashboard appearance
- [Authentication Setup](./authentication) - Secure API access
