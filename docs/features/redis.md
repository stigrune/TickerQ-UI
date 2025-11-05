# Redis Integration

TickerQ.Caching.StackExchangeRedis provides Redis integration for multi-node distributed coordination, node heartbeat tracking, and distributed caching.

## Sections

### [Installation](./redis/installation)
Install the Redis package and configure Redis server connection.

### [Setup](./redis/setup)
Basic Redis configuration and integration with TickerQ.

### [Distributed Coordination](./redis/distributed-coordination)
Multi-node coordination, leader election, and distributed locking.

### [Caching](./redis/caching)
Distributed caching strategies and cache management.

### [Performance](./redis/performance)
Redis performance optimization, connection pooling, and monitoring.

### [Troubleshooting](./redis/troubleshooting)
Common Redis issues, debugging, and monitoring.

### [Integration](./redis/integration)
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

### Dead Node Detection

TickerQ can detect dead nodes (nodes that stopped sending heartbeats):

```csharp
// Access Redis context
var redisContext = serviceProvider.GetRequiredService<ITickerQRedisContext>();

// Get dead nodes
var deadNodes = await redisContext.GetDeadNodesAsync();
```

## Distributed Caching

### Get or Set Array

TickerQ provides a caching utility for arrays:

```csharp
var redisContext = serviceProvider.GetRequiredService<ITickerQRedisContext>();

var cachedData = await redisContext.GetOrSetArrayAsync<MyData>(
    cacheKey: "my-cache-key",
    factory: async (ct) =>
    {
        // Fetch data if not in cache
        return await FetchDataFromDatabaseAsync(ct);
    },
    expiration: TimeSpan.FromMinutes(5),
    cancellationToken: cancellationToken
);
```

### Cache Key Format

Cache keys follow the pattern:
```
{instanceName}{your-key}
```

## Background Service

TickerQ automatically registers a background service (`NodeHeartBeatBackgroundService`) that:
- Sends periodic heartbeats
- Updates node registry
- Notifies dashboard of node status

This service runs automatically when Redis is configured.

## Redis Key Structure

### Heartbeat Keys
```
{instanceName}hb:{nodeIdentifier}
```
Stores heartbeat payload with timestamp.

### Registry Key
```
{instanceName}nodes:registry
```
Stores set of all registered node identifiers.

### Custom Cache Keys
```
{instanceName}{your-custom-key}
```
Your application-specific cache keys.

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

### 3. Cache Warmup

Pre-populate cache on application startup:

```csharp
[TickerFunction("WarmupCache", cronExpression: "0 0 * * * *")]
public async Task WarmupCache(
    TickerFunctionContext context,
    CancellationToken cancellationToken)
{
    var redisContext = context.ServiceScope.ServiceProvider
        .GetRequiredService<ITickerQRedisContext>();
    
    var data = await redisContext.GetOrSetArrayAsync<PopularData>(
        "popular-items",
        async (ct) => await FetchPopularItemsAsync(ct),
        expiration: TimeSpan.FromHours(1),
        cancellationToken
    );
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

