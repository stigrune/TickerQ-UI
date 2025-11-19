# Dashboard Customization

TickerQ ships with a prebuilt Vue dashboard that is embedded into your ASP.NET Core application.  
Customization from your app focuses on backend configuration: routing, CORS, middleware, and response formatting.  
For visual or component-level changes you must modify the `TickerQ.Dashboard` project directly; the options below cover everything exposed via `DashboardOptionsBuilder`.

## Base Path & Backend Domain

Set the URL where the dashboard is hosted and, if needed, override the backend API domain that the SPA uses when making requests.

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.SetBasePath("/admin/jobs");            // default: /tickerq/dashboard
        dashboardOptions.SetBackendDomain("https://api.example.com"); // optional
    });
});
```

## CORS Policy

If the dashboard is served from a different origin than the backend APIs, configure a named CORS policy for the dashboard endpoints and hub.

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.SetCorsPolicy(cors =>
        {
            cors.WithOrigins("https://dashboard.example.com")
                .AllowAnyHeader()
                .AllowAnyMethod()
                .AllowCredentials();
        });
    });
});
```

## Middleware Hooks

`DashboardOptionsBuilder` exposes three middleware hooks that let you inject behavior around the dashboard branch:

| Property | Description |
| --- | --- |
| `PreDashboardMiddleware` | Runs before static files and routing (good place to add security headers). |
| `CustomMiddleware` | Runs after auth middleware but before endpoint mapping. |
| `PostDashboardMiddleware` | Runs after the endpoints are registered. |

```csharp
options.AddDashboard(dashboardOptions =>
{
    dashboardOptions.PreDashboardMiddleware = app =>
    {
        app.Use(async (context, next) =>
        {
            context.Response.Headers["X-Frame-Options"] = "DENY";
            await next();
        });
    };

    dashboardOptions.CustomMiddleware = app =>
    {
        app.Use(async (context, next) =>
        {
            Console.WriteLine($"Dashboard request: {context.Request.Path}");
            await next();
        });
    };

    dashboardOptions.PostDashboardMiddleware = app =>
    {
        app.Use(async (context, next) =>
        {
            await next();
            // telemetry or logging after dashboard endpoints
        });
    };
});
```

## Dashboard JSON Options

Dashboard APIs use their own `JsonSerializerOptions` so your job payload serialization settings do not interfere.  
If you need to customize JSON behavior (e.g., naming policies or converters) call `ConfigureDashboardJsonOptions`.

```csharp
options.AddDashboard(dashboardOptions =>
{
    dashboardOptions.ConfigureDashboardJsonOptions(json =>
    {
        json.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        json.WriteIndented = false;
        json.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
    });
});
```

## Authentication

Use the built-in helpers to control access:

```csharp
options.AddDashboard(dashboardOptions =>
{
    dashboardOptions.WithBasicAuth("admin", "password");
    // or dashboardOptions.WithApiKey("secret");
    // or dashboardOptions.WithHostAuthentication();
    // or dashboardOptions.WithCustomAuth(token => token == "my-token");
});
```

You can chain `WithSessionTimeout(minutes)` to control the dashboard session lifetime.

## Summary

- **Base Path & Backend Domain**: `SetBasePath`, `SetBackendDomain`
- **CORS**: `SetCorsPolicy`
- **Auth**: `WithNoAuth`, `WithBasicAuth`, `WithApiKey`, `WithHostAuthentication`, `WithCustomAuth`, `WithSessionTimeout`
- **Middleware Hooks**: `PreDashboardMiddleware`, `CustomMiddleware`, `PostDashboardMiddleware`
- **JSON Options**: `ConfigureDashboardJsonOptions`

These APIs represent the full surface area available from the backend. For any other customization you must fork or modify the `TickerQ.Dashboard` project itself.
