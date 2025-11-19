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
        dashboardOptions.SetBasePath("/tickerq/dashboard");
        dashboardOptions.WithHostAuthentication();
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
        dashboardOptions.SetBasePath("/tickerq/dashboard");
        dashboardOptions.WithHostAuthentication();
        // Use your app's normal cookie login path.
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
        dashboardOptions.SetBasePath("/tickerq/dashboard");
        dashboardOptions.WithHostAuthentication();
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
        dashboardOptions.SetBasePath("/tickerq/dashboard");
        dashboardOptions.WithHostAuthentication();
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
        dashboardOptions.SetBasePath("/tickerq/dashboard");
        dashboardOptions.WithApiKey("your-secure-api-key");
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
        dashboardOptions.SetBasePath("/tickerq/dashboard");
        dashboardOptions.WithHostAuthentication();
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
        dashboardOptions.SetBasePath("/tickerq/dashboard");
        dashboardOptions.WithHostAuthentication();
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
            dashboardOptions.SetBasePath("/tickerq/dashboard");
            dashboardOptions.WithNoAuth(); // Disable auth for dev
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
            dashboardOptions.SetBasePath("/tickerq/dashboard");
            dashboardOptions.WithHostAuthentication();
            // Apply role-based authorization using your normal ASP.NET Core policies.
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
