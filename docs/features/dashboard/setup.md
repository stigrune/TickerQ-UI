# Dashboard Setup

Configure the TickerQ Dashboard with custom options, authentication, and advanced features.

## Basic Configuration

### Default Setup

```csharp
using TickerQ.DependencyInjection;
using TickerQ.Dashboard.DependencyInjection;

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(); // Uses default configuration
});
```

### Custom Configuration

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        // Custom base path
        dashboardOptions.SetBasePath("/admin/jobs");
    });
});
```

## Configuration Options

### Path Configuration

```csharp
dashboardOptions.SetBasePath("/custom/path"); // Default: /tickerq/dashboard
```

> Note  
> Older versions of this documentation mentioned feature toggles such as
> `EnableJobCreation`, `EnableJobDeletion`, `ShowSystemJobs`, `RefreshInterval`,
> `EnableSignalR`, etc. These properties do **not** exist on `DashboardOptionsBuilder`
> and have been removed. The current dashboard behavior is controlled primarily
> by authentication (`WithNoAuth`, `WithBasicAuth`, `WithApiKey`, `WithHostAuthentication`)
> and by the base path / backend domain configuration.

## Advanced Configuration

### Custom Middleware

```csharp
// Example of inserting custom middleware around the dashboard branch
app.UseTickerQ();
// Any additional middleware can be configured in your normal ASP.NET Core pipeline.
```

### Environment-Specific Settings

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        if (builder.Environment.IsDevelopment())
        {
            dashboardOptions.SetBasePath("/tickerq/dashboard");
            dashboardOptions.WithNoAuth();
        }
        else
        {
            dashboardOptions.SetBasePath("/tickerq/dashboard");
            dashboardOptions.WithHostAuthentication();
        }
    });
});
```

## Integration with ASP.NET Core

### With Authentication

```csharp
builder.Services.AddAuthentication()
    .AddJwtBearer();

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("DashboardAccess", policy =>
        policy.RequireRole("Admin", "JobManager"));
});

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.SetBasePath("/tickerq/dashboard");
        dashboardOptions.WithHostAuthentication();
        // Protect the dashboard with the "DashboardAccess" policy using your normal middleware.
    });
});
```

### With CORS

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("DashboardCors", policy =>
    {
        policy.WithOrigins("https://admin.myapp.com")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

app.UseCors("DashboardCors");
app.UseTickerQ();
```

## Docker Configuration

### Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY . .

# Expose dashboard port
EXPOSE 80

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Docker Compose

```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - TickerQ__Dashboard__BasePath=/jobs
    depends_on:
      - database
```

## Health Checks

### Dashboard Health Check

```csharp
builder.Services.AddHealthChecks()
    .AddTickerQDashboard();

app.MapHealthChecks("/health");
```

### Custom Health Check

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("dashboard", () =>
    {
        // Custom dashboard health logic
        return HealthCheckResult.Healthy("Dashboard is running");
    });
```

## Troubleshooting

### Common Configuration Issues

**Dashboard not accessible:**
- Check `SetBasePath()` configuration
- Verify middleware order
- Ensure authentication is properly configured

**SignalR not working:**
- Check WebSocket support
- Verify CORS configuration
- Check proxy/load balancer settings

## Next Steps

- [Authentication Setup](./authentication) - Secure dashboard access
- [Customization Guide](./customization) - Customize appearance
- [API Integration](./api-integration) - Integrate with external systems
