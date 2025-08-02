---
title: "[Building a Multi-Layer Data Pipeline Architecture] The Mining Layer: Authenticated API Extraction at Scale"
categories:
  - system-design
tags:
  - data-pipeline
---

# The Mining Layer: Authenticated API Extraction at Scale

*Part 3 of 7: Multi-Layer Data Pipeline Architecture*

> **Note**: This series uses a hypothetical Facebook analytics platform as an example to illustrate the architecture patterns. All code examples are for educational purposes only.

---

## The Workhorses: Individual Data Miners

In [Part 2](./part-2.md), we saw how schedulers fan out work to message queues. Now we dive into the **Mining Layer** - where the real magic happens. These Azure Functions are the workhorses that make authenticated API calls, handle pagination intelligently, and implement the self-scheduling pattern that makes the entire system autonomous.

Each miner is responsible for extracting one user's data from Facebook's API, but the patterns we'll explore apply to any authenticated, paginated API system.

## The Miner Architecture Pattern

### Core Miner Structure

Here's the anatomy of a production miner:

```csharp
public class PostsMiner
{
    private readonly ILogger<PostsMiner> logger;
    private readonly ServiceBusSender serviceBusSender;
    private readonly FacebookClientManager facebookClientManager;
    private readonly PostsMiningProcessor postsProcessor;
    private readonly IUserAuthDetailsService userAuthDetailsService;
    private readonly IOptions<FacebookClientSettings> facebookClientSettings;

    [Function(nameof(PostsMiner))]
    public async Task Run(
        [ServiceBusTrigger(MessageBusRoutes.Posts.POSTS_MINING_QUEUE,
         Connection = "MINER_AZURE_SERVICE_BUS")]
        ServiceBusReceivedMessage message)
    {
        logger.LogTrace($"Posts Miner Started");

        // Step 1: Deserialize the mining message
        var messageData = JsonSerializer.Deserialize<PostMiningMessage>(message.Body)
            ?? throw new NullReferenceException($"There was no payload in the message.");

        // Step 2: Validate message state (prevent infinite loops)
        if (!messageData.IsValidMessageState)
        {
            logger.LogWarning($"Invalid message state for user {messageData.UserId}, stopping processing");
            return;
        }

        // Step 3: Get user authentication details
        var authData = await userAuthDetailsService.GetUserAuthDetailsAsync(messageData.UserId);
        if (authData.IsFailed)
        {
            authData.LogIfFailed(LogLevel.Critical);
            throw new NullReferenceException($"Failed to get auth details: {authData.Reasons}");
        }

        // Step 4: Create authenticated HTTP client for this user
        await facebookClientManager.CreateForUserAsync(authData.Value, facebookClientSettings.Value);

        // Step 5: Build API query with pagination state
        var query = new PostsQuery()
        {
            Limit = messageData.Limit,
            PaginationToken = messageData.CurrentMarker?.ToString() ?? PostsQuery.CreateLatestMarker()
        };

        // Step 6: Make authenticated API call
        var clientResult = await facebookClientManager.Clients[authData.Value.FacebookUserId]
            .Client.GetPostsAsync(query);

        if (clientResult.IsFailed)
        {
            clientResult.LogIfFailed(LogLevel.Critical);
            throw new Exception($"API call failed: {clientResult.Reasons}");
        }

        // Step 7: Process the data (deduplication, transformation)
        var shouldContinueProcessing = await postsProcessor.ProcessAsync(
            messageData.UserId, clientResult.Value);

        // Step 8: Self-schedule continuation if more data exists
        if (shouldContinueProcessing
            && clientResult.Value.HasMore.HasValue
            && clientResult.Value.HasMore.Value
            && clientResult.Value.NextToken.HasValue)
        {
            messageData.IncrementDynamicOffsetWithMarker(
                clientResult.Value.Items.Count,
                clientResult.Value.NextToken.Value);

            var jsonPayload = JsonSerializer.Serialize(messageData);
            await serviceBusSender.SendMessageAsync(new ServiceBusMessage(jsonPayload));
        }
        else
        {
            logger.LogInformation($"Mining completed for user {messageData.UserId}. " +
                $"Reason: shouldContinue={shouldContinueProcessing}, " +
                $"hasMore={clientResult.Value.HasMore.GetValueOrDefault()}");
        }
    }
}
```

## Key Patterns in the Mining Layer

### 1. Per-User Client Isolation

The most critical pattern is **per-user authentication isolation**:

