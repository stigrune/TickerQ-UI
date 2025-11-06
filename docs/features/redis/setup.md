# Redis Setup

Configure Redis for optimal performance with TickerQ distributed job coordination and caching.

## Basic Setup

### Minimal Configuration

```csharp
// Program.cs
using TickerQ.Redis;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddTickerQ(options =>
{
    options.UseRedis("localhost:6379");
});

var app = builder.Build();
app.Run();
```

### Configuration with Options

```csharp
builder.Services.AddTickerQ(options =>
{
    options.UseRedis(redisOptions =>
    {
        redisOptions.ConnectionString = "localhost:6379";
        redisOptions.InstanceName = "TickerQ";
        redisOptions.Database = 0;
    });
});
```

## Connection Configuration

### Connection String Formats

#### Basic Connection

```csharp
// Simple connection string
redisOptions.ConnectionString = "localhost:6379";

// With password
redisOptions.ConnectionString = "localhost:6379,password=mypassword";

// With SSL
redisOptions.ConnectionString = "localhost:6380,ssl=true,password=mypassword";
```

#### Advanced Connection String

```csharp
redisOptions.ConnectionString = "localhost:6379,connectTimeout=5000,syncTimeout=5000,abortConnect=false";
```

### Configuration Options

```csharp
redisOptions.ConfigurationOptions = configOptions =>
{
    // Connection settings
    configOptions.ConnectTimeout = 5000;
    configOptions.SyncTimeout = 5000;
    configOptions.AsyncTimeout = 5000;
    configOptions.ConnectRetry = 3;
    configOptions.ReconnectRetryPolicy = new ExponentialRetry(1000);
    
    // Connection behavior
    configOptions.AbortOnConnectFail = false;
    configOptions.AllowAdmin = false;
    configOptions.KeepAlive = 60;
    
    // SSL/TLS settings
    configOptions.Ssl = false;
    configOptions.SslProtocols = SslProtocols.Tls12;
    
    // Command behavior
    configOptions.CommandMap = CommandMap.Default;
    configOptions.DefaultDatabase = 0;
};
```

## Database Configuration

### Multiple Databases

```csharp
// Use different databases for different purposes
builder.Services.AddTickerQ(options =>
{
    options.UseRedis(redisOptions =>
    {
        redisOptions.ConnectionString = "localhost:6379";
        redisOptions.Database = 1; // Jobs database
    });
});

// Separate caching database
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "Cache";
    options.ConfigurationOptions = configOptions =>
    {
        configOptions.DefaultDatabase = 2; // Cache database
    };
});
```

### Database Isolation

```csharp
public class RedisConfiguration
{
    public const int JobsDatabase = 0;
    public const int CacheDatabase = 1;
    public const int SessionDatabase = 2;
    public const int MetricsDatabase = 3;
}

redisOptions.Database = RedisConfiguration.JobsDatabase;
```

## Key Management

### Key Prefixes

```csharp
redisOptions.KeyPrefix = "tickerq:";

// Results in keys like:
// tickerq:jobs:job-123
// tickerq:queues:default
// tickerq:locks:job-456
```

### Custom Key Patterns

```csharp
redisOptions.KeyPatterns = keyPatterns =>
{
    keyPatterns.JobPattern = "app:jobs:{0}";
    keyPatterns.QueuePattern = "app:queues:{0}";
    keyPatterns.LockPattern = "app:locks:{0}";
    keyPatterns.MetricsPattern = "app:metrics:{0}";
};
```

### Key Expiration

```csharp
redisOptions.ExpirationOptions = expirationOptions =>
{
    expirationOptions.DefaultJobExpiry = TimeSpan.FromDays(7);
    expirationOptions.CompletedJobExpiry = TimeSpan.FromDays(1);
    expirationOptions.FailedJobExpiry = TimeSpan.FromDays(30);
    expirationOptions.LockExpiry = TimeSpan.FromMinutes(5);
};
```

## Serialization Configuration

### JSON Serialization

```csharp
redisOptions.SerializationOptions = serializationOptions =>
{
    serializationOptions.UseSystemTextJson = true;
    serializationOptions.JsonSerializerOptions = new JsonSerializerOptions
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        WriteIndented = false,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
        Converters = { new JsonStringEnumConverter() }
    };
};
```

### Binary Serialization

```csharp
redisOptions.SerializationOptions = serializationOptions =>
{
    serializationOptions.UseMessagePack = true;
    serializationOptions.MessagePackOptions = MessagePackSerializerOptions.Standard
        .WithCompression(MessagePackCompression.Lz4BlockArray);
};
```

### Custom Serialization

