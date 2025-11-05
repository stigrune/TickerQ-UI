# Dashboard

TickerQ.Dashboard provides a real-time web UI for monitoring, managing, and triggering jobs. Built with Vue.js and SignalR, it offers live updates and comprehensive job management.

## Sections

### [Installation](./dashboard/installation)
Install the TickerQ.Dashboard package and configure dependencies.

### [Setup](./dashboard/setup)
Basic dashboard configuration and integration with your application.

### [Authentication](./dashboard/authentication)
Configure authentication and authorization for dashboard access.

### [Customization](./dashboard/customization)
Customize dashboard appearance, themes, and branding.

### [Features](./dashboard/features)
Explore dashboard features including job monitoring, management, and real-time updates.

### [API Integration](./dashboard/api-integration)
Integrate dashboard with external APIs and custom endpoints.

### [Integration](./dashboard/integration)
Integrate dashboard with frameworks, cloud platforms, and infrastructure.

## Quick Start

```csharp
using TickerQ.DependencyInjection;
using TickerQ.Dashboard.DependencyInjection;

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        // Dashboard will be available at /tickerq/dashboard
    });
});

var app = builder.Build();
app.UseTickerQ();
app.Run();
```

Access the dashboard at: `http://localhost:5000/tickerq/dashboard`

## Configuration Options

### Base Path

Customize the dashboard URL path:

```csharp
options.AddDashboard(dashboardOptions =>
{
    dashboardOptions.SetBasePath("/admin/jobs");
});
```

Dashboard will be available at: `/admin/jobs`

### Backend Domain

If dashboard is served from a different domain:

```csharp
dashboardOptions.SetBackendDomain("https://api.example.com");
```

### CORS Policy

Configure CORS for cross-origin requests:

```csharp
dashboardOptions.SetCorsPolicy(corsOptions =>
{
    corsOptions.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
});
```

## Authentication

### No Authentication (Public Dashboard)

```csharp
options.AddDashboard(dashboardOptions =>
{
    // No authentication - dashboard is public
});
```

### Basic Authentication

Simple username/password authentication:

```csharp
options.AddDashboard(dashboardOptions =>
{
    dashboardOptions.WithBasicAuth("admin", "secure-password");
});
```

### Bearer Token Authentication

API key-based authentication:

```csharp
options.AddDashboard(dashboardOptions =>
{
    dashboardOptions.WithApiKey("your-secret-api-key-12345");
});
```

The frontend will prompt for the API key.

### Host Application Authentication

Use your application's existing authentication:

```csharp
options.AddDashboard(dashboardOptions =>
{
    dashboardOptions.WithHostAuthentication();
});
```

This delegates to your application's authentication middleware (e.g., JWT, Cookies, Identity).

## Dashboard Features

### Real-Time Updates

The dashboard uses SignalR for live updates:

- **Active Thread Count**: Real-time worker thread monitoring
- **Next Planned Occurrence**: Updates when next job is scheduled
- **Job Status Changes**: Immediate updates when job status changes
- **Host Status**: Monitor scheduler health
- **Exception Notifications**: Real-time error alerts

### Main Dashboard View

Displays:
- **Active Threads**: Current number of worker threads
- **Next Occurrence**: Next scheduled job execution time
- **Job Status Overview**: Summary of job statuses (Done, Failed, InProgress, etc.)
- **Job Status Past Week**: Chart showing job status distribution
- **Machine Jobs**: Jobs locked by each machine/node
- **Registered Functions**: List of all registered job functions

### TimeTicker Management

View and manage time-based jobs:

- **List View**: All TimeTickers with status, execution time, and metadata
- **Details**: Full job information including request data, retry count, execution history
- **Filter**: Filter by status, function name, date range
- **Actions**: Cancel, delete, or manually trigger jobs

### CronTicker Management

View and manage cron-based jobs:

- **List View**: All CronTickers with expressions and metadata
- **Occurrences**: View all execution occurrences for a CronTicker
- **Edit**: Update cron expression or request data
- **Enable/Disable**: Temporarily disable cron tickers
- **History**: View execution history with status and timing

### Job Creation

Create new jobs directly from the dashboard:

- **TimeTicker Creation**: Schedule one-time jobs
- **CronTicker Creation**: Create recurring jobs with cron expressions
- **Request Data**: Provide JSON request data
- **Retry Configuration**: Set retry counts and intervals

### Job Monitoring

Track job execution:

- **Status Tracking**: Real-time status updates
- **Execution History**: View past executions
- **Retry Information**: See retry attempts and intervals
- **Error Details**: View exception messages and stack traces
- **Timing Metrics**: Execution time and elapsed time tracking

## API Endpoints

The dashboard exposes REST API endpoints:

### Get Next Planned Occurrence

```
GET /api/tickerq/next-occurrence
```

### Get Options

```
GET /api/tickerq/options
```

### Get Machine Jobs

```
GET /api/tickerq/machine-jobs
```

### Get Job Statuses

```
GET /api/tickerq/job-statuses/overall
GET /api/tickerq/job-statuses/past-week
```

### TimeTicker Operations

```
GET    /api/tickerq/timetickers
GET    /api/tickerq/timetickers/{id}
POST   /api/tickerq/timetickers
PUT    /api/tickerq/timetickers/{id}
DELETE /api/tickerq/timetickers/{id}
```

### CronTicker Operations

```
GET    /api/tickerq/crontickers
GET    /api/tickerq/crontickers/{id}
POST   /api/tickerq/crontickers
PUT    /api/tickerq/crontickers/{id}
DELETE /api/tickerq/crontickers/{id}
GET    /api/tickerq/crontickers/{id}/occurrences
```

## SignalR Hub

The dashboard uses SignalR for real-time communication:

### Hub Methods

- `UpdateActiveThreads(threadCount)`: Updates active thread count
- `UpdateNextOccurrence(nextOccurrence)`: Updates next scheduled occurrence
- `UpdateHostStatus(status)`: Updates host scheduler status
- `UpdateHostException(exceptionMessage)`: Broadcasts host exceptions
- `ReceiveThreadsActive(threadCount)`: Client receives thread count updates
- `ReceiveNextOccurrence(nextOccurrence)`: Client receives next occurrence updates

### Connection

The SignalR hub is automatically configured at:
```
{basePath}/hubs/tickerq-notification
```

## Customization

### Styling

The dashboard uses Tailwind CSS. You can customize styles by overriding CSS classes, though this requires rebuilding the frontend assets.

### Extending Endpoints

Extend dashboard endpoints by adding custom API controllers:

```csharp
[ApiController]
[Route("api/tickerq/custom")]
public class CustomDashboardController : ControllerBase
{
    // Your custom endpoints
}
```

## Troubleshooting

### 403 Forbidden Errors

If you get random 403 responses, ensure no filters or middleware are blocking TickerQ endpoints. Check:
- Authentication middleware ordering
- CORS configuration
- Authorization policies
- Route constraints

### SignalR Connection Issues

If real-time updates don't work:
- Verify SignalR is properly configured
- Check authentication for SignalR connections
- Ensure WebSocket support is enabled
- Check firewall/proxy settings

### Dashboard Not Loading

- Verify `UseTickerQ()` is called
- Check base path configuration
- Verify static files middleware is configured
- Check browser console for errors

## Best Practices

### 1. Secure the Dashboard

Always use authentication in production:

```csharp
dashboardOptions.WithHostAuthentication();
```

### 2. Use Appropriate Base Path

Avoid conflicts with your application routes:

```csharp
dashboardOptions.SetBasePath("/admin/tickerq");
```

### 3. Monitor Performance

The dashboard makes frequent API calls. Monitor:
- Database query performance
- API response times
- SignalR connection count

### 4. Limit Access

Restrict dashboard access to authorized personnel only using role-based or policy-based authentication.

## Next Steps

- [Learn About Entity Framework Integration](/features/entity-framework)
- [Configure Redis for Multi-Node](/features/redis)
- [Set Up OpenTelemetry](/features/opentelemetry)
