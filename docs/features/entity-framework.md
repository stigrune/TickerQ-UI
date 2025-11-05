# Entity Framework Core Integration

TickerQ.EntityFrameworkCore provides persistence support for TickerQ jobs, allowing you to store job state, execution history, and manage jobs through your database.

## Sections

### [Installation](./entity-framework/installation)
Install the TickerQ.EntityFrameworkCore package and database providers.

### [Setup](./entity-framework/setup/index)
Configure Entity Framework Core integration. Choose between built-in TickerQDbContext, application DbContext, or custom entities.

### [Migrations](./entity-framework/migrations)
Create and manage database migrations for TickerQ entities.

### [Database Operations](./entity-framework/database-operations)
Query, update, and manage TickerQ jobs through Entity Framework Core.

### [Performance](./entity-framework/performance)
DbContext pooling, indexes, transactions, and performance optimization.

### [Best Practices](./entity-framework/best-practices)
Recommended practices for production deployments.

## Quick Setup

```csharp
using TickerQ.EntityFrameworkCore.DbContextFactory;

builder.Services.AddTickerQ(options =>
{
    options.AddOperationalStore(efOptions =>
    {
        efOptions.UseTickerQDbContext<TickerQDbContext>(optionsBuilder =>
        {
            optionsBuilder.UseNpgsql(connectionString);
        });
    });
});
```

## See Also

- [Configuration Reference](/api-reference/configuration/entity-framework-configuration) - Complete API reference
- [Entity Reference](/api-reference/entities/index) - Entity properties and types
- [Dashboard](/features/dashboard) - Visual job management
- [Redis Integration](/features/redis) - Multi-node coordination