```csharp
redisOptions.SerializationOptions = serializationOptions =>
{
    serializationOptions.CustomSerializer = new CustomRedisSerializer();
};

public class CustomRedisSerializer : IRedisSerializer
{
    public byte[] Serialize<T>(T value)
    {
        // Custom serialization logic
        return JsonSerializer.SerializeToUtf8Bytes(value);
    }
    
    public T Deserialize<T>(byte[] data)
    {
        // Custom deserialization logic
        return JsonSerializer.Deserialize<T>(data);
    }
}
```

## Compression Configuration

### Enable Compression

```csharp
redisOptions.EnableCompression = true;
redisOptions.CompressionThreshold = 1024; // Compress if payload > 1KB
redisOptions.CompressionLevel = CompressionLevel.Fastest;
```

### Compression Algorithms

```csharp
redisOptions.CompressionOptions = compressionOptions =>
{
    compressionOptions.Algorithm = CompressionAlgorithm.Gzip;
    compressionOptions.Level = CompressionLevel.Optimal;
    compressionOptions.Threshold = 512; // Compress if > 512 bytes
};
```

## Distributed Locking

### Lock Configuration

```csharp
redisOptions.LockingOptions = lockingOptions =>
{
    lockingOptions.DefaultLockTimeout = TimeSpan.FromMinutes(5);
    lockingOptions.LockRetryDelay = TimeSpan.FromMilliseconds(100);
    lockingOptions.MaxLockRetries = 10;
    lockingOptions.LockExtensionInterval = TimeSpan.FromMinutes(1);
};
```

### Custom Lock Provider

```csharp
builder.Services.AddSingleton<IDistributedLockProvider, RedisDistributedLockProvider>();

public class RedisDistributedLockProvider : IDistributedLockProvider
{
    private readonly IConnectionMultiplexer _redis;
    
    public async Task<IDistributedLock> AcquireLockAsync(string key, TimeSpan timeout)
    {
        var database = _redis.GetDatabase();
        var lockKey = $"lock:{key}";
        var lockValue = Guid.NewGuid().ToString();
        
        var acquired = await database.StringSetAsync(lockKey, lockValue, timeout, When.NotExists);
        
        if (acquired)
        {
            return new RedisDistributedLock(database, lockKey, lockValue, timeout);
        }
        
        throw new LockAcquisitionException($"Failed to acquire lock for key: {key}");
    }
}
```

## High Availability Configuration

### Redis Sentinel

```csharp
redisOptions.ConfigurationOptions = configOptions =>
{
    configOptions.EndPoints.Add("sentinel1:26379");
    configOptions.EndPoints.Add("sentinel2:26379");
    configOptions.EndPoints.Add("sentinel3:26379");
    configOptions.ServiceName = "mymaster";
    configOptions.TieBreaker = "";
    configOptions.CommandMap = CommandMap.Sentinel;
    configOptions.AbortOnConnectFail = false;
};
```

### Redis Cluster

```csharp
redisOptions.ConfigurationOptions = configOptions =>
{
    configOptions.EndPoints.Add("cluster-node1:6379");
    configOptions.EndPoints.Add("cluster-node2:6379");
    configOptions.EndPoints.Add("cluster-node3:6379");
    configOptions.AbortOnConnectFail = false;
    configOptions.ConnectRetry = 3;
};
```

### Failover Configuration

```csharp
redisOptions.FailoverOptions = failoverOptions =>
{
    failoverOptions.EnableFailover = true;
    failoverOptions.FailoverTimeout = TimeSpan.FromSeconds(30);
    failoverOptions.MaxFailoverAttempts = 3;
    failoverOptions.FailoverRetryDelay = TimeSpan.FromSeconds(5);
};
```

## Performance Tuning

### Connection Pool Settings

```csharp
redisOptions.ConfigurationOptions = configOptions =>
{
    configOptions.ConnectTimeout = 5000;
    configOptions.SyncTimeout = 5000;
    configOptions.AsyncTimeout = 5000;
    configOptions.KeepAlive = 60;
    configOptions.ConnectRetry = 3;
    configOptions.ReconnectRetryPolicy = new LinearRetry(1000);
};
```

### Pipeline Configuration

```csharp
redisOptions.PipelineOptions = pipelineOptions =>
{
    pipelineOptions.EnablePipelining = true;
    pipelineOptions.PipelineSize = 100;
    pipelineOptions.PipelineTimeout = TimeSpan.FromSeconds(5);
};
```

### Batch Operations

```csharp
redisOptions.BatchOptions = batchOptions =>
{
    batchOptions.EnableBatching = true;
    batchOptions.BatchSize = 50;
    batchOptions.BatchTimeout = TimeSpan.FromMilliseconds(100);
    batchOptions.MaxConcurrentBatches = 10;
};
```

## Monitoring and Logging

### Redis Monitoring

