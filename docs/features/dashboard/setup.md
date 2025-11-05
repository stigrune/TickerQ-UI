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
        
        // Enable/disable features
        dashboardOptions.EnableJobCreation = true;
        dashboardOptions.EnableJobDeletion = false;
        dashboardOptions.ShowSystemJobs = true;
        
        // Real-time updates
        dashboardOptions.RefreshInterval = TimeSpan.FromSeconds(5);
        dashboardOptions.EnableSignalR = true;
    });
});
```

## Configuration Options

### Path Configuration

```csharp
dashboardOptions.SetBasePath("/custom/path"); // Default: /tickerq/dashboard
```

### Feature Toggles

```csharp
dashboardOptions.EnableJobCreation = true;    // Allow creating jobs via UI
dashboardOptions.EnableJobDeletion = true;    // Allow deleting jobs via UI
dashboardOptions.EnableJobEditing = true;     // Allow editing job properties
dashboardOptions.ShowSystemJobs = false;      // Hide internal system jobs
dashboardOptions.ShowJobHistory = true;       // Display execution history
```

### Real-time Settings

```csharp
dashboardOptions.RefreshInterval = TimeSpan.FromSeconds(10); // UI refresh rate
dashboardOptions.EnableSignalR = true;                       // Real-time updates
dashboardOptions.MaxHistoryEntries = 1000;                   // History limit
```

## Advanced Configuration

### Custom Middleware

```csharp
app.UseTickerQ(tickerQOptions =>
{
    tickerQOptions.UseDashboard("/admin/scheduler", dashboardConfig =>
    {
        dashboardConfig.EnableAuthentication = true;
        dashboardConfig.RequiredRole = "Administrator";
    });
});
```

### Environment-Specific Settings

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        if (builder.Environment.IsDevelopment())
        {
            dashboardOptions.EnableJobCreation = true;
            dashboardOptions.EnableJobDeletion = true;
            dashboardOptions.ShowSystemJobs = true;
        }
        else
        {
            dashboardOptions.EnableJobCreation = false;
            dashboardOptions.EnableJobDeletion = false;
            dashboardOptions.ShowSystemJobs = false;
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
        dashboardOptions.RequireAuthentication = true;
        dashboardOptions.AuthorizationPolicy = "DashboardAccess";
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

**Performance issues:**
- Adjust `RefreshInterval`
- Limit `MaxHistoryEntries`
- Consider disabling real-time updates for large datasets

## Next Steps

- [Authentication Setup](./authentication) - Secure dashboard access
- [Customization Guide](./customization) - Customize appearance
- [API Integration](./api-integration) - Integrate with external systems
