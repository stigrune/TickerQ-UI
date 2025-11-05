# Dashboard Installation

Install and configure the TickerQ Dashboard package for real-time job monitoring and management.

## Package Installation

### NuGet Package Manager

```bash
dotnet add package TickerQ.Dashboard
```

### Package Manager Console

```powershell
Install-Package TickerQ.Dashboard
```

### PackageReference

```xml
<PackageReference Include="TickerQ.Dashboard" Version="8.0.0-beta" />
```

## Prerequisites

### Required Dependencies

The dashboard package automatically includes:
- **ASP.NET Core** - Web framework
- **SignalR** - Real-time communication
- **Vue.js** - Frontend framework (embedded)

### Framework Requirements

- **.NET 8.0** or later
- **ASP.NET Core** application
- **Modern web browser** (Chrome, Firefox, Safari, Edge)

## Basic Installation

### 1. Add Package

```bash
dotnet add package TickerQ.Dashboard
```

### 2. Configure Services

```csharp
using TickerQ.DependencyInjection;
using TickerQ.Dashboard.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(); // Add dashboard with default settings
});

var app = builder.Build();
app.UseTickerQ(); // Includes dashboard middleware
app.Run();
```

### 3. Access Dashboard

Navigate to: `http://localhost:5000/tickerq/dashboard`

## Verification

### Check Installation

Verify the dashboard is working:

```bash
curl http://localhost:5000/tickerq/dashboard
```

Should return the dashboard HTML page.

### Browser Test

1. Start your application
2. Open browser to `http://localhost:5000/tickerq/dashboard`
3. You should see the TickerQ Dashboard interface

## Troubleshooting

### Common Issues

**Dashboard not loading:**
- Verify `app.UseTickerQ()` is called
- Check that the application is running
- Ensure no firewall blocking the port

**404 Not Found:**
- Confirm the correct URL path
- Check if custom base path is configured
- Verify middleware order in `Program.cs`

**SignalR connection issues:**
- Check browser console for errors
- Verify WebSocket support
- Check proxy/load balancer configuration

## Next Steps

- [Basic Setup](./setup) - Configure dashboard options
- [Authentication](./authentication) - Secure dashboard access
- [Customization](./customization) - Customize appearance
