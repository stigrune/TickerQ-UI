# Dashboard Features

Explore the comprehensive features available in the TickerQ Dashboard for monitoring and managing your background jobs.

## Job Management

### Job Overview

The dashboard provides a comprehensive view of all your background jobs:

- **Real-time Status Updates** - Live job status monitoring
- **Job History** - Complete execution history with timestamps
- **Performance Metrics** - Execution time, success rates, and resource usage
- **Error Tracking** - Detailed error logs and stack traces

### Job Operations

#### Manual Job Execution

```csharp
// Trigger jobs manually from the dashboard
[HttpPost("api/jobs/{id}/execute")]
public async Task<IActionResult> ExecuteJob(string id)
{
    await _jobManager.ExecuteJobAsync(id);
    return Ok();
}
```

#### Job Scheduling

- **Immediate Execution** - Run jobs instantly
- **Scheduled Execution** - Set future execution times
- **Recurring Jobs** - Configure cron expressions
- **Bulk Operations** - Manage multiple jobs simultaneously

### Job Filtering and Search

#### Advanced Filters

- **Status Filtering** - Filter by Pending, Running, Completed, Failed
- **Date Range** - Filter jobs by execution time
- **Job Type** - Filter by specific job classes
- **Priority Levels** - Filter by job priority

#### Search Capabilities

```javascript
// Dashboard search functionality
const searchJobs = (query) => {
    return jobs.filter(job => 
        job.name.toLowerCase().includes(query.toLowerCase()) ||
        job.type.toLowerCase().includes(query.toLowerCase()) ||
        job.description.toLowerCase().includes(query.toLowerCase())
    );
};
```

## Real-time Monitoring

### Live Updates

The dashboard uses SignalR for real-time updates:

```csharp
// Configure SignalR for real-time updates
builder.Services.AddSignalR();

app.MapHub<TickerQHub>("/tickerq/hub");
```

### Performance Metrics

#### System Metrics

- **CPU Usage** - Real-time CPU utilization
- **Memory Usage** - Memory consumption tracking
- **Queue Length** - Number of pending jobs
- **Throughput** - Jobs processed per minute/hour

#### Job Metrics

- **Execution Time** - Average, min, max execution times
- **Success Rate** - Percentage of successful executions
- **Failure Rate** - Error frequency and patterns
- **Resource Usage** - Memory and CPU per job type

### Alerts and Notifications

#### Configurable Alerts

```csharp
dashboardOptions.AlertOptions = alertOptions =>
{
    alertOptions.EnableAlerts = true;
    alertOptions.AlertThresholds = new AlertThresholds
    {
        FailureRate = 0.1, // 10% failure rate
        QueueLength = 1000,
        ExecutionTime = TimeSpan.FromMinutes(30)
    };
};
```

#### Notification Channels

- **Email Notifications** - Send alerts via email
- **Slack Integration** - Post alerts to Slack channels
- **Webhook Notifications** - Custom webhook endpoints
- **In-Dashboard Notifications** - Real-time dashboard alerts

## Data Visualization

### Charts and Graphs

#### Job Execution Timeline

```javascript
// Job execution timeline chart
const timelineChart = {
    type: 'line',
    data: {
        labels: timestamps,
        datasets: [{
            label: 'Jobs Executed',
            data: executionCounts,
            borderColor: '#3b82f6',
            backgroundColor: 'rgba(59, 130, 246, 0.1)'
        }]
    }
};
```

#### Success/Failure Rates

- **Pie Charts** - Success vs failure distribution
- **Bar Charts** - Job type performance comparison
- **Heat Maps** - Execution patterns over time
- **Trend Lines** - Performance trends and predictions

### Performance Analytics

#### Historical Analysis

- **Execution Trends** - Performance over time
- **Peak Usage Times** - Identify busy periods
- **Resource Utilization** - Optimize resource allocation
- **Error Patterns** - Identify recurring issues

#### Predictive Analytics

```csharp
dashboardOptions.AnalyticsOptions = analyticsOptions =>
{
    analyticsOptions.EnablePredictiveAnalytics = true;
    analyticsOptions.PredictionWindow = TimeSpan.FromDays(7);
    analyticsOptions.AnalyticsRetention = TimeSpan.FromDays(90);
};
```

## Queue Management

### Queue Monitoring

#### Queue Status

- **Queue Length** - Number of pending jobs
- **Processing Rate** - Jobs processed per unit time
- **Worker Status** - Active worker threads
- **Queue Health** - Overall queue performance

#### Queue Operations

```csharp
// Queue management operations
public class QueueController : ControllerBase
{
    [HttpPost("api/queues/{name}/pause")]
    public async Task<IActionResult> PauseQueue(string name)
    {
        await _queueManager.PauseQueueAsync(name);
        return Ok();
    }
    
    [HttpPost("api/queues/{name}/resume")]
    public async Task<IActionResult> ResumeQueue(string name)
    {
        await _queueManager.ResumeQueueAsync(name);
        return Ok();
    }
}
```

