# Redis Integration

Integrate TickerQ Redis caching with various Redis deployments, cloud services, and infrastructure setups.

## Redis Deployment Integrations

### Redis Standalone

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "localhost:6379";
        redisOptions.InstanceName = "tickerq:";
        redisOptions.Database = 0;
        redisOptions.ConnectTimeout = TimeSpan.FromSeconds(5);
        redisOptions.SyncTimeout = TimeSpan.FromSeconds(5);
    });
});
```

### Redis Cluster

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "redis-node1:6379,redis-node2:6379,redis-node3:6379";
        redisOptions.InstanceName = "tickerq:cluster:";
        redisOptions.EnableClusterMode = true;
        redisOptions.ClusterConfiguration = clusterOptions =>
        {
            clusterOptions.MaxRedirects = 3;
            clusterOptions.RetryPolicy = new ExponentialRetry(TimeSpan.FromMilliseconds(100));
        };
    });
});
```

### Redis Sentinel

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "sentinel1:26379,sentinel2:26379,sentinel3:26379";
        redisOptions.ServiceName = "mymaster";
        redisOptions.InstanceName = "tickerq:sentinel:";
        redisOptions.SentinelConfiguration = sentinelOptions =>
        {
            sentinelOptions.MasterName = "mymaster";
            sentinelOptions.SentinelPassword = "sentinel-password";
        };
    });
});
```

## Cloud Redis Services

### Azure Cache for Redis

```csharp
// Using connection string
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = builder.Configuration.GetConnectionString("AzureRedis");
        redisOptions.InstanceName = "tickerq:azure:";
        redisOptions.EnableSsl = true;
        redisOptions.AbortOnConnectFail = false;
    });
});

// Using Azure Key Vault for connection string
builder.Services.AddAzureKeyVault();
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        var connectionString = builder.Configuration["AzureKeyVault:RedisConnectionString"];
        redisOptions.Configuration = connectionString;
        redisOptions.InstanceName = "tickerq:";
    });
});
```

### AWS ElastiCache

```csharp
// ElastiCache Redis
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "tickerq-cluster.abc123.cache.amazonaws.com:6379";
        redisOptions.InstanceName = "tickerq:elasticache:";
        redisOptions.EnableSsl = true;
        redisOptions.CertificateValidation = CertificateValidationMode.None; // For ElastiCache
    });
});

// ElastiCache Redis Cluster
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "tickerq-cluster.abc123.clustercfg.use1.cache.amazonaws.com:6379";
        redisOptions.EnableClusterMode = true;
        redisOptions.InstanceName = "tickerq:cluster:";
    });
});
```

### Google Cloud Memorystore

```csharp
// Memorystore for Redis
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "10.0.0.3:6379"; // Private IP
        redisOptions.InstanceName = "tickerq:gcp:";
        redisOptions.EnableAuth = true;
        redisOptions.Password = builder.Configuration["GCP:Redis:Password"];
    });
});

// With VPC and authentication
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        var configString = $"{builder.Configuration["GCP:Redis:Host"]}:{builder.Configuration["GCP:Redis:Port"]}";
        redisOptions.Configuration = configString;
        redisOptions.Password = builder.Configuration["GCP:Redis:AuthString"];
        redisOptions.InstanceName = "tickerq:memorystore:";
    });
});
```

## Container Orchestration

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --requirepass mypassword
    volumes:
      - redis-data:/data
    networks:
      - tickerq-network

  redis-cluster:
    image: redis:7-alpine
    deploy:
      replicas: 6
    command: >
      sh -c "redis-server --port 7000 --cluster-enabled yes 
             --cluster-config-file nodes.conf --cluster-node-timeout 5000 
             --appendonly yes"
    networks:
      - tickerq-network

  tickerq-app:
    build: .
    depends_on:
      - redis
    environment:
      - Redis__Configuration=redis:6379
      - Redis__Password=mypassword
    networks:
      - tickerq-network

volumes:
  redis-data:

networks:
  tickerq-network:
```

### Kubernetes

```yaml
# redis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: password
        command: ["redis-server"]
        args: ["--requirepass", "$(REDIS_PASSWORD)"]
        volumeMounts:
        - name: redis-storage
          mountPath: /data
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
---
# Application deployment
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
        env:
        - name: Redis__Configuration
          value: "redis-service:6379"
        - name: Redis__Password
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: password
```

## High Availability Setups

### Redis Sentinel with TickerQ

```csharp
// Multiple sentinel nodes
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "sentinel1:26379,sentinel2:26379,sentinel3:26379";
        redisOptions.ServiceName = "mymaster";
        redisOptions.InstanceName = "tickerq:ha:";
        redisOptions.SentinelConfiguration = sentinelOptions =>
        {
            sentinelOptions.MasterName = "mymaster";
            sentinelOptions.CommandMap = CommandMap.Sentinel;
            sentinelOptions.TieBreaker = ""; // Disable tie-breaker for Sentinel
        };
        redisOptions.ConnectionFailover = failoverOptions =>
        {
            failoverOptions.ReconnectRetryPolicy = new ExponentialRetry(TimeSpan.FromSeconds(1));
            failoverOptions.FailoverDetectionTimeout = TimeSpan.FromSeconds(10);
        };
    });
});
```