```csharp
public class FacebookClientManager
{
    private readonly ConcurrentDictionary<long, IFacebookClient> _clients = new();

    public async Task CreateForUserAsync(FacebookUserAuthDetails authDetails, FacebookClientSettings settings)
    {
        if (_clients.ContainsKey(authDetails.FacebookUserId))
            return; // Client already exists

        var httpClient = _httpClientFactory.CreateClient();

        // Configure authentication headers
        httpClient.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", authDetails.AccessToken);

        httpClient.DefaultRequestHeaders.Add("X-Facebook-User-ID",
            authDetails.FacebookUserId.ToString());

        // Add rate limiting and retry policies
        var client = new FacebookClient(httpClient, settings, authDetails);
        _clients[authDetails.FacebookUserId] = client;

        logger.LogDebug($"Created Facebook client for user {authDetails.FacebookUserId}");
    }
}
```

**Why Per-User Clients?**
- **Authentication isolation**: Each user has their own tokens and rate limits
- **Fault isolation**: One user's banned token doesn't affect others
- **Rate limit distribution**: Facebook limits are per-user, not per-application
- **Debugging**: Easy to trace issues to specific users

### 2. Intelligent Pagination State Management

Traditional pagination loses context on failures. Our approach embeds **resumable state** in messages:

```csharp
public sealed record PostMiningMessage(
    [property: JsonPropertyName("userId")] long UserId,
    [property: JsonPropertyName("limit")] int Limit,
    [property: JsonPropertyName("currentMarker")] long? CurrentMarker,
    [property: JsonPropertyName("consecutiveBatchRepeats")] short? ConsecutiveBatchRepeats = 0,
    [property: JsonPropertyName("consecutiveBatchRepeatsLimit")] short? ConsecutiveBatchRepeatsLimit = 5
) : MarkerMessage(Limit, CurrentMarker, ConsecutiveBatchRepeats, ConsecutiveBatchRepeatsLimit)
{
    public bool IsValidMessageState => ConsecutiveBatchRepeats < ConsecutiveBatchRepeatsLimit;

    public void IncrementDynamicOffsetWithMarker(int itemsProcessed, long nextMarker)
    {
        if (CurrentMarker == nextMarker)
        {
            // Same marker returned - increment repeat counter
            ConsecutiveBatchRepeats++;
        }
        else
        {
            // New marker - reset repeat counter
            ConsecutiveBatchRepeats = 0;
            CurrentMarker = nextMarker;
        }
    }
}
```

This prevents the **infinite pagination loop** problem where APIs return the same "next" marker repeatedly.

### 3. The Self-Scheduling Pattern

This is where the architecture becomes truly elegant. Miners **schedule their own continuation**:

```csharp
// After processing current batch
var shouldContinueProcessing = await postsProcessor.ProcessAsync(messageData.UserId, clientResult.Value);

// Decision tree for continuation
if (shouldContinueProcessing
    && clientResult.Value.HasMore.HasValue
    && clientResult.Value.HasMore.Value
    && clientResult.Value.NextToken.HasValue)
{
    // Update pagination state
    messageData.IncrementDynamicOffsetWithMarker(
        clientResult.Value.Items.Count,
        clientResult.Value.NextToken.Value);

    // Schedule next batch
    var continuationMessage = JsonSerializer.Serialize(messageData);
    await serviceBusSender.SendMessageAsync(new ServiceBusMessage(continuationMessage));

    logger.LogDebug($"Scheduled continuation for user {messageData.UserId}, marker: {messageData.CurrentMarker}");
}
```

**This creates a self-sustaining system:**
1. Scheduler kicks off initial mining jobs
2. Each miner processes a batch and decides whether to continue
3. If continuing, miner schedules itself for the next batch
4. System automatically traverses entire datasets without external coordination

## Advanced Mining Patterns

### 1. Rate Limit Handling with Exponential Backoff

Facebook APIs have complex rate limiting. Here's how we handle it:

