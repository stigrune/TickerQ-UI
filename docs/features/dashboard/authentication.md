# Dashboard Authentication

Secure your TickerQ Dashboard with authentication and authorization controls.

## Authentication Methods

### JWT Bearer Authentication

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = "https://your-auth-server.com",
            ValidAudience = "tickerq-dashboard",
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("your-secret-key"))
        };
    });

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.RequireAuthentication = true;
    });
});
```

### Cookie Authentication

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.LoginPath = "/Account/Login";
        options.LogoutPath = "/Account/Logout";
        options.ExpireTimeSpan = TimeSpan.FromHours(8);
    });

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.RequireAuthentication = true;
        dashboardOptions.LoginPath = "/Account/Login";
    });
});
```

### Identity Integration

```csharp
builder.Services.AddDefaultIdentity<IdentityUser>(options =>
{
    options.SignIn.RequireConfirmedAccount = false;
})
.AddEntityFrameworkStores<ApplicationDbContext>();

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.RequireAuthentication = true;
        dashboardOptions.RequiredRole = "JobManager";
    });
});
```

## Authorization Policies

### Role-Based Authorization

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("DashboardAccess", policy =>
    {
        policy.RequireRole("Administrator", "JobManager");
    });
    
    options.AddPolicy("JobCreation", policy =>
    {
        policy.RequireRole("Administrator");
    });
    
    options.AddPolicy("JobDeletion", policy =>
    {
        policy.RequireRole("Administrator");
        policy.RequireClaim("permission", "delete-jobs");
    });
});

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.AuthorizationPolicy = "DashboardAccess";
        dashboardOptions.JobCreationPolicy = "JobCreation";
        dashboardOptions.JobDeletionPolicy = "JobDeletion";
    });
});
```

### Custom Authorization Handler

```csharp
public class DashboardAuthorizationHandler : AuthorizationHandler<DashboardRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        DashboardRequirement requirement)
    {
        var user = context.User;
        
        if (user.Identity.IsAuthenticated && 
            user.HasClaim("department", "IT") &&
            user.IsInRole("JobManager"))
        {
            context.Succeed(requirement);
        }
        
        return Task.CompletedTask;
    }
}

public class DashboardRequirement : IAuthorizationRequirement { }

// Registration
builder.Services.AddScoped<IAuthorizationHandler, DashboardAuthorizationHandler>();
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CustomDashboardAccess", policy =>
        policy.Requirements.Add(new DashboardRequirement()));
});
```

## API Key Authentication

### Simple API Key

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.RequireApiKey = true;
        dashboardOptions.ApiKey = "your-secure-api-key";
        dashboardOptions.ApiKeyHeaderName = "X-TickerQ-Key";
    });
});
```

### Custom API Key Validation

```csharp
public class ApiKeyAuthenticationHandler : AuthenticationHandler<ApiKeyAuthenticationSchemeOptions>
{
    public ApiKeyAuthenticationHandler(IOptionsMonitor<ApiKeyAuthenticationSchemeOptions> options,
        ILoggerFactory logger, UrlEncoder encoder, ISystemClock clock)
        : base(options, logger, encoder, clock)
    {
    }

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-API-Key", out var apiKeyHeaderValues))
        {
            return Task.FromResult(AuthenticateResult.NoResult());
        }

        var providedApiKey = apiKeyHeaderValues.FirstOrDefault();
        
        if (IsValidApiKey(providedApiKey))
        {
            var claims = new[]
            {
                new Claim(ClaimTypes.Name, "ApiKeyUser"),
                new Claim(ClaimTypes.Role, "JobManager")
            };
            
            var identity = new ClaimsIdentity(claims, Scheme.Name);
            var principal = new ClaimsPrincipal(identity);
            var ticket = new AuthenticationTicket(principal, Scheme.Name);
            
            return Task.FromResult(AuthenticateResult.Success(ticket));
        }
        
        return Task.FromResult(AuthenticateResult.Fail("Invalid API Key"));
    }
    
    private bool IsValidApiKey(string apiKey)
    {
        // Implement your API key validation logic
        return apiKey == "your-valid-api-key";
    }
}
```

## OAuth 2.0 / OpenID Connect

### Azure AD Integration

```csharp
builder.Services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("AzureAd"));

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.RequireAuthentication = true;
        dashboardOptions.RequiredRole = "TickerQ.Admin";
    });
});
```

### Google OAuth

```csharp
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = GoogleDefaults.AuthenticationScheme;
})
.AddCookie()
.AddGoogle(options =>
{
    options.ClientId = builder.Configuration["Google:ClientId"];
    options.ClientSecret = builder.Configuration["Google:ClientSecret"];
});

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.RequireAuthentication = true;
        dashboardOptions.AllowedDomains = new[] { "yourcompany.com" };
    });
});
```

## Security Best Practices

### HTTPS Enforcement

```csharp
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
});

app.UseHsts();
app.UseHttpsRedirection();
```

### Content Security Policy

```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("Content-Security-Policy", 
        "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';");
    await next();
});
```

### Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("DashboardPolicy", limiterOptions =>
    {
        limiterOptions.PermitLimit = 100;
        limiterOptions.Window = TimeSpan.FromMinutes(1);
    });
});

app.UseRateLimiter();

builder.Services.AddTickerQ(options =>
{
    options.AddDashboard(dashboardOptions =>
    {
        dashboardOptions.RateLimitPolicy = "DashboardPolicy";
    });
});
```

## Environment-Specific Configuration

### Development

```csharp
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddTickerQ(options =>
    {
        options.AddDashboard(dashboardOptions =>
        {
            dashboardOptions.RequireAuthentication = false; // Disable for dev
            dashboardOptions.EnableJobCreation = true;
            dashboardOptions.EnableJobDeletion = true;
        });
    });
}
```

### Production

```csharp
if (builder.Environment.IsProduction())
{
    builder.Services.AddTickerQ(options =>
    {
        options.AddDashboard(dashboardOptions =>
        {
            dashboardOptions.RequireAuthentication = true;
            dashboardOptions.RequiredRole = "Administrator";
            dashboardOptions.EnableJobCreation = false;
            dashboardOptions.EnableJobDeletion = false;
            dashboardOptions.RequireHttps = true;
        });
    });
}
```

## Troubleshooting

### Common Authentication Issues

**401 Unauthorized:**
- Check authentication configuration
- Verify token/cookie validity
- Ensure proper claims/roles

**403 Forbidden:**
- Check authorization policies
- Verify user roles/claims
- Review policy requirements

**SignalR Authentication:**
- Configure SignalR authentication
- Check WebSocket authentication
- Verify CORS for authenticated requests

## Next Steps

- [Dashboard Customization](./customization) - Customize UI appearance
- [API Integration](./api-integration) - Integrate with external systems
- [Features Overview](./features) - Explore dashboard capabilities
