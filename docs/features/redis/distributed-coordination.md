# Distributed Coordination

Learn how to use Redis for distributed job coordination, locking, and synchronization in TickerQ.

## Overview

Redis provides robust distributed coordination capabilities for TickerQ, enabling:

- **Distributed Locking** - Prevent concurrent job execution
- **Job Coordination** - Synchronize job execution across multiple instances
- **Leader Election** - Coordinate singleton jobs
- **State Synchronization** - Share job state across instances

## Distributed Locking

### Basic Distributed Lock

```csharp
public class EmailService
{
    private readonly IDistributedLockProvider _lockProvider;
    
    public async Task SendBulkEmailsAsync()
    {
        using var lockHandle = await _lockProvider.AcquireLockAsync(
            "bulk-email-job", 
            TimeSpan.FromMinutes(30));
            
        // Only one instance will execute this code
        await ProcessBulkEmails();
    }
}
```

### Lock Configuration

```csharp
builder.Services.AddTickerQ(options =>
{
    options.UseRedis(redisOptions =>
    {
        redisOptions.LockingOptions = lockingOptions =>
        {
            lockingOptions.DefaultLockTimeout = TimeSpan.FromMinutes(5);
            lockingOptions.LockRetryDelay = TimeSpan.FromMilliseconds(100);
            lockingOptions.MaxLockRetries = 10;
            lockingOptions.EnableLockExtension = true;
            lockingOptions.LockExtensionInterval = TimeSpan.FromMinutes(1);
        };
    });
});
```

### Advanced Locking Patterns

#### Reentrant Locks

```csharp
public class ReentrantLockExample
{
    private readonly IDistributedLockProvider _lockProvider;
    
    public async Task ProcessOrderAsync(string orderId)
    {
        using var lockHandle = await _lockProvider.AcquireLockAsync(
            $"order:{orderId}", 
            TimeSpan.FromMinutes(5),
            reentrant: true);
            
        await UpdateOrderStatus(orderId);
        await ProcessPayment(orderId); // Can acquire same lock
    }
    
    private async Task ProcessPayment(string orderId)
    {
        // This will succeed because the lock is reentrant
        using var lockHandle = await _lockProvider.AcquireLockAsync(
            $"order:{orderId}", 
            TimeSpan.FromMinutes(2),
            reentrant: true);
            
        // Process payment logic
    }
}
```

#### Read-Write Locks

```csharp
public class ReadWriteLockExample
{
    private readonly IDistributedLockProvider _lockProvider;
    
    public async Task<Order> GetOrderAsync(string orderId)
    {
        using var readLock = await _lockProvider.AcquireReadLockAsync(
            $"order:{orderId}", 
            TimeSpan.FromMinutes(1));
            
        return await _orderRepository.GetAsync(orderId);
    }
    
    public async Task UpdateOrderAsync(Order order)
    {
        using var writeLock = await _lockProvider.AcquireWriteLockAsync(
            $"order:{order.Id}", 
            TimeSpan.FromMinutes(5));
            
        await _orderRepository.UpdateAsync(order);
    }
}
```

## Job Coordination

### Singleton Jobs

```csharp
[SingletonJob("data-backup")]
public class DataBackupJob : IJob
{
    public async Task ExecuteAsync(JobContext context)
    {
        // Only one instance across all servers will execute this
        await BackupDatabase();
    }
}
```

### Job Barriers

```csharp
public class JobBarrierExample
{
    private readonly IJobBarrier _jobBarrier;
    
    public async Task ProcessBatchAsync(string batchId, int totalJobs)
    {
        // Wait for all jobs in the batch to complete
        await _jobBarrier.WaitAsync($"batch:{batchId}", totalJobs);
        
        // This code runs only after all batch jobs complete
        await ProcessBatchResults(batchId);
    }
}
```

### Job Dependencies

```csharp
public class DependentJobExample
{
    private readonly IJobDependencyManager _dependencyManager;
    
    public async Task ScheduleDependentJobsAsync()
    {
        var job1Id = await ScheduleJob<DataExtractionJob>();
        var job2Id = await ScheduleJob<DataTransformationJob>();
        
        // This job waits for both dependencies
        await ScheduleJob<DataLoadJob>(dependencies: new[] { job1Id, job2Id });
    }
}
```

## Leader Election

### Simple Leader Election