```csharp
public class FacebookClient : IFacebookClient
{
    private readonly AsyncRetryPolicy _retryPolicy;

    public FacebookClient(HttpClient httpClient, FacebookClientSettings settings, FacebookUserAuthDetails authDetails)
    {
        _retryPolicy = Policy
            .Handle<HttpRequestException>()
            .Or<TaskCanceledException>()
            .OrResult<HttpResponseMessage>(r => r.StatusCode == HttpStatusCode.TooManyRequests)
            .WaitAndRetryAsync(
                retryCount: 5,
                sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (outcome, timespan, retryCount, context) =>
                {
                    logger.LogWarning($"User {authDetails.FacebookUserId}: Retry {retryCount} after {timespan.TotalSeconds}s");
                });
    }

    public async Task<Result<FacebookListResponse<Post>>> GetPostsAsync(PostsQuery query)
    {
        return await _retryPolicy.ExecuteAsync(async () =>
        {
            var response = await _httpClient.GetAsync($"/v18.0/me/posts?limit={query.Limit}&after={query.PaginationToken}");

            if (response.StatusCode == HttpStatusCode.TooManyRequests)
            {
                // Extract rate limit reset time from headers
                var resetTime = response.Headers.GetValues("X-Business-Use-Case-Usage").FirstOrDefault();
                logger.LogWarning($"Rate limited. Reset time: {resetTime}");
                throw new HttpRequestException("Rate limited");
            }

            response.EnsureSuccessStatusCode();
            var content = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<FacebookListResponse<Post>>(content);
        });
    }
}
```

### 2. User Validation and Filtering

Not all users should be processed. Here's a real-world filtering pattern:

```csharp
// In the miner, before processing
var activeUsers = new List<long>() { 12345, 67890, 11111 }; // Example user whitelist
if (!activeUsers.Contains(messageData.UserId))
{
    logger.LogDebug($"User {messageData.UserId} not in active users list, skipping");
    return;
}

// Additional validation
var userAuthDetails = await userAuthDetailsService.GetUserAuthDetailsAsync(messageData.UserId);
if (userAuthDetails.IsFailed)
{
    logger.LogError($"User {messageData.UserId}: Failed to get auth details: {userAuthDetails.Errors}");
    return; // Don't throw - just skip this user
}

// Check if token is expired
if (userAuthDetails.Value.AccessTokenExpiry < DateTimeOffset.UtcNow)
{
    logger.LogWarning($"User {messageData.UserId}: Access token expired, skipping");
    return;
}
```

### 3. Error Recovery and Dead Letter Handling

When miners fail, we need intelligent recovery:

```csharp
public class PostsDeadLetterRetry
{
    [Function(nameof(PostsDeadLetterRetry))]
    public async Task Run(
        [ServiceBusTrigger(MessageBusRoutes.Posts.POSTS_MINING_QUEUE + "/$deadletterqueue",
         Connection = "MINER_AZURE_SERVICE_BUS")]
        ServiceBusReceivedMessage message)
    {
        logger.LogWarning($"Processing dead letter message for user: {message.Subject}");

        try
        {
            var messageData = JsonSerializer.Deserialize<PostMiningMessage>(message.Body);

            // Analyze failure reason
            var failureReason = GetFailureReason(message);

            switch (failureReason)
            {
                case FailureReason.RateLimited:
                    // Wait longer before retry
                    await Task.Delay(TimeSpan.FromMinutes(15));
                    await RetryMessage(messageData);
                    break;

                case FailureReason.InvalidToken:
                    // Don't retry - user needs to re-authenticate
                    logger.LogError($"User {messageData.UserId}: Invalid token, manual intervention required");
                    break;

                case FailureReason.TemporaryError:
                    // Reset pagination state and retry from beginning
                    messageData.CurrentMarker = null;
                    messageData.ConsecutiveBatchRepeats = 0;
                    await RetryMessage(messageData);
                    break;

                default:
                    logger.LogError($"Unknown failure reason for user {messageData.UserId}");
                    break;
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, $"Failed to process dead letter message: {ex.Message}");
        }
    }
}
```

## Connection Management and Resource Optimization

### HTTP Client Pooling

Managing HTTP connections efficiently is crucial:

```csharp
public class FacebookClientFactory
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ConcurrentDictionary<long, (IFacebookClient Client, DateTime LastUsed)> _clientPool = new();
    private readonly Timer _cleanupTimer;

    public FacebookClientFactory(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;

        // Cleanup unused clients every 30 minutes
        _cleanupTimer = new Timer(CleanupUnusedClients, null,
            TimeSpan.FromMinutes(30), TimeSpan.FromMinutes(30));
    }

    public async Task<IFacebookClient> GetClientAsync(FacebookUserAuthDetails authDetails)
    {
        var userId = authDetails.FacebookUserId;

        if (_clientPool.TryGetValue(userId, out var existingClient))
        {
            // Update last used time
            _clientPool[userId] = (existingClient.Client, DateTime.UtcNow);
            return existingClient.Client;
        }

        // Create new client
        var httpClient = _httpClientFactory.CreateClient("FacebookAPI");
        var client = new FacebookClient(httpClient, authDetails);

        _clientPool[userId] = (client, DateTime.UtcNow);
        return client;
    }

    private void CleanupUnusedClients(object state)
    {
        var cutoffTime = DateTime.UtcNow.AddHours(-2);
        var keysToRemove = _clientPool
            .Where(kvp => kvp.Value.LastUsed < cutoffTime)
            .Select(kvp => kvp.Key)
            .ToList();

        foreach (var key in keysToRemove)
        {
            if (_clientPool.TryRemove(key, out var client))
            {
                client.Client.Dispose();
                logger.LogDebug($"Cleaned up unused client for user {key}");
            }
        }
    }
}
```