### Priority Management

#### Priority Queues

- **High Priority** - Critical jobs processed first
- **Normal Priority** - Standard job processing
- **Low Priority** - Background maintenance tasks
- **Custom Priorities** - Application-specific priority levels

#### Dynamic Priority Adjustment

```csharp
// Adjust job priority dynamically
[HttpPut("api/jobs/{id}/priority")]
public async Task<IActionResult> UpdateJobPriority(string id, JobPriority priority)
{
    await _jobManager.UpdateJobPriorityAsync(id, priority);
    return Ok();
}
```

## Error Handling and Debugging

### Error Dashboard

#### Error Overview

- **Error Summary** - Total errors by type and time
- **Error Trends** - Error frequency patterns
- **Critical Errors** - High-priority error alerts
- **Error Resolution** - Tracking error fixes

#### Detailed Error Information

```csharp
// Error details model
public class JobError
{
    public string JobId { get; set; }
    public string ErrorMessage { get; set; }
    public string StackTrace { get; set; }
    public DateTime OccurredAt { get; set; }
    public string JobType { get; set; }
    public Dictionary<string, object> Context { get; set; }
}
```

### Debugging Tools

#### Job Execution Logs

- **Detailed Logging** - Step-by-step execution logs
- **Context Information** - Job parameters and environment
- **Performance Profiling** - Execution time breakdown
- **Resource Usage** - Memory and CPU consumption

#### Retry Management

```csharp
// Configure retry policies
dashboardOptions.RetryOptions = retryOptions =>
{
    retryOptions.MaxRetryAttempts = 3;
    retryOptions.RetryDelay = TimeSpan.FromMinutes(5);
    retryOptions.ExponentialBackoff = true;
    retryOptions.RetryOnlyOnSpecificErrors = new[]
    {
        typeof(HttpRequestException),
        typeof(TimeoutException)
    };
};
```

## Security Features

### Access Control

#### Role-Based Access

```csharp
dashboardOptions.SecurityOptions = securityOptions =>
{
    securityOptions.RequireAuthentication = true;
    securityOptions.RequiredRoles = new[] { "Admin", "JobManager" };
    securityOptions.RequiredClaims = new Dictionary<string, string>
    {
        ["permission"] = "job-management"
    };
};
```

#### Operation Permissions

- **View Only** - Read-only dashboard access
- **Job Management** - Start, stop, and modify jobs
- **Queue Management** - Manage queues and workers
- **System Administration** - Full system control

### Audit Logging

#### Activity Tracking

```csharp
// Audit log entry
public class AuditLogEntry
{
    public string UserId { get; set; }
    public string Action { get; set; }
    public string ResourceId { get; set; }
    public DateTime Timestamp { get; set; }
    public string IpAddress { get; set; }
    public Dictionary<string, object> Details { get; set; }
}
```

#### Compliance Features

- **Action Logging** - Track all user actions
- **Data Retention** - Configurable log retention
- **Export Capabilities** - Export audit logs
- **Compliance Reports** - Generate compliance reports

## API Integration

### REST API

#### Job Management Endpoints

```csharp
// Job management API
[ApiController]
[Route("api/jobs")]
public class JobsController : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetJobs([FromQuery] JobFilter filter)
    {
        var jobs = await _jobService.GetJobsAsync(filter);
        return Ok(jobs);
    }
    
    [HttpPost("{id}/execute")]
    public async Task<IActionResult> ExecuteJob(string id)
    {
        await _jobService.ExecuteJobAsync(id);
        return Ok();
    }
}
```

#### Metrics API

```csharp
// Metrics API endpoints
[HttpGet("api/metrics/summary")]
public async Task<IActionResult> GetMetricsSummary()
{
    var metrics = await _metricsService.GetSummaryAsync();
    return Ok(metrics);
}

[HttpGet("api/metrics/performance")]
public async Task<IActionResult> GetPerformanceMetrics([FromQuery] TimeRange range)
{
    var metrics = await _metricsService.GetPerformanceMetricsAsync(range);
    return Ok(metrics);
}
```

### WebSocket Integration

#### Real-time Data Streaming

```javascript
// WebSocket connection for real-time updates
const ws = new WebSocket('ws://localhost:5000/tickerq/ws');

ws.onmessage = function(event) {
    const data = JSON.parse(event.data);
    
    switch(data.type) {
        case 'job-status-changed':
            updateJobStatus(data.jobId, data.status);
            break;
        case 'metrics-updated':
            updateMetrics(data.metrics);
            break;
    }
};
```

## Next Steps

- [Customization Guide](./customization) - Customize dashboard appearance
- [Authentication Setup](./authentication) - Secure dashboard access
- [API Integration](./api-integration) - Integrate with external systems
