# API Integration

Learn how to integrate the TickerQ Dashboard with your application and other systems.

> **Note**  
> The TickerQ Dashboard is primarily a UI on top of TickerQ’s core APIs (`ITimeTickerManager`, `ICronTickerManager`, EF Core providers, etc.).  
> The dashboard’s HTTP endpoints are designed for the dashboard UI itself and are **not a separate, fully-public REST API surface**.

## Where the Dashboard API Lives

When you configure the dashboard:

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.SetBasePath("/tickerq/dashboard");
    });
});
```

the dashboard will be available at:

```text
{basePath} = /tickerq/dashboard
```

The dashboard API endpoints are then available under:

```text
{basePath}/api/...
```

For example:

- `GET {basePath}/api/options` – dashboard + scheduler info  
- `GET {basePath}/api/time-tickers` – list of time tickers  
- `GET {basePath}/api/cron-tickers` – list of cron tickers  
- `GET {basePath}/api/ticker/statuses/get` – overall job status summary  
- `GET {basePath}/api/ticker/statuses/get-last-week` – last-week status summary  
- `GET {basePath}/api/ticker/machine/jobs` – jobs per machine/node

> See [Dashboard](../dashboard.md#api-endpoints) for a more complete list mapped from the backend.

## Recommended Integration Approach

For programmatic integration you should normally:

- Use the core managers:
  - `ITimeTickerManager<TTimeTicker>`
  - `ICronTickerManager<TCronTicker>`
- Or use the EF Core provider to query entities:
  - `TimeTickerEntity`
  - `CronTickerEntity`
  - `CronTickerOccurrenceEntity`

This gives you:

- Strongly-typed access
- Full control over authorization and validation
- Stable APIs that evolve with TickerQ itself

Example – scheduling via managers:

```csharp
public class CleanupJobs(ICronTickerManager<CronTickerEntity> cronManager)
{
    public async Task ScheduleCleanupAsync()
    {
        await cronManager.AddAsync(new CronTickerEntity
        {
            Function = "CleanupLogs",
            Expression = "0 0 */6 * * *"
        });
    }
}
```

## Calling Dashboard Endpoints

If you still want to call the dashboard’s own endpoints (e.g., from a custom admin tool), use the same base path configuration:

```csharp
var dashboardBasePath = "/tickerq/dashboard";
var httpClient = new HttpClient
{
    BaseAddress = new Uri("https://your-app.example.com")
};

var response = await httpClient.GetAsync($"{dashboardBasePath}/api/options");
response.EnsureSuccessStatusCode();
```

Make sure your authentication mode matches how the dashboard is configured:

- `WithBasicAuth` – send `Authorization: Basic ...`
- `WithApiKey` – send `Authorization: Bearer {apiKey}`
- `WithHostAuthentication` – reuse your app’s existing auth (JWT, cookies, etc.)

## Real-time Updates

The dashboard uses SignalR for real-time updates (host status, next occurrence, active threads).

The hub is available at:

```text
{basePath}/ticker-notification-hub
```

If you connect from a custom client, use the same URL and authentication as the dashboard UI.

## When to Avoid the Dashboard API

Prefer using the core TickerQ APIs instead of reverse-engineering or tightly coupling to dashboard endpoints when:

- Building backend integrations
- Implementing custom workflows
- Exporting metrics or job data

The core APIs (`ITimeTickerManager`, `ICronTickerManager`, EF Core entities, OpenTelemetry integration, etc.) are the stable surface intended for automation and integrations.