## Monitoring and Observability

### Key Metrics for Miners

```csharp
// Performance metrics
logger.LogInformation($"User {userId}: Processing batch {currentBatch}, " +
    $"items: {clientResult.Value.Items.Count}, " +
    $"duration: {stopwatch.ElapsedMilliseconds}ms");

// Progress tracking
logger.LogDebug($"User {userId}: Pagination marker {messageData.CurrentMarker} -> {nextMarker}");

// Error rates
logger.LogWarning($"User {userId}: API error rate: {errorCount}/{totalRequests} = {errorRate:P}");

// Resource utilization
logger.LogTrace($"Active HTTP clients: {_clientPool.Count}, " +
    $"Memory usage: {GC.GetTotalMemory(false) / 1024 / 1024}MB");
```

### Production Dashboard Metrics

Monitor these key indicators:
- **Processing rate**: Posts/minute per user
- **API error rates**: 4xx/5xx responses per user
- **Queue depth**: Messages waiting to be processed
- **Client pool size**: Active HTTP connections
- **Memory usage**: Prevent memory leaks
- **Authentication failures**: Users needing token refresh

## Performance Optimizations

### 1. Batch Size Optimization

```csharp
// Dynamic batch sizing based on user activity
public int GetOptimalBatchSize(long userId)
{
    var userStats = _userStatsCache.GetValueOrDefault(userId);
    if (userStats == null) return 100; // Default

    // Heavy users get larger batches
    if (userStats.AveragePostsPerDay > 1000) return 500;
    if (userStats.AveragePostsPerDay > 100) return 200;
    return 50; // Light users get smaller batches
}
```

### 2. Parallel Processing with Semaphore

```csharp
private readonly SemaphoreSlim _concurrencyLimiter = new(maxConcurrentRequests: 10);

public async Task ProcessUserAsync(PostMiningMessage message)
{
    await _concurrencyLimiter.WaitAsync();
    try
    {
        // Process user data
        await ProcessUserDataAsync(message);
    }
    finally
    {
        _concurrencyLimiter.Release();
    }
}
```

## When Miners Go Wrong: Common Anti-Patterns

### L **What NOT to Do**

**1. Synchronous Processing**
```csharp
// DON'T DO THIS
foreach (var user in users)
{
    var data = await facebookClient.GetPosts(user.Id);
    await ProcessPosts(data);
}
// Blocks on each user sequentially
```

**2. Shared HTTP Client**
```csharp
// DON'T DO THIS
private static readonly HttpClient _sharedClient = new HttpClient();
// Different users will interfere with each other's auth headers
```

**3. Ignoring Rate Limits**
```csharp
// DON'T DO THIS
while (hasMore)
{
    var data = await api.GetData(); // No rate limiting
    // Will get banned quickly
}
```

## Performance Results

In production, our mining layer achieves:

- **2M+ API calls per day** across 500+ users
- **99.5% success rate** with automatic retry handling
- **Linear scalability** - each function instance processes ~50 users
- **Sub-minute error recovery** via dead letter processing
- **Zero authentication conflicts** with per-user client isolation

## What's Next

In **Part 4**, we'll explore the **Processing Layer** - the smart component that handles deduplication, data transformation, and routing. We'll see how the dual-layer caching system (Redis + Database) prevents duplicate processing and how smart early termination saves processing costs.

The processor is where raw API responses become clean, deduplicated data ready for analytics.

---

*Have you built similar mining systems? What patterns have you used for handling authentication and pagination at scale? Share your experiences in the comments below.*

**Previous**: [<- Part 2 - Schedulers & Message Bus Design](./part-2.md)

**Next**: [Part 4 - Processing & Transformation ->](./part-4.md)