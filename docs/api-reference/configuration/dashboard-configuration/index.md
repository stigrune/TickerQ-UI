# Dashboard Configuration

Configure the TickerQ Dashboard UI for monitoring and managing jobs.

## Configuration Sections

### [Basic Setup](./basic-setup)
Configure dashboard base path, backend domain, and CORS policies.

### [Authentication](./authentication)
Set up authentication methods for securing the dashboard.

## Quick Example

```csharp
options.AddDashboard(dashboardOptions =>
{
    dashboardOptions.SetBasePath("/admin/tickerq");
    dashboardOptions.WithBasicAuth("admin", "secure-password");
});
```

## AddDashboard

Configure the TickerQ Dashboard UI.

**Method:**
```csharp
TickerOptionsBuilder<TTimeTicker, TCronTicker> AddDashboard(
    Action<DashboardOptionsBuilder> dashboardOptions);
```

## See Also

- [Basic Setup](./basic-setup) - Path, domain, and CORS configuration
- [Authentication](./authentication) - Security options
- [Dashboard Guide](../../../features/dashboard) - Complete dashboard setup
- [Configuration Overview](../index) - All configuration sections

