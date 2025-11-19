# Redis Integration

TickerQ.Caching.StackExchangeRedis provides Redis integration for multi-node coordination (heartbeats + dead-node detection).

## Sections

### [Installation](./redis/installation)
Install the Redis package and configure Redis server connection.

### [Setup](./redis/setup)
Basic Redis configuration and integration with TickerQ.

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

## Configuration Options

### Connection String

```csharp
redisOptions.Configuration = "localhost:6379";
// Or with password
redisOptions.Configuration = "localhost:6379,password=your-password";
// Or full connection string
redisOptions.Configuration = "server=localhost:6379;password=secret;ssl=true";
```

### Instance Name

Prefix for all Redis keys:

```csharp
redisOptions.InstanceName = "tickerq:"; // Default
// Keys will be: tickerq:hb:node1, tickerq:nodes:registry, etc.
```

### Node Heartbeat Interval

How often each node sends heartbeat signals:

```csharp
redisOptions.NodeHeartbeatInterval = TimeSpan.FromMinutes(1); // Default
```

## Multi-Node Coordination

### Node Identification

Each node is identified by a unique name:

```csharp
builder.Services.AddTickerQ(options =>
{
    options.ConfigureScheduler(scheduler =>
    {
        scheduler.NodeIdentifier = "production-server-01";
    });
    // Or use Environment.MachineName (default)
});
```

### Node Heartbeat

TickerQ automatically sends heartbeats to Redis to indicate node health:

- **Heartbeat Key**: `{instanceName}hb:{nodeIdentifier}`
- **TTL**: Heartbeat interval + 20 seconds
- **Format**: JSON with timestamp and node identifier

### Node Registry

All active nodes are registered in Redis:

- **Registry Key**: `{instanceName}nodes:registry`
- **Format**: JSON array of node identifiers
- **TTL**: 30 days sliding expiration

{instanceName}{your-key}
```

## Background Service

TickerQ automatically registers a background service (`NodeHeartBeatBackgroundService`) that:
- Sends periodic heartbeats
- Updates node registry
- Notifies dashboard of node status

This service runs automatically when Redis is configured.

## Use Cases

### 1. Multi-Node Deployment

Deploy TickerQ across multiple servers with Redis coordination:

```csharp
// Server 1
options.ConfigureScheduler(scheduler =>
{
    scheduler.NodeIdentifier = "web-server-01";
});

// Server 2
options.ConfigureScheduler(scheduler =>
{
    scheduler.NodeIdentifier = "web-server-02";
});

// Server 3
options.ConfigureScheduler(scheduler =>
{
    scheduler.NodeIdentifier = "web-server-03";
});
```

All servers share the same Redis instance, and jobs are distributed based on locking mechanisms.

### 2. Health Monitoring

Monitor node health through Redis:

```csharp
public class HealthCheckService
{
    private readonly ITickerQRedisContext _redisContext;
    
    public async Task<NodeHealthStatus> GetNodeHealthAsync()
    {
        var deadNodes = await _redisContext.GetDeadNodesAsync();
        var allNodes = await GetAllRegisteredNodesAsync();
        
        return new NodeHealthStatus
        {
            TotalNodes = allNodes.Count,
            DeadNodes = deadNodes.Length,
            HealthyNodes = allNodes.Count - deadNodes.Length
        };
    }
}
```

## Best Practices

### 1. Unique Node Identifiers

Ensure each node has a unique identifier:

```csharp
// Good: Use machine-specific identifier
options.ConfigureScheduler(scheduler =>
{
    scheduler.NodeIdentifier = Environment.MachineName;
});

// Good: Use deployment-specific identifier
options.ConfigureScheduler(scheduler =>
{
    scheduler.NodeIdentifier = $"{Environment.MachineName}-{Environment.GetEnvironmentVariable("DEPLOYMENT_ID")}";
});

// Bad: Hardcoded value (will conflict in multi-node)
options.ConfigureScheduler(scheduler =>
{
    scheduler.NodeIdentifier = "server";
});
```

### 2. Heartbeat Interval

Balance between freshness and Redis load:

```csharp
// Too frequent: High Redis load
redisOptions.NodeHeartbeatInterval = TimeSpan.FromSeconds(10);

// Good: Balance
redisOptions.NodeHeartbeatInterval = TimeSpan.FromMinutes(1);

// Too infrequent: Slower dead node detection
redisOptions.NodeHeartbeatInterval = TimeSpan.FromMinutes(5);
```

### 3. Redis High Availability

Use Redis Sentinel or Cluster for production:

```csharp
redisOptions.Configuration = "sentinel1:26379,sentinel2:26379,serviceName=mymaster";
```

### 4. Connection Resilience

Configure Redis connection resilience:

```csharp
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.ConfigurationOptions = new ConfigurationOptions
    {
        ConnectRetry = 3,
        ConnectTimeout = 5000,
        SyncTimeout = 5000,
        AbortOnConnectFail = false
    };
});
```

## Troubleshooting

### Nodes Not Appearing in Registry

- Verify Redis connection string
- Check node identifier uniqueness
- Verify heartbeat interval is not too long
- Check Redis key expiration

### Dead Node Detection Not Working

- Verify heartbeat TTL calculation (interval + 20 seconds)
- Check Redis connectivity
- Verify background service is running
- Check application logs for errors

### Cache Not Working

- Verify Redis connection
- Check cache key format
- Verify serialization/deserialization
- Check expiration settings

## Next Steps

- [Learn About Entity Framework](/features/entity-framework)
- [Explore Dashboard Features](/features/dashboard)
- [Set Up OpenTelemetry](/features/opentelemetry)
