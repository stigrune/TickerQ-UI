# Redis Installation

Learn how to install and configure Redis for use with TickerQ for distributed job coordination and caching.

## Prerequisites

Before installing Redis for TickerQ, ensure you have:

- .NET 6.0 or later
- Redis server (local or cloud-based)
- TickerQ core package installed

## Package Installation

### Install TickerQ Redis Package

```bash
dotnet add package TickerQ.Redis
```

### Package Dependencies

The TickerQ Redis package includes:

- `StackExchange.Redis` - Redis client library
- `Microsoft.Extensions.Caching.StackExchangeRedis` - ASP.NET Core Redis caching
- `Microsoft.Extensions.DependencyInjection` - Dependency injection support

## Redis Server Setup

### Local Redis Installation

#### Windows (using Chocolatey)

```powershell
# Install Redis using Chocolatey
choco install redis-64

# Start Redis service
redis-server
```

#### Windows (using WSL)

```bash
# Install Redis on Ubuntu/WSL
sudo apt update
sudo apt install redis-server

# Start Redis service
sudo service redis-server start

# Test Redis connection
redis-cli ping
```

#### macOS (using Homebrew)

```bash
# Install Redis using Homebrew
brew install redis

# Start Redis service
brew services start redis

# Test Redis connection
redis-cli ping
```

#### Linux (Ubuntu/Debian)

```bash
# Update package list
sudo apt update

# Install Redis
sudo apt install redis-server

# Start Redis service
sudo systemctl start redis-server
sudo systemctl enable redis-server

# Test Redis connection
redis-cli ping
```

### Docker Redis Setup

#### Basic Redis Container

```bash
# Run Redis in Docker
docker run -d \
  --name tickerq-redis \
  -p 6379:6379 \
  redis:7-alpine

# Test connection
docker exec -it tickerq-redis redis-cli ping
```

#### Redis with Persistence

```bash
# Run Redis with data persistence
docker run -d \
  --name tickerq-redis \
  -p 6379:6379 \
  -v redis-data:/data \
  redis:7-alpine redis-server --appendonly yes
```

#### Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    container_name: tickerq-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  app:
    build: .
    depends_on:
      - redis
    environment:
      - ConnectionStrings__Redis=localhost:6379

volumes:
  redis-data:
```

### Cloud Redis Services

#### Azure Cache for Redis

```csharp
// Azure Redis configuration
builder.Services.AddTickerQ(options =>
{
    options.UseRedis(redisOptions =>
    {
        redisOptions.ConnectionString = "your-azure-redis.redis.cache.windows.net:6380,password=your-access-key,ssl=True,abortConnect=False";
        redisOptions.InstanceName = "TickerQ";
    });
});
```

#### AWS ElastiCache

```csharp
// AWS ElastiCache configuration
builder.Services.AddTickerQ(options =>
{
    options.UseRedis(redisOptions =>
    {
        redisOptions.ConnectionString = "your-cluster.cache.amazonaws.com:6379";
        redisOptions.InstanceName = "TickerQ";
        redisOptions.ConfigurationOptions = configOptions =>
        {
            configOptions.Ssl = true;
            configOptions.SslProtocols = SslProtocols.Tls12;
        };
    });
});
```

#### Google Cloud Memorystore

```csharp
// Google Cloud Memorystore configuration
builder.Services.AddTickerQ(options =>
{
    options.UseRedis(redisOptions =>
    {
        redisOptions.ConnectionString = "10.0.0.3:6379"; // Internal IP
        redisOptions.InstanceName = "TickerQ";
    });
});
```

## TickerQ Redis Configuration

### Basic Configuration

```csharp
// Program.cs
using TickerQ.Redis;

var builder = WebApplication.CreateBuilder(args);

// Add TickerQ with Redis
builder.Services.AddTickerQ(options =>
{
    options.UseRedis(redisOptions =>
    {
        redisOptions.ConnectionString = "localhost:6379";
        redisOptions.InstanceName = "TickerQ";
        redisOptions.Database = 0;
    });
});

var app = builder.Build();
app.Run();
```

### Advanced Configuration

```csharp
builder.Services.AddTickerQ(options =>
{
    options.UseRedis(redisOptions =>
    {
        redisOptions.ConnectionString = builder.Configuration.GetConnectionString("Redis");
        redisOptions.InstanceName = "TickerQ";
        redisOptions.Database = 0;
        
        // Connection options
        redisOptions.ConfigurationOptions = configOptions =>
        {
            configOptions.ConnectTimeout = 5000;
            configOptions.SyncTimeout = 5000;
            configOptions.AsyncTimeout = 5000;
            configOptions.ConnectRetry = 3;
            configOptions.ReconnectRetryPolicy = new ExponentialRetry(1000);
            configOptions.AbortOnConnectFail = false;
        };
        
        // Serialization options
        redisOptions.SerializationOptions = serializationOptions =>
        {
            serializationOptions.UseSystemTextJson = true;
            serializationOptions.JsonSerializerOptions = new JsonSerializerOptions
            {
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
                WriteIndented = false
            };
        };
        
        // Key prefix for Redis keys
        redisOptions.KeyPrefix = "tickerq:";
        
        // Enable compression for large payloads
        redisOptions.EnableCompression = true;
        redisOptions.CompressionThreshold = 1024; // Compress if > 1KB
    });
});
```

### Connection String Configuration

#### appsettings.json

```json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379"
  },
  "TickerQ": {
    "Redis": {
      "InstanceName": "TickerQ",
      "Database": 0,
      "KeyPrefix": "tickerq:",
      "EnableCompression": true,
      "CompressionThreshold": 1024
    }
  }
}
```

#### Environment Variables

```bash
# Connection string
export ConnectionStrings__Redis="localhost:6379"

