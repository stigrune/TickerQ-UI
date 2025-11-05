# Entity Framework Setup

Configure Entity Framework Core persistence for TickerQ. Choose the setup approach that best fits your application architecture.

## Setup Options

### [Built-in TickerQDbContext](./built-in-dbcontext)
Use the lightweight, optimized `TickerQDbContext` designed specifically for TickerQ. Recommended for most scenarios.

### [Application DbContext](./application-dbcontext)
Integrate TickerQ entities into your existing application DbContext for shared database connections and unified migrations.

### [Custom Entity Types](./custom-entities)
Create custom entity types with additional properties, then configure TickerQ to use them.

## Quick Comparison

| Feature | TickerQDbContext | Application DbContext | Custom Entities |
|---------|------------------|----------------------|-----------------|
| **Setup Complexity** | Simple | Moderate | More complex |
| **Connection Sharing** | Separate | Shared pool | Configurable |
| **Entity Mixing** | Isolated | Mixed | Custom |
| **Migrations** | Separate | Combined | Separate |
| **Recommended For** | Most scenarios | When integration needed | Multi-tenancy, custom properties |

## Common Setup Pattern

Most applications should use the built-in `TickerQDbContext`:

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
        efOptions.SetDbContextPoolSize(34);
    });
});
```

## See Also

- [Built-in TickerQDbContext](./built-in-dbcontext) - Recommended setup
- [Application DbContext](./application-dbcontext) - Integration with existing DbContext
- [Custom Entities](./custom-entities) - Custom entity types
- [Migrations](../migrations) - Database migration guide