```csharp
redisOptions.MonitoringOptions = monitoringOptions =>
{
    monitoringOptions.EnableMonitoring = true;
    monitoringOptions.MonitoringInterval = TimeSpan.FromSeconds(30);
    monitoringOptions.CollectPerformanceCounters = true;
    monitoringOptions.CollectConnectionMetrics = true;
};
```

### Logging Configuration

```csharp
// Enable Redis operation logging
builder.Services.AddLogging(logging =>
{
    logging.AddConsole();
    logging.AddFilter("TickerQ.Redis", LogLevel.Information);
    logging.AddFilter("StackExchange.Redis", LogLevel.Warning);
});

redisOptions.LoggingOptions = loggingOptions =>
{
    loggingOptions.LogCommands = true;
    loggingOptions.LogConnectionEvents = true;
    loggingOptions.LogPerformanceMetrics = true;
    loggingOptions.LogLevel = LogLevel.Information;
};
```

### Custom Logging

```csharp
public class RedisLogger : IRedisLogger
{
    private readonly ILogger<RedisLogger> _logger;
    
    public void LogCommand(string command, TimeSpan duration)
    {
        _logger.LogInformation("Redis command {Command} executed in {Duration}ms", 
            command, duration.TotalMilliseconds);
    }
    
    public void LogConnectionEvent(string eventType, string details)
    {
        _logger.LogInformation("Redis connection event: {EventType} - {Details}", 
            eventType, details);
    }
}
```

## Security Configuration

### Authentication

```csharp
redisOptions.ConfigurationOptions = configOptions =>
{
    configOptions.Password = "your-redis-password";
    configOptions.User = "your-redis-username"; // Redis 6.0+
};
```

### SSL/TLS Configuration

```csharp
redisOptions.ConfigurationOptions = configOptions =>
{
    configOptions.Ssl = true;
    configOptions.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;
    configOptions.CertificateValidation += (sender, certificate, chain, errors) =>
    {
        // Custom certificate validation logic
        return errors == SslPolicyErrors.None;
    };
};
```

### Access Control

```csharp
redisOptions.SecurityOptions = securityOptions =>
{
    securityOptions.EnableAccessControl = true;
    securityOptions.AllowedCommands = new[]
    {
        "GET", "SET", "DEL", "EXISTS", "EXPIRE",
        "HGET", "HSET", "HDEL", "HGETALL",
        "LPUSH", "RPOP", "LLEN"
    };
    securityOptions.RestrictedKeys = new[] { "admin:*", "config:*" };
};
```

## Environment-Specific Configuration

### Development Environment

```csharp
if (builder.Environment.IsDevelopment())
{
    redisOptions.ConfigurationOptions = configOptions =>
    {
        configOptions.ConnectTimeout = 10000;
        configOptions.AbortOnConnectFail = false;
    };
    
    redisOptions.LoggingOptions = loggingOptions =>
    {
        loggingOptions.LogCommands = true;
        loggingOptions.LogLevel = LogLevel.Debug;
    };
}
```

### Production Environment

```csharp
if (builder.Environment.IsProduction())
{
    redisOptions.ConfigurationOptions = configOptions =>
    {
        configOptions.ConnectTimeout = 5000;
        configOptions.SyncTimeout = 5000;
        configOptions.ConnectRetry = 5;
        configOptions.ReconnectRetryPolicy = new ExponentialRetry(1000);
    };
    
    redisOptions.EnableCompression = true;
    redisOptions.CompressionThreshold = 512;
    
    redisOptions.MonitoringOptions = monitoringOptions =>
    {
        monitoringOptions.EnableMonitoring = true;
        monitoringOptions.CollectPerformanceCounters = true;
    };
}
```

## Configuration Validation

### Startup Validation

```csharp
builder.Services.AddOptions<RedisOptions>()
    .Configure(options =>
    {
        builder.Configuration.GetSection("TickerQ:Redis").Bind(options);
    })
    .ValidateDataAnnotations()
    .ValidateOnStart();

public class RedisOptions
{
    [Required]
    public string ConnectionString { get; set; }
    
    [Range(0, 15)]
    public int Database { get; set; } = 0;
    
    [Required]
    public string InstanceName { get; set; } = "TickerQ";
}
```

### Health Checks

```csharp
builder.Services.AddHealthChecks()
    .AddRedis(redisOptions.ConnectionString, name: "redis")
    .AddCheck<RedisConfigurationHealthCheck>("redis-config");

public class RedisConfigurationHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        // Validate Redis configuration
        try
        {
            // Check connection, validate settings, etc.
            return Task.FromResult(HealthCheckResult.Healthy("Redis configuration is valid"));
        }
        catch (Exception ex)
        {
            return Task.FromResult(HealthCheckResult.Unhealthy("Redis configuration is invalid", ex));
        }
    }
}
```

## Next Steps

- [Distributed Coordination](./distributed-coordination) - Configure distributed job coordination
