# Dashboard Integration

Integrate TickerQ Dashboard with various frameworks, platforms, and third-party services.

## Framework Integrations

### ASP.NET Core MVC

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllersWithViews();
        
        services.AddTickerQ(options =>
        {
            options.AddDashboard(dashboardOptions =>
            {
                dashboardOptions.SetBasePath("/admin/jobs");
                dashboardOptions.RequireAuthentication = true;
            });
        });
    }
    
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        
        app.UseTickerQ(); // Dashboard middleware
        
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllerRoute(
                name: "default",
                pattern: "{controller=Home}/{action=Index}/{id?}");
        });
    }
}
```

### Blazor Server

```csharp
// Program.cs
builder.Services.AddRazorPages();
builder.Services.AddServerSideBlazor();

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.SetBasePath("/dashboard");
        dashboardOptions.EnableSignalR = true; // Works with Blazor SignalR
    });
});

var app = builder.Build();

app.UseRouting();
app.UseTickerQ();

app.MapBlazorHub();
app.MapFallbackToPage("/_Host");
```

### Minimal APIs

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard();
});

var app = builder.Build();

app.UseTickerQ();

// Your minimal API endpoints
app.MapGet("/api/health", () => "Healthy");
app.MapPost("/api/jobs", (CreateJobRequest request) => { /* logic */ });

app.Run();
```

## Cloud Platform Integrations

### Azure App Service

```csharp
// appsettings.json for Azure
{
  "TickerQ": {
    "Dashboard": {
      "BasePath": "/admin/scheduler",
      "RequireHttps": true,
      "AllowedHosts": ["myapp.azurewebsites.net"]
    }
  },
  "ApplicationInsights": {
    "ConnectionString": "InstrumentationKey=..."
  }
}

// Program.cs
builder.Services.AddApplicationInsightsTelemetry();

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.SetBasePath(builder.Configuration["TickerQ:Dashboard:BasePath"]);
        dashboardOptions.RequireHttps = builder.Configuration.GetValue<bool>("TickerQ:Dashboard:RequireHttps");
    });
});
```

### AWS Elastic Beanstalk

```csharp
// Configure for AWS load balancer
builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
});

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.SetBasePath("/jobs");
        dashboardOptions.TrustProxyHeaders = true; // For AWS ALB
    });
});

var app = builder.Build();
app.UseForwardedHeaders();
app.UseTickerQ();
```

### Google Cloud Run

```dockerfile
# Dockerfile for Cloud Run
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY . .

# Cloud Run expects port 8080
ENV ASPNETCORE_URLS=http://*:8080
EXPOSE 8080

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

```csharp
// Configure for Cloud Run
var port = Environment.GetEnvironmentVariable("PORT") ?? "8080";
builder.WebHost.UseUrls($"http://*:{port}");

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.SetBasePath("/scheduler");
        dashboardOptions.EnableCloudRunIntegration = true;
    });
});
```

## Reverse Proxy Integrations

### Nginx

```nginx
# nginx.conf
server {
    listen 80;
    server_name myapp.com;
    
    location /jobs/ {
        proxy_pass http://localhost:5000/tickerq/dashboard/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support for SignalR
        proxy_cache_bypass $http_upgrade;
    }
}
```

```csharp
// Configure app for Nginx proxy
builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    options.KnownProxies.Add(IPAddress.Parse("10.0.0.100")); // Nginx server IP
});

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.SetBasePath("/jobs");
        dashboardOptions.PathBase = "/jobs"; // Match Nginx location
    });
});
```

### Apache HTTP Server

```apache
# .htaccess or virtual host config
ProxyPreserveHost On
ProxyPass /scheduler/ http://localhost:5000/tickerq/dashboard/
ProxyPassReverse /scheduler/ http://localhost:5000/tickerq/dashboard/

# WebSocket support
RewriteEngine On
RewriteCond %{HTTP:Upgrade} websocket [NC]
RewriteCond %{HTTP:Connection} upgrade [NC]
RewriteRule ^/scheduler/?(.*) "ws://localhost:5000/tickerq/dashboard/$1" [P,L]
```

### Traefik

```yaml
# docker-compose.yml with Traefik
version: '3.8'
services:
  app:
    image: myapp:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.tickerq.rule=Host(`myapp.com`) && PathPrefix(`/jobs`)"
      - "traefik.http.routers.tickerq.middlewares=tickerq-stripprefix"
      - "traefik.http.middlewares.tickerq-stripprefix.stripprefix.prefixes=/jobs"
      - "traefik.http.services.tickerq.loadbalancer.server.port=80"
