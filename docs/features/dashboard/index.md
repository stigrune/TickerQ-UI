# Dashboard

TickerQ.Dashboard provides a real-time web UI for monitoring, managing, and triggering jobs. Built with Vue.js and SignalR, it offers live updates and comprehensive job management.

## Sections

### [Installation](./installation)
Install the TickerQ.Dashboard package and configure dependencies.

### [Setup](./setup)
Basic dashboard configuration and integration with your application.

### [Authentication](./authentication)
Configure authentication and authorization for dashboard access.

### [Customization](./customization)
Customize dashboard appearance, themes, and branding.

### [Features](./features)
Explore dashboard features including job monitoring, management, and real-time updates.

### [API Integration](./api-integration)
Integrate dashboard with external APIs and custom endpoints.

### [Integration](./integration)
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

## Next Steps

- [Installation Guide](./installation) - Get started with dashboard setup
- [Authentication Setup](./authentication) - Secure your dashboard
- [Customization Options](./customization) - Brand and customize the UI