### Load Balancer Integration

```csharp
// HAProxy/Nginx load balancer
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "redis-lb.internal:6379";
        redisOptions.InstanceName = "tickerq:lb:";
        redisOptions.LoadBalancerConfiguration = lbOptions =>
        {
            lbOptions.HealthCheckInterval = TimeSpan.FromSeconds(30);
            lbOptions.MaxConnectionPoolSize = 50;
            lbOptions.ConnectionMultiplexer = multiplexerOptions =>
            {
                multiplexerOptions.ReconnectRetryPolicy = new LinearRetry(TimeSpan.FromSeconds(5));
            };
        };
    });
});
```

## Security Integrations

### TLS/SSL Configuration

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "secure-redis.company.com:6380";
        redisOptions.EnableSsl = true;
        redisOptions.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;
        redisOptions.CertificateValidation = CertificateValidationMode.ChainTrust;
        redisOptions.ClientCertificate = new X509Certificate2("client-cert.pfx", "password");
    });
});
```

### Redis AUTH and ACL

```csharp
// Redis 6+ ACL support
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "redis.company.com:6379";
        redisOptions.Username = "tickerq-user";
        redisOptions.Password = builder.Configuration["Redis:Password"];
        redisOptions.AclConfiguration = aclOptions =>
        {
            aclOptions.AllowedCommands = new[] { "GET", "SET", "DEL", "EXPIRE", "TTL" };
            aclOptions.AllowedKeyPatterns = new[] { "tickerq:*" };
        };
    });
});
```

## Monitoring and Observability

### Redis Insights Integration

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "redis:6379";
        redisOptions.EnableProfiling = true;
        redisOptions.ProfilingConfiguration = profilingOptions =>
        {
            profilingOptions.EnableSlowLog = true;
            profilingOptions.SlowLogThreshold = TimeSpan.FromMilliseconds(100);
            profilingOptions.EnableCommandLogging = true;
        };
    });
});
```

### Prometheus Metrics

```csharp
// Custom Redis metrics exporter
public class RedisMetricsExporter : IHostedService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IMetricsLogger _metrics;
    
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        // Export Redis connection metrics
        var info = await _redis.GetDatabase().ExecuteAsync("INFO", "stats");
        _metrics.Counter("redis_commands_processed_total")
               .WithTag("instance", _redis.Configuration)
               .Increment(ParseCommandsProcessed(info));
    }
}

builder.Services.AddHostedService<RedisMetricsExporter>();
```

### Application Performance Monitoring

```csharp
// Integration with APM tools
builder.Services.AddTickerQ(options =>
{
    options.AddStackExchangeRedis(redisOptions =>
    {
        redisOptions.Configuration = "redis:6379";
        redisOptions.EnableTracing = true;
        redisOptions.TracingConfiguration = tracingOptions =>
        {
            tracingOptions.IncludeCommandData = true;
            tracingOptions.SanitizeCommands = true;
            tracingOptions.TracingProvider = "OpenTelemetry";
        };
    });
});
```

## Development and Testing

### Redis in Development

```csharp
// Development configuration
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddTickerQ(options =>
    {
        options.AddStackExchangeRedis(redisOptions =>
        {
            redisOptions.Configuration = "localhost:6379";
            redisOptions.InstanceName = "tickerq:dev:";
            redisOptions.EnableDebugging = true;
            redisOptions.LogLevel = LogLevel.Debug;
        });
    });
}
```

### Testing with TestContainers

```csharp
// Integration tests with TestContainers
[Test]
public async Task TestRedisIntegration()
{
    using var redisContainer = new RedisTestcontainer(new RedisTestcontainerConfiguration());
    await redisContainer.StartAsync();
    
    var services = new ServiceCollection();
    services.AddTickerQ(options =>
    {
        options.AddStackExchangeRedis(redisOptions =>
        {
            redisOptions.Configuration = redisContainer.ConnectionString;
            redisOptions.InstanceName = "tickerq:test:";
        });
    });
    
    // Run tests...
}
```

## Migration and Backup

### Redis Data Migration

```csharp
// Migrate from one Redis instance to another
public class RedisMigrationService
{
    public async Task MigrateAsync(string sourceConnection, string targetConnection)
    {
        using var source = ConnectionMultiplexer.Connect(sourceConnection);
        using var target = ConnectionMultiplexer.Connect(targetConnection);
        
        var sourceDb = source.GetDatabase();
        var targetDb = target.GetDatabase();
        
        // Migrate TickerQ keys
        var keys = source.GetServer(source.GetEndPoints().First())
                        .Keys(pattern: "tickerq:*");
        
        foreach (var key in keys)
        {
            var value = await sourceDb.StringGetAsync(key);
            var ttl = await sourceDb.KeyTimeToLiveAsync(key);
            
            await targetDb.StringSetAsync(key, value, ttl);
        }
    }
}
```

## Next Steps

- [Setup Guide](./setup) - Basic Redis configuration
