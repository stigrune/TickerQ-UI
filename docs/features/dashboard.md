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

## Dashboard Screenshots

### Overview

![Dashboard overview](/Screenshot_Dashboard.jpeg)

### Time Tickers

![TimeTicker view](/Screenshot_TimeTicker_View.jpeg)

### Cron Tickers

![CronTicker view](/Screenshot_CronTicker_View.jpeg)

### Occurrences

![Cron occurrences view](/Screenshot_Show_Occurrecies.jpeg)

### Create / Update

![Add new TimeTicker](/Screenshot_Add_New_Time_Ticker.jpeg)
![Update CronTicker](/Screenshot_Update_Cron_Ticker.jpeg)

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

### Timezones and Scheduling

- **Scheduler Timezone**: The logical scheduling timezone is configured on the server via `scheduler.SchedulerTimeZone`. All TimeTicker execution times are ultimately stored and evaluated in UTC using this scheduler timezone.
- **UI Timezone Selector**: The dashboard header includes a timezone selector. By default it uses the server scheduler timezone, but you can choose another display timezone (for example, your browser's local timezone).
- **Display vs Scheduling**:
  - All dates and times shown in tables, charts, and details are rendered in the **currently selected UI timezone**.
  - When you create or update a TimeTicker from the dashboard, the time you enter is interpreted in the **scheduler timezone**, and then converted to UTC on the server. This ensures that `22:00` in the scheduler timezone always runs at exactly `22:00` there, regardless of who is viewing the dashboard or from which timezone.
- **Editing Existing TimeTickers**: When you open the edit dialog, the stored UTC execution time is converted back to the selected UI timezone for display. After you change the time and save, the server converts the new time from the scheduler timezone to UTC before scheduling it.

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

The dashboard exposes REST API endpoints under the configured base path.

Assuming `dashboardOptions.SetBasePath("/tickerq/dashboard");`, the base URL will be:

```
http://localhost:5000/tickerq/dashboard
```

All API endpoints are relative to `{basePath}/api`.

### Get Dashboard Options

```
GET {basePath}/api/options
```

### Get Host Status and Next Ticker

```
GET {basePath}/api/ticker-host/status
GET {basePath}/api/ticker-host/next-ticker
```

### Get Machine Jobs

```
GET {basePath}/api/ticker/machine/jobs
```

### Get Job Statuses

```
GET {basePath}/api/ticker/statuses/get
GET {basePath}/api/ticker/statuses/get-last-week
```

### TimeTicker Operations

```
GET    {basePath}/api/time-tickers
GET    {basePath}/api/time-tickers/paginated
GET    {basePath}/api/time-tickers/graph-data-range
GET    {basePath}/api/time-tickers/graph-data
POST   {basePath}/api/time-ticker/add
PUT    {basePath}/api/time-ticker/update
DELETE {basePath}/api/time-ticker/delete
```

### CronTicker Operations

```
GET    {basePath}/api/cron-tickers
GET    {basePath}/api/cron-tickers/paginated
GET    {basePath}/api/cron-tickers/graph-data
GET    {basePath}/api/cron-tickers/graph-data-range
GET    {basePath}/api/cron-ticker-occurrences/{cronTickerId}
GET    {basePath}/api/cron-ticker-occurrences/{cronTickerId}/paginated
GET    {basePath}/api/cron-ticker-occurrences/{cronTickerId}/graph-data
POST   {basePath}/api/cron-ticker/add
PUT    {basePath}/api/cron-ticker/update
POST   {basePath}/api/cron-ticker/run
DELETE {basePath}/api/cron-ticker/delete
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
{basePath}/ticker-notification-hub
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
