# Dashboard Customization

Customize the TickerQ Dashboard appearance, themes, and branding to match your application's design.

## Theme Customization

### Custom CSS

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.CustomCssPath = "/css/tickerq-custom.css";
        dashboardOptions.EnableCustomStyling = true;
    });
});
```

### Custom Theme Colors

```css
/* Custom CSS file */
:root {
    --tickerq-primary: #3b82f6;
    --tickerq-secondary: #64748b;
    --tickerq-success: #10b981;
    --tickerq-warning: #f59e0b;
    --tickerq-error: #ef4444;
    --tickerq-background: #ffffff;
    --tickerq-surface: #f8fafc;
}

.dark {
    --tickerq-primary: #60a5fa;
    --tickerq-secondary: #94a3b8;
    --tickerq-background: #0f172a;
    --tickerq-surface: #1e293b;
}
```

## Branding

### Custom Logo

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.BrandingOptions = brandingOptions =>
        {
            brandingOptions.LogoUrl = "/images/company-logo.png";
            brandingOptions.CompanyName = "Your Company";
            brandingOptions.ShowPoweredBy = false;
        };
    });
});
```

### Custom Title and Favicon

```csharp
dashboardOptions.BrandingOptions = brandingOptions =>
{
    brandingOptions.PageTitle = "Job Scheduler - Your Company";
    brandingOptions.FaviconUrl = "/favicon.ico";
    brandingOptions.MetaDescription = "Internal job scheduling dashboard";
};
```

## Layout Customization

### Custom Header

```csharp
dashboardOptions.LayoutOptions = layoutOptions =>
{
    layoutOptions.ShowHeader = true;
    layoutOptions.HeaderHeight = "60px";
    layoutOptions.CustomHeaderContent = "<div class='custom-header'>Custom Content</div>";
};
```

### Sidebar Configuration

```csharp
dashboardOptions.LayoutOptions = layoutOptions =>
{
    layoutOptions.SidebarWidth = "280px";
    layoutOptions.CollapsibleSidebar = true;
    layoutOptions.ShowNavigationBreadcrumbs = true;
};
```

## Component Customization

### Job Cards

```css
.job-card {
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    transition: transform 0.2s ease;
}

.job-card:hover {
    transform: translateY(-2px);
}

.job-card.success {
    border-left: 4px solid var(--tickerq-success);
}

.job-card.error {
    border-left: 4px solid var(--tickerq-error);
}
```

### Status Indicators

```css
.status-indicator {
    display: inline-flex;
    align-items: center;
    padding: 4px 8px;
    border-radius: 4px;
    font-size: 0.75rem;
    font-weight: 600;
}

.status-indicator.running {
    background-color: #dbeafe;
    color: #1e40af;
}

.status-indicator.completed {
    background-color: #d1fae5;
    color: #065f46;
}

.status-indicator.failed {
    background-color: #fee2e2;
    color: #991b1b;
}
```

## Dark Mode Support

### Automatic Theme Detection

```csharp
dashboardOptions.ThemeOptions = themeOptions =>
{
    themeOptions.EnableDarkMode = true;
    themeOptions.AutoDetectTheme = true;
    themeOptions.DefaultTheme = "light";
};
```

### Custom Dark Mode Styles

```css
@media (prefers-color-scheme: dark) {
    .dashboard-container {
        background-color: var(--tickerq-background);
        color: #f1f5f9;
    }
    
    .job-card {
        background-color: var(--tickerq-surface);
        border-color: #374151;
    }
}
```

## Responsive Design

### Mobile Optimization

```css
@media (max-width: 768px) {
    .dashboard-grid {
        grid-template-columns: 1fr;
        gap: 1rem;
    }
    
    .sidebar {
        transform: translateX(-100%);
        transition: transform 0.3s ease;
    }
    
    .sidebar.open {
        transform: translateX(0);
    }
}
```

### Tablet Layout

```css
@media (min-width: 769px) and (max-width: 1024px) {
    .dashboard-grid {
        grid-template-columns: repeat(2, 1fr);
    }
    
    .sidebar {
        width: 240px;
    }
}
```

## Custom JavaScript

### Dashboard Extensions

```javascript
// Custom dashboard extensions
window.TickerQDashboard = {
    onJobComplete: function(job) {
        // Custom notification
        showNotification(`Job ${job.name} completed successfully`);
    },
    
    onJobFailed: function(job, error) {
        // Custom error handling
        showErrorDialog(`Job ${job.name} failed: ${error.message}`);
    },
    
    customActions: {
        'restart-job': function(jobId) {
            // Custom job restart logic
            fetch(`/api/jobs/${jobId}/restart`, { method: 'POST' });
        }
    }
};
```

### Real-time Updates

```javascript
// Custom SignalR event handlers
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/tickerq/hub")
    .build();

connection.on("JobStatusChanged", function (jobId, status) {
    updateJobCard(jobId, status);
    
    if (status === 'Completed') {
        showSuccessAnimation(jobId);
    }
});
```

## Configuration Examples

### Corporate Theme

```csharp
dashboardOptions.BrandingOptions = brandingOptions =>
{
    brandingOptions.LogoUrl = "/images/corporate-logo.png";
    brandingOptions.CompanyName = "Enterprise Corp";
    brandingOptions.PrimaryColor = "#1f2937";
    brandingOptions.SecondaryColor = "#6b7280";
    brandingOptions.ShowPoweredBy = false;
};

dashboardOptions.LayoutOptions = layoutOptions =>
{
    layoutOptions.ShowHeader = true;
    layoutOptions.HeaderBackgroundColor = "#1f2937";
    layoutOptions.SidebarBackgroundColor = "#374151";
};
```

### Minimal Theme

```csharp
dashboardOptions.ThemeOptions = themeOptions =>
{
    themeOptions.MinimalMode = true;
    themeOptions.HideNavigationLabels = true;
    themeOptions.CompactLayout = true;
};

dashboardOptions.LayoutOptions = layoutOptions =>
{
    layoutOptions.SidebarWidth = "60px";
    layoutOptions.ShowBreadcrumbs = false;
    layoutOptions.ShowFooter = false;
};
```

## Advanced Customization

### Custom Components

```csharp
dashboardOptions.ComponentOptions = componentOptions =>
{
    componentOptions.CustomComponents = new Dictionary<string, string>
    {
        ["job-card"] = "/components/CustomJobCard.vue",
        ["status-indicator"] = "/components/CustomStatusIndicator.vue"
    };
};
```

### Plugin System

```csharp
dashboardOptions.PluginOptions = pluginOptions =>
{
    pluginOptions.EnablePlugins = true;
    pluginOptions.PluginDirectory = "/dashboard-plugins";
    pluginOptions.LoadedPlugins = new[]
    {
        "CustomMetrics",
        "JobAnalytics",
        "NotificationCenter"
    };
};
```

## Next Steps

- [Authentication Setup](./authentication) - Secure dashboard access
- [Integration Guide](./integration) - Framework and platform integration
- [Features Overview](./features) - Explore dashboard capabilities
