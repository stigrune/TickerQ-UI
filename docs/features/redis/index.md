# Redis Integration

TickerQ.Caching.StackExchangeRedis provides Redis integration for multi-node distributed coordination, node heartbeat tracking, and distributed caching.

## Sections

### [Installation](./installation)
Install the Redis package and configure Redis server connection.

### [Setup](./setup)
Basic Redis configuration and integration with TickerQ.

### [Distributed Coordination](./distributed-coordination)
Multi-node coordination, leader election, and distributed locking.

### [Caching](./caching)
Distributed caching strategies and cache management.

### [Performance](./performance)
Redis performance optimization, connection pooling, and monitoring.

### [Troubleshooting](./troubleshooting)
Common Redis issues, debugging, and monitoring.

### [Integration](./integration)
Integrate Redis with cloud services, containers, and infrastructure platforms.

## Quick Start

```csharp
using TickerQ.DependencyInjection;
using TickerQ.Caching.StackExchangeRedis.DependencyInjection;

builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "localhost:6379";
        redisOptions.InstanceName = "tickerq:";
        redisOptions.NodeHeartbeatInterval = TimeSpan.FromMinutes(1);
    });
});
```

## Key Features

- **Distributed Coordination**: Multi-node job coordination and leader election
- **Node Heartbeat**: Automatic node health monitoring and failover
- **Distributed Caching**: Shared cache across multiple application instances
- **Connection Pooling**: Efficient Redis connection management
- **High Availability**: Redis Cluster and Sentinel support

## Use Cases

- **Multi-instance Applications**: Coordinate jobs across multiple app instances
- **Microservices**: Share job state between different services
- **High Availability**: Automatic failover and load distribution
- **Performance**: Reduce database load with distributed caching

## Next Steps

- [Installation Guide](./installation) - Set up Redis integration
- [Configuration](./setup) - Configure Redis options
- [Distributed Features](./distributed-coordination) - Multi-node coordination