```

## Authentication Provider Integrations

### Auth0

```csharp
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.Authority = "https://your-domain.auth0.com/";
    options.Audience = "your-api-identifier";
});

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.RequireAuthentication = true;
        dashboardOptions.RequiredClaim = "scope";
        dashboardOptions.RequiredClaimValue = "read:jobs";
    });
});
```

### Okta

```csharp
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
    options.ClientId = builder.Configuration["Okta:ClientId"];
    options.ClientSecret = builder.Configuration["Okta:ClientSecret"];
    options.Authority = builder.Configuration["Okta:Authority"];
    options.ResponseType = "code";
    options.Scope.Add("openid");
    options.Scope.Add("profile");
});

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.RequireAuthentication = true;
        dashboardOptions.RequiredRole = "TickerQAdmin";
    });
});
```

## Monitoring Integrations

### Application Insights

```csharp
builder.Services.AddApplicationInsightsTelemetry();

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.EnableTelemetry = true;
        dashboardOptions.TelemetryProvider = "ApplicationInsights";
    });
});
```

### Prometheus

```csharp
builder.Services.AddSingleton<IMetricsLogger, PrometheusMetricsLogger>();

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.EnableMetrics = true;
        dashboardOptions.MetricsEndpoint = "/metrics";
    });
});

// Expose Prometheus metrics
app.MapGet("/metrics", async context =>
{
    var metrics = context.RequestServices.GetRequiredService<IMetricsLogger>();
    await context.Response.WriteAsync(await metrics.GetMetricsAsync());
});
```

### Grafana Integration

```json
{
  "dashboard": {
    "title": "TickerQ Dashboard Metrics",
    "panels": [
      {
        "title": "Active Jobs",
        "type": "stat",
        "targets": [
          {
            "expr": "tickerq_active_jobs_total",
            "legendFormat": "Active Jobs"
          }
        ]
      },
      {
        "title": "Job Execution Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(tickerq_jobs_executed_total[5m])",
            "legendFormat": "Jobs/sec"
          }
        ]
      }
    ]
  }
}
```

## Container Integrations

### Docker

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY . .

# Dashboard assets
COPY --from=build /app/wwwroot ./wwwroot

EXPOSE 80
EXPOSE 443

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  tickerq-app:
    build: .
    ports:
      - "8080:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - TickerQ__Dashboard__BasePath=/admin
    volumes:
      - dashboard-data:/app/data
    depends_on:
      - redis
      - postgres

volumes:
  dashboard-data:
```

### Kubernetes

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tickerq-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tickerq-app
  template:
    metadata:
      labels:
        app: tickerq-app
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 80
        env:
        - name: TickerQ__Dashboard__BasePath
          value: "/jobs"
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
---
apiVersion: v1
kind: Service
metadata:
  name: tickerq-service
spec:
  selector:
    app: tickerq-app
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tickerq-ingress
spec:
  rules:
  - host: jobs.mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tickerq-service
            port:
              number: 80
```

## API Gateway Integrations

### Azure API Management

```xml
<!-- APIM Policy -->
<policies>
    <inbound>
        <base />
        <rewrite-uri template="/tickerq/dashboard/{path}" />
        <set-header name="X-Forwarded-Host" exists-action="override">
            <value>@(context.Request.OriginalUrl.Host)</value>
        </set-header>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
</policies>
```

### AWS API Gateway

```yaml
# serverless.yml
service: tickerq-dashboard

provider:
  name: aws
  runtime: dotnet8
  
functions:
  dashboard:
    handler: MyApp::MyApp.LambdaEntryPoint::FunctionHandlerAsync
    events:
      - http:
          path: /jobs/{proxy+}
          method: ANY
          cors: true
    environment:
      TickerQ__Dashboard__BasePath: /jobs
```

## Next Steps

- [Authentication Setup](./authentication) - Configure security
- [Customization Guide](./customization) - Customize dashboard appearance
- [API Integration](./api-integration) - External API integration