```csharp
public class LeaderElectionService : IHostedService
{
    private readonly ILeaderElection _leaderElection;
    private readonly ILogger<LeaderElectionService> _logger;
    private CancellationTokenSource _cancellationTokenSource;
    
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        _cancellationTokenSource = new CancellationTokenSource();
        
        _ = Task.Run(async () =>
        {
            while (!_cancellationTokenSource.Token.IsCancellationRequested)
            {
                try
                {
                    using var leadership = await _leaderElection.AcquireLeadershipAsync(
                        "job-scheduler", 
                        TimeSpan.FromMinutes(1));
                        
                    _logger.LogInformation("Acquired leadership");
                    
                    // Execute leader-only tasks
                    await ExecuteLeaderTasks();
                }
                catch (LeadershipLostException)
                {
                    _logger.LogInformation("Leadership lost");
                }
                
                await Task.Delay(TimeSpan.FromSeconds(30), _cancellationTokenSource.Token);
            }
        });
    }
    
    private async Task ExecuteLeaderTasks()
    {
        // Schedule recurring jobs
        // Clean up expired data
        // Send system notifications
    }
}
```

### Leader Election with Callbacks

```csharp
builder.Services.AddTickerQ(options =>
{
    options.UseRedis(redisOptions =>
    {
        redisOptions.LeaderElectionOptions = leaderOptions =>
        {
            leaderOptions.LeadershipTimeout = TimeSpan.FromMinutes(1);
            leaderOptions.RenewalInterval = TimeSpan.FromSeconds(30);
            leaderOptions.OnLeadershipAcquired = async (context) =>
            {
                // Start leader-only services
                await context.ServiceProvider.GetService<IJobScheduler>().StartAsync();
            };
            leaderOptions.OnLeadershipLost = async (context) =>
            {
                // Stop leader-only services
                await context.ServiceProvider.GetService<IJobScheduler>().StopAsync();
            };
        };
    });
});
```

## State Synchronization

### Shared State Management

```csharp
public class SharedStateExample
{
    private readonly IDistributedState _distributedState;
    
    public async Task UpdateJobProgressAsync(string jobId, int progress)
    {
        await _distributedState.SetAsync($"progress:{jobId}", progress);
        
        // Notify other instances of progress update
        await _distributedState.PublishAsync("job-progress-updated", new
        {
            JobId = jobId,
            Progress = progress
        });
    }
    
    public async Task<int> GetJobProgressAsync(string jobId)
    {
        return await _distributedState.GetAsync<int>($"progress:{jobId}");
    }
}
```

### Event Synchronization

```csharp
public class EventSynchronizationExample
{
    private readonly IDistributedEventBus _eventBus;
    
    public async Task PublishJobCompletedEventAsync(string jobId)
    {
        await _eventBus.PublishAsync("job-completed", new JobCompletedEvent
        {
            JobId = jobId,
            CompletedAt = DateTime.UtcNow
        });
    }
    
    public async Task SubscribeToJobEventsAsync()
    {
        await _eventBus.SubscribeAsync<JobCompletedEvent>("job-completed", async (evt) =>
        {
            // Handle job completion across all instances
            await HandleJobCompletion(evt);
        });
    }
}
```

## Coordination Patterns

### Producer-Consumer Pattern

```csharp
public class ProducerConsumerExample
{
    private readonly IDistributedQueue<WorkItem> _workQueue;
    
    // Producer
    public async Task ProduceWorkAsync(IEnumerable<WorkItem> items)
    {
        foreach (var item in items)
        {
            await _workQueue.EnqueueAsync(item);
        }
    }
    
    // Consumer
    public async Task ConsumeWorkAsync(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            var item = await _workQueue.DequeueAsync(TimeSpan.FromSeconds(30));
            if (item != null)
            {
                await ProcessWorkItem(item);
            }
        }
    }
}
```

### Work Stealing Pattern

```csharp
public class WorkStealingExample
{
    private readonly IWorkStealingQueue _workQueue;
    
    public async Task ProcessWorkAsync(string workerId)
    {
        while (true)
        {
            // Try to get work from own queue first
            var work = await _workQueue.TryDequeueAsync(workerId);
            
            if (work == null)
            {
                // Steal work from other workers
                work = await _workQueue.StealWorkAsync(workerId);
            }
            
            if (work != null)
            {
                await ProcessWork(work);
            }
            else
            {
                await Task.Delay(TimeSpan.FromSeconds(1));
            }
        }
    }
}
```

