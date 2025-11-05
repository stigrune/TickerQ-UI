# Redis Configuration

Configure Redis for distributed coordination and multi-node support.

## AddStackExchangeRedis

Configure Redis for distributed coordination.

**Method:**
```csharp
TickerOptionsBuilder<TTimeTicker, TCronTicker> AddStackExchangeRedis(
    Action<TickerQRedisOptionBuilder> setupAction);
```

**Example:**
```csharp
options.AddStackExchangeRedis(redisOptions =>
{
    redisOptions.Configuration = "localhost:6379";
    redisOptions.InstanceName = "tickerq:";
    redisOptions.NodeHeartbeatInterval = TimeSpan.FromMinutes(1);
});
```

## TickerQRedisOptionBuilder Options

### Configuration

Redis connection string.

**Type:** `string`

```csharp
redisOptions.Configuration = "localhost:6379";
// Or with password
redisOptions.Configuration = "localhost:6379,password=secret";
```

### InstanceName

Key prefix for all Redis keys.

**Type:** `string`  
**Default:** `"tickerq:"`

```csharp
redisOptions.InstanceName = "myapp:tickerq:";
```

### NodeHeartbeatInterval

How often nodes send heartbeat signals.

**Type:** `TimeSpan`  
**Default:** `TimeSpan.FromMinutes(1)`

```csharp
redisOptions.NodeHeartbeatInterval = TimeSpan.FromMinutes(1);
```

**Notes:**
- Heartbeat TTL = Interval + 20 seconds
- Lower intervals = faster dead node detection but more Redis load

## Multi-Node Setup

For multi-node deployments, each node should have a unique `NodeIdentifier`:

```csharp
options.ConfigureScheduler(scheduler =>
{
    scheduler.NodeIdentifier = "web-server-01"; // Unique per node
});

options.AddStackExchangeRedis(redisOptions =>
{
    redisOptions.Configuration = "redis-cluster:6379";
});
```

## See Also

- [Redis Integration](../../features/redis) - Complete Redis setup guide
- [Core Configuration](./core-configuration) - Scheduler options
- [Configuration Overview](./index) - All configuration sections