# Redis options
export TickerQ__Redis__InstanceName="TickerQ"
export TickerQ__Redis__Database="0"
export TickerQ__Redis__KeyPrefix="tickerq:"
```

### Configuration Options

```csharp
public class RedisOptions
{
    public string ConnectionString { get; set; }
    public string InstanceName { get; set; } = "TickerQ";
    public int Database { get; set; } = 0;
    public string KeyPrefix { get; set; } = "tickerq:";
    public bool EnableCompression { get; set; } = false;
    public int CompressionThreshold { get; set; } = 1024;
    public TimeSpan DefaultExpiry { get; set; } = TimeSpan.FromHours(24);
    public bool EnableHealthChecks { get; set; } = true;
    public Action<ConfigurationOptions> ConfigurationOptions { get; set; }
    public Action<SerializationOptions> SerializationOptions { get; set; }
}
```

## Redis Cluster Configuration

### Redis Cluster Setup

```csharp
builder.Services.AddTickerQ(options =>
{
    options.UseRedis(redisOptions =>
    {
        redisOptions.ConnectionString = "node1:6379,node2:6379,node3:6379";
        redisOptions.InstanceName = "TickerQ";
        
        redisOptions.ConfigurationOptions = configOptions =>
        {
            configOptions.EndPoints.Add("node1", 6379);
            configOptions.EndPoints.Add("node2", 6379);
            configOptions.EndPoints.Add("node3", 6379);
            configOptions.AbortOnConnectFail = false;
            configOptions.ConnectRetry = 3;
        };
    });
});
```

### Redis Sentinel Configuration

```csharp
builder.Services.AddTickerQ(options =>
{
    options.UseRedis(redisOptions =>
    {
        redisOptions.ConfigurationOptions = configOptions =>
        {
            configOptions.EndPoints.Add("sentinel1", 26379);
            configOptions.EndPoints.Add("sentinel2", 26379);
            configOptions.EndPoints.Add("sentinel3", 26379);
            configOptions.ServiceName = "mymaster";
            configOptions.TieBreaker = "";
            configOptions.CommandMap = CommandMap.Sentinel;
        };
    });
});
```

## Health Checks

### Redis Health Check

```csharp
// Add Redis health checks
builder.Services.AddHealthChecks()
    .AddRedis(builder.Configuration.GetConnectionString("Redis"));

// Configure health check endpoint
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

### Custom Health Check

```csharp
public class RedisHealthCheck : IHealthCheck
{
    private readonly IConnectionMultiplexer _redis;
    
    public RedisHealthCheck(IConnectionMultiplexer redis)
    {
        _redis = redis;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, 
        CancellationToken cancellationToken = default)
    {
        try
        {
            var database = _redis.GetDatabase();
            await database.PingAsync();
            
            return HealthCheckResult.Healthy("Redis is responsive");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Redis is not responsive", ex);
        }
    }
}
```

## Performance Optimization

### Connection Pooling

```csharp
// Configure connection multiplexer as singleton
builder.Services.AddSingleton<IConnectionMultiplexer>(provider =>
{
    var connectionString = builder.Configuration.GetConnectionString("Redis");
    var configuration = ConfigurationOptions.Parse(connectionString);
    
    // Optimize for high throughput
    configuration.ConnectTimeout = 5000;
    configuration.SyncTimeout = 5000;
    configuration.AsyncTimeout = 5000;
    configuration.ConnectRetry = 3;
    configuration.KeepAlive = 60;
    
    return ConnectionMultiplexer.Connect(configuration);
});
```

### Memory Optimization

```csharp
redisOptions.SerializationOptions = serializationOptions =>
{
    // Use binary serialization for better performance
    serializationOptions.UseMessagePack = true;
    
    // Enable compression for large objects
    serializationOptions.EnableCompression = true;
    serializationOptions.CompressionLevel = CompressionLevel.Fastest;
};
```

## Troubleshooting

### Common Issues

#### Connection Timeouts

```csharp
redisOptions.ConfigurationOptions = configOptions =>
{
    configOptions.ConnectTimeout = 10000; // 10 seconds
    configOptions.SyncTimeout = 10000;
    configOptions.AsyncTimeout = 10000;
    configOptions.AbortOnConnectFail = false;
};
```

#### Memory Issues

```bash
# Check Redis memory usage
redis-cli info memory

# Set max memory policy
redis-cli config set maxmemory-policy allkeys-lru
redis-cli config set maxmemory 2gb
```

#### Network Issues

```bash
# Test Redis connectivity
telnet localhost 6379

# Check Redis logs
tail -f /var/log/redis/redis-server.log

# Monitor Redis commands
redis-cli monitor
```

### Logging Configuration

```csharp
// Enable Redis logging
builder.Services.AddLogging(logging =>
{
    logging.AddConsole();
    logging.SetMinimumLevel(LogLevel.Debug);
});

// Configure StackExchange.Redis logging
builder.Services.Configure<LoggerFilterOptions>(options =>
{
    options.AddFilter("StackExchange.Redis", LogLevel.Information);
});
```

## Next Steps

- [Redis Setup Guide](./setup) - Configure Redis for TickerQ
- [Integration Guide](./integration) - Integrate Redis with your application