### Circuit Breaker Pattern

```csharp
public class CircuitBreakerExample
{
    private readonly IDistributedCircuitBreaker _circuitBreaker;
    
    public async Task<T> ExecuteWithCircuitBreakerAsync<T>(
        string operationName, 
        Func<Task<T>> operation)
    {
        return await _circuitBreaker.ExecuteAsync(operationName, operation);
    }
}

// Configuration
redisOptions.CircuitBreakerOptions = circuitBreakerOptions =>
{
    circuitBreakerOptions.FailureThreshold = 5;
    circuitBreakerOptions.RecoveryTimeout = TimeSpan.FromMinutes(1);
    circuitBreakerOptions.SamplingDuration = TimeSpan.FromMinutes(5);
};
```

## Coordination Monitoring

### Lock Monitoring

```csharp
public class LockMonitoringService
{
    private readonly ILockMonitor _lockMonitor;
    
    public async Task<LockStatistics> GetLockStatisticsAsync()
    {
        return await _lockMonitor.GetStatisticsAsync();
    }
    
    public async Task<IEnumerable<ActiveLock>> GetActiveLocks()
    {
        return await _lockMonitor.GetActiveLocksAsync();
    }
    
    public async Task ReleaseStaleLocks()
    {
        var staleLocks = await _lockMonitor.GetStaleLocksAsync();
        foreach (var staleLock in staleLocks)
        {
            await _lockMonitor.ReleaseLockAsync(staleLock.Key);
        }
    }
}
```

### Coordination Health Checks

```csharp
public class CoordinationHealthCheck : IHealthCheck
{
    private readonly IDistributedLockProvider _lockProvider;
    private readonly ILeaderElection _leaderElection;
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, 
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Test lock acquisition
            using var testLock = await _lockProvider.AcquireLockAsync(
                "health-check-lock", 
                TimeSpan.FromSeconds(5));
                
            // Test leader election
            var leaderInfo = await _leaderElection.GetLeaderInfoAsync("test-election");
            
            return HealthCheckResult.Healthy("Coordination services are healthy");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Coordination services are unhealthy", ex);
        }
    }
}
```

## Performance Optimization

### Lock Optimization

```csharp
redisOptions.LockingOptions = lockingOptions =>
{
    // Use Lua scripts for atomic operations
    lockingOptions.UseLuaScripts = true;
    
    // Optimize lock acquisition
    lockingOptions.LockRetryDelay = TimeSpan.FromMilliseconds(50);
    lockingOptions.MaxLockRetries = 20;
    
    // Enable lock pooling
    lockingOptions.EnableLockPooling = true;
    lockingOptions.LockPoolSize = 100;
};
```

### Coordination Batching

```csharp
redisOptions.CoordinationOptions = coordinationOptions =>
{
    coordinationOptions.EnableBatching = true;
    coordinationOptions.BatchSize = 50;
    coordinationOptions.BatchTimeout = TimeSpan.FromMilliseconds(100);
};
```

## Troubleshooting

### Common Issues

#### Lock Contention

```csharp
// Monitor lock contention
public class LockContentionMonitor
{
    public async Task MonitorLockContentionAsync()
    {
        var contentionMetrics = await _lockMonitor.GetContentionMetricsAsync();
        
        if (contentionMetrics.AverageWaitTime > TimeSpan.FromSeconds(5))
        {
            // High contention detected
            _logger.LogWarning("High lock contention detected: {Metrics}", contentionMetrics);
        }
    }
}
```

#### Split-Brain Prevention

```csharp
redisOptions.LeaderElectionOptions = leaderOptions =>
{
    leaderOptions.RequireQuorum = true;
    leaderOptions.MinimumNodes = 3;
    leaderOptions.HeartbeatInterval = TimeSpan.FromSeconds(10);
};
```

### Debugging Tools

```csharp
// Enable coordination debugging
redisOptions.DebuggingOptions = debuggingOptions =>
{
    debuggingOptions.EnableLockTracing = true;
    debuggingOptions.EnableLeaderElectionLogging = true;
    debuggingOptions.LogCoordinationEvents = true;
};
```

## Next Steps

- [Integration Guide](./integration) - Integrate with your application
