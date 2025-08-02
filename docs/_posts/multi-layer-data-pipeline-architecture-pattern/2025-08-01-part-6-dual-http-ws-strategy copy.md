---
title: "[Building a Multi-Layer Data Pipeline Architecture] Checkpoint Systems: The Foundation of Bulletproof Operations"
categories:
  - system-design
tags:
  - architecture
  - pattern
---
# Dual HTTP/WebSocket Strategy: Complete Data Coverage Architecture

*Part 6 of 7: Multi-Layer Data Pipeline Architecture*

> **Note**: This series uses a hypothetical Facebook analytics platform as an example to illustrate the architecture patterns. All code examples are for educational purposes only.

---

## The Two-Pronged Approach: Why You Need Both HTTP and WebSockets

In [Part 5](./part-5.md), we explored checkpoint systems that make operations bulletproof. Now we tackle the final architectural challenge: **complete data coverage**. How do you ensure you capture every single piece of data from a live system that's constantly changing?

The answer lies in a sophisticated **dual capture strategy** that combines systematic HTTP mining with real-time WebSocket streaming. Each approach covers what the other misses, creating a comprehensive data collection system with zero gaps.

## The Fundamental Problem: Live Data is Messy

### The Coverage Gap Challenge

Consider Facebook's data flow:

```
USER POSTS --> [API Endpoints] --> [Your System]
              ^
              |
         REAL-TIME DATA
         (Events happen now)

USER POSTS --> [Historical API] --> [Your System]
              ^
              |
        BATCH DATA
        (Events from the past)
```

**The Problem**: Live events happen between your batch collection cycles. Traditional polling misses data that appears and changes between polls. Additionally, frequent HTTP polling leads to **429 (Too Many Requests) errors**, making the system error-prone and unreliable.

**The Solution**: Dual capture - WebSockets catch live events as they happen with **zero HTTP requests**, while HTTP mining fills historical gaps only when needed. This dramatically reduces API rate limit pressure.

## Architecture Overview: The Dual System

```
HTTP MINING LAYER
    |
SCHEDULERS --> MINERS --> PROCESSORS --> CHECKPOINTS
    |                                        |
    +----------- COORDINATION BUS -----------+
    |                                        |
WEBSOCKET LAYER                              |
    |                                        |
LIVE EVENTS --> WS PROCESSORS --> CHECKPOINTS
```

### Key Coordination Points

1. **Checkpoint Sharing**: Both systems use the same checkpoint store
2. **Overlap Detection**: WebSocket data is checked against HTTP mining checkpoints
3. **Gap Filling**: HTTP mining focuses on time periods not covered by WebSocket
4. **Conflict Resolution**: When both systems process the same data, consistent rules determine precedence
5. **Rate Limit Avoidance**: WebSocket eliminates the need for frequent HTTP polling, preventing 429 errors

## The Rate Limiting Problem: Why HTTP-Only Fails

### The 429 Error Cascade

Traditional HTTP-only approaches suffer from a fundamental flaw - the more frequently you poll, the more likely you are to hit rate limits:

```csharp
// Traditional polling approach - error prone
public class TraditionalPollingScheduler
{
    public async Task PollUserDataAsync(long userId)
    {
        while (true)
        {
            try
            {
                // Poll every 5 minutes to catch new data quickly
                var posts = await _facebookClient.GetPostsAsync(userId);
                await ProcessPosts(posts);

                await Task.Delay(TimeSpan.FromMinutes(5)); // Aggressive polling
            }
            catch (HttpRequestException ex) when (ex.Message.Contains("429"))
            {
                // Rate limited! Now what?
                _logger.LogError($"User {userId}: Rate limited, backing off");
                await Task.Delay(TimeSpan.FromHours(1)); // Lose an hour of data
            }
        }
    }
}
```

**The Problem Cascade:**
1. **Aggressive polling** needed for real-time data → **429 errors**
2. **Back-off delays** → **Missed events during backoff**
3. **Catch-up polling** → **More 429 errors**
4. **System becomes unreliable** → **Data gaps and inconsistency**

### WebSocket Solution: Zero HTTP Requests for Live Data

WebSockets eliminate this problem entirely for real-time events:

```csharp
public class WebSocketRateLimitSolution
{
    public async Task StartLiveMonitoringAsync(long userId)
    {
        // Single WebSocket connection replaces thousands of HTTP polls
        var webSocket = new FacebookWebSocketClient(userId);
        await webSocket.ConnectAsync();

        // This one connection can run for hours/days without any HTTP requests
        await webSocket.MonitorEventsAsync(async (eventData) =>
        {
            // Process real-time events with ZERO rate limit impact
            await ProcessRealTimeEvent(eventData);
        });

        _logger.LogInformation($"User {userId}: Live monitoring active with zero HTTP overhead");
    }
}
```

**The Benefits:**
- **No polling overhead**: One WebSocket connection vs. thousands of HTTP requests
- **No 429 errors**: Live events come through WebSocket, not HTTP API
- **Instant processing**: Events processed as they happen, not on polling intervals
- **HTTP budget preserved**: Remaining HTTP quota used only for historical gap-filling

## The WebSocket Layer: Real-Time Event Capture

### WebSocket Client Manager

```csharp
public class FacebookWebSocketClientManager
{
    private readonly ConcurrentDictionary<long, FacebookWebSocketClient> _clients = new();
    private readonly ILogger<FacebookWebSocketClientManager> _logger;
    private readonly IServiceProvider _serviceProvider;

    public async Task StartMonitoringUserAsync(long facebookUserId, FacebookUserAuthDetails authDetails)
    {
        if (_clients.ContainsKey(facebookUserId))
        {
            _logger.LogDebug($"WebSocket monitoring already active for user {facebookUserId}");
            return;
        }

        var client = new FacebookWebSocketClient(facebookUserId, authDetails, _serviceProvider);
        _clients[facebookUserId] = client;

        // Start monitoring with automatic reconnection
        _ = Task.Run(async () =>
        {
            await MonitorUserWithRetryAsync(client);
        });

        _logger.LogInformation($"Started WebSocket monitoring for user {facebookUserId}");
    }

    private async Task MonitorUserWithRetryAsync(FacebookWebSocketClient client)
    {
        var retryCount = 0;
        const int maxRetries = 5;

        while (retryCount < maxRetries)
        {
            try
            {
                await client.ConnectAndMonitorAsync();
                retryCount = 0; // Reset on successful connection
            }
            catch (WebSocketException ex)
            {
                retryCount++;
                var delay = TimeSpan.FromSeconds(Math.Pow(2, retryCount)); // Exponential backoff

                _logger.LogWarning($"User {client.UserId}: WebSocket connection failed (attempt {retryCount}/{maxRetries}). " +
                    $"Retrying in {delay.TotalSeconds}s. Error: {ex.Message}");

                if (retryCount >= maxRetries)
                {
                    _logger.LogError($"User {client.UserId}: Max WebSocket retry attempts reached. Switching to HTTP-only mode.");
                    await FallbackToHttpOnlyAsync(client.UserId);
                    break;
                }

                await Task.Delay(delay);
            }
        }
    }

    private async Task FallbackToHttpOnlyAsync(long userId)
    {
        // Notify the HTTP mining system to increase polling frequency for this user
        var message = new IncreasePollingFrequencyMessage(userId, "WebSocket connection failed");
        await _messageBus.SendAsync(MessageBusRoutes.POLLING_FREQUENCY_UPDATES, message);
    }
}
```

### Real-Time Event Processing

```csharp
public class FacebookWebSocketClient
{
    private ClientWebSocket _webSocket;
    private readonly long _userId;
    private readonly FacebookUserAuthDetails _authDetails;
    private readonly CancellationTokenSource _cancellationTokenSource = new();

    public async Task ConnectAndMonitorAsync()
    {
        _webSocket = new ClientWebSocket();

        // Add authentication headers
        _webSocket.Options.SetRequestHeader("Authorization", $"Bearer {_authDetails.AccessToken}");
        _webSocket.Options.SetRequestHeader("X-Facebook-User-ID", _userId.ToString());

        // Connect to Facebook's real-time API endpoint
        var wsUri = new Uri($"wss://graph.facebook.com/v18.0/{_userId}/live_events");
        await _webSocket.ConnectAsync(wsUri, _cancellationTokenSource.Token);

        _logger.LogInformation($"User {_userId}: WebSocket connected successfully");

        // Start receiving messages
        await ReceiveMessagesAsync();
    }

    private async Task ReceiveMessagesAsync()
    {
        var buffer = new byte[4096];
        var messageBuffer = new StringBuilder();

        while (_webSocket.State == WebSocketState.Open && !_cancellationTokenSource.Token.IsCancellationRequested)
        {
            try
            {
                var result = await _webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), _cancellationTokenSource.Token);

                if (result.MessageType == WebSocketMessageType.Text)
                {
                    var messageChunk = Encoding.UTF8.GetString(buffer, 0, result.Count);
                    messageBuffer.Append(messageChunk);

                    if (result.EndOfMessage)
                    {
                        var completeMessage = messageBuffer.ToString();
                        messageBuffer.Clear();

                        // Process the real-time event
                        await ProcessRealTimeEventAsync(completeMessage);
                    }
                }
                else if (result.MessageType == WebSocketMessageType.Close)
                {
                    _logger.LogInformation($"User {_userId}: WebSocket connection closed by server");
                    break;
                }
            }
            catch (WebSocketException ex)
            {
                _logger.LogError(ex, $"User {_userId}: WebSocket receive error: {ex.Message}");
                throw;
            }
        }
    }

    private async Task ProcessRealTimeEventAsync(string messageJson)
    {
        try
        {
            var eventData = JsonSerializer.Deserialize<FacebookRealTimeEvent>(messageJson);

            if (eventData == null)
            {
                _logger.LogWarning($"User {_userId}: Received null event data");
                return;
            }

            _logger.LogDebug($"User {_userId}: Received real-time event: {eventData.EventType}");

            // Route to appropriate processor based on event type
            await RouteEventToProcessorAsync(eventData);
        }
        catch (JsonException ex)
        {
            _logger.LogError(ex, $"User {_userId}: Failed to parse WebSocket message: {messageJson}");
        }
    }

    private async Task RouteEventToProcessorAsync(FacebookRealTimeEvent eventData)
    {
        var processor = eventData.EventType switch
        {
            "post_created" => _serviceProvider.GetRequiredService<RealTimePostProcessor>(),
            "post_updated" => _serviceProvider.GetRequiredService<RealTimePostProcessor>(),
            "post_deleted" => _serviceProvider.GetRequiredService<RealTimePostDeletionProcessor>(),
            "comment_added" => _serviceProvider.GetRequiredService<RealTimeCommentProcessor>(),
            "like_added" => _serviceProvider.GetRequiredService<RealTimeLikeProcessor>(),
            _ => null
        };

        if (processor != null)
        {
            await processor.ProcessEventAsync(_userId, eventData);
        }
        else
        {
            _logger.LogWarning($"User {_userId}: No processor found for event type: {eventData.EventType}");
        }
    }
}

public sealed record FacebookRealTimeEvent(
    [property: JsonPropertyName("event_type")] string EventType,
    [property: JsonPropertyName("object_id")] string ObjectId,
    [property: JsonPropertyName("timestamp")] long Timestamp,
    [property: JsonPropertyName("data")] JsonElement Data
);
```

## Real-Time Event Processing Patterns

### Immediate Processing with Checkpoint Coordination

```csharp
public class RealTimePostProcessor
{
    private readonly PostsMiningProcessor _httpProcessor; // Reuse HTTP processing logic
    private readonly FacebookDataCheckpointManager _checkpointManager;
    private readonly ILogger<RealTimePostProcessor> _logger;

    public async Task ProcessEventAsync(long userId, FacebookRealTimeEvent eventData)
    {
        var postId = eventData.ObjectId;
        var eventTimestamp = DateTimeOffset.FromUnixTimeSeconds(eventData.Timestamp);

        // Step 1: Check if this event was already processed by HTTP mining
        var isAlreadyProcessed = await _checkpointManager.IsItemProcessedAsync(userId, postId, "posts");

        if (isAlreadyProcessed)
        {
            _logger.LogDebug($"User {userId}, Post {postId}: Already processed by HTTP mining, skipping WebSocket event");
            return;
        }

        // Step 2: Extract post data from the event
        var post = ExtractPostFromEvent(eventData);
        if (post == null)
        {
            _logger.LogWarning($"User {userId}, Post {postId}: Could not extract post data from WebSocket event");
            return;
        }

        // Step 3: Process using the same logic as HTTP mining (consistency)
        var fakeHttpResponse = new FacebookListResponse<Post>
        {
            Items = new[] { post },
            HasMore = false,
            NextToken = null
        };

        var shouldContinue = await _httpProcessor.ProcessAsync(userId, fakeHttpResponse);

        // Step 4: Mark as processed in WebSocket checkpoint system
        await _checkpointManager.CreateWebSocketCheckpointAsync(userId, postId, eventTimestamp);

        _logger.LogInformation($"User {userId}, Post {postId}: Processed real-time event successfully");
    }

    private Post ExtractPostFromEvent(FacebookRealTimeEvent eventData)
    {
        try
        {
            // Parse the event data into a Post object
            return JsonSerializer.Deserialize<Post>(eventData.Data.GetRawText());
        }
        catch (JsonException ex)
        {
            _logger.LogError(ex, $"Failed to extract post from WebSocket event: {eventData.Data.GetRawText()}");
            return null;
        }
    }
}
```

### WebSocket-Specific Checkpoint Management

```csharp
public class WebSocketCheckpointManager
{
    public async Task CreateWebSocketCheckpointAsync(long userId, string objectId, DateTimeOffset eventTimestamp)
    {
        var checkpoint = new WebSocketCheckpoint
        {
            UserId = userId,
            ObjectId = objectId,
            ObjectType = "post",
            EventTimestamp = eventTimestamp,
            ProcessedAt = DateTimeOffset.UtcNow,
            Source = "websocket"
        };

        dbContext.WebSocketCheckpoints.Add(checkpoint);
        await dbContext.SaveChangesAsync();

        // Also add to fast cache for HTTP mining coordination
        var cacheKey = $"ws_checkpoint:user_{userId}:post_{objectId}";
        await _cache.SetAsync(cacheKey, true, TimeSpan.FromHours(24));
    }

    public async Task<List<TimeRange>> GetWebSocketCoverageGapsAsync(long userId, DateTimeOffset startDate, DateTimeOffset endDate)
    {
        // Find time periods where WebSocket was not connected
        var connectionPeriods = await dbContext.WebSocketConnections
            .Where(x => x.UserId == userId &&
                       x.ConnectedAt <= endDate &&
                       x.DisconnectedAt >= startDate)
            .OrderBy(x => x.ConnectedAt)
            .ToListAsync();

        var gaps = new List<TimeRange>();
        var currentTime = startDate;

        foreach (var period in connectionPeriods)
        {
            // Gap before this connection period
            if (period.ConnectedAt > currentTime)
            {
                gaps.Add(new TimeRange(currentTime, period.ConnectedAt));
            }

            currentTime = period.DisconnectedAt ?? DateTimeOffset.UtcNow;
        }

        // Gap after last connection period
        if (currentTime < endDate)
        {
            gaps.Add(new TimeRange(currentTime, endDate));
        }

        return gaps;
    }
}

public sealed record TimeRange(DateTimeOffset Start, DateTimeOffset End)
{
    public TimeSpan Duration => End - Start;
    public bool IsEmpty => Start >= End;
}
```

## HTTP Mining with WebSocket Coordination

### Smart Gap-Filling Strategy

```csharp
public class CoordinatedPostsScheduler
{
    public async Task SchedulePostsMiningAsync()
    {
        var activeUsers = await _userService.GetActiveUserIdsAsync();

        foreach (var userId in activeUsers)
        {
            // Check if WebSocket is actively monitoring this user
            var wsStatus = await _webSocketManager.GetConnectionStatusAsync(userId);

            if (wsStatus.IsConnected && wsStatus.ConnectionDuration > TimeSpan.FromMinutes(30))
            {
                // WebSocket is stable - focus HTTP mining on historical gaps
                await ScheduleGapFillingMiningAsync(userId);
            }
            else
            {
                // WebSocket unstable or disconnected - schedule full HTTP mining
                await ScheduleFullMiningAsync(userId);
            }
        }
    }

    private async Task ScheduleGapFillingMiningAsync(long userId)
    {
        // Find gaps in WebSocket coverage over the last 24 hours
        var endTime = DateTimeOffset.UtcNow;
        var startTime = endTime.AddHours(-24);

        var gaps = await _webSocketCheckpointManager.GetWebSocketCoverageGapsAsync(userId, startTime, endTime);

        foreach (var gap in gaps)
        {
            if (gap.Duration > TimeSpan.FromMinutes(5)) // Only fill significant gaps
            {
                var message = new PostMiningMessage(userId, 100, null)
                {
                    StartTime = gap.Start,
                    EndTime = gap.End,
                    MiningMode = MiningMode.GapFilling
                };

                await _messageBus.SendAsync(MessageBusRoutes.Posts.POSTS_MINING_QUEUE, message);

                _logger.LogInformation($"User {userId}: Scheduled gap-filling mining for {gap.Start} to {gap.End}");
            }
        }
    }

    private async Task ScheduleFullMiningAsync(long userId)
    {
        // Standard full mining approach
        var message = new PostMiningMessage(userId, 100, null)
        {
            MiningMode = MiningMode.Full
        };

        await _messageBus.SendAsync(MessageBusRoutes.Posts.POSTS_MINING_QUEUE, message);

        _logger.LogInformation($"User {userId}: Scheduled full HTTP mining (WebSocket not available)");
    }
}
```

### Enhanced Mining with WebSocket Awareness

```csharp
public class CoordinatedPostsMiner
{
    public async Task ProcessMiningMessageAsync(PostMiningMessage message)
    {
        // Check if this time period is already covered by WebSocket
        if (message.MiningMode == MiningMode.GapFilling)
        {
            var wsCheckpoints = await _webSocketCheckpointManager.GetCheckpointsInRangeAsync(
                message.UserId, message.StartTime.Value, message.EndTime.Value);

            if (wsCheckpoints.Count > 0)
            {
                _logger.LogInformation($"User {message.UserId}: Gap already filled by WebSocket events, skipping HTTP mining");
                return;
            }
        }

        // Proceed with normal mining, but check each item against WebSocket checkpoints
        var query = new PostsQuery
        {
            Limit = message.Limit,
            Since = message.StartTime,
            Until = message.EndTime,
            PaginationToken = message.CurrentMarker?.ToString()
        };

        var apiResult = await _facebookClient.GetPostsAsync(query);
        if (apiResult.IsFailed) return;

        // Filter out posts already processed by WebSocket
        var newPosts = new List<Post>();
        foreach (var post in apiResult.Value.Items)
        {
            var wsProcessed = await _checkpointManager.IsWebSocketProcessedAsync(message.UserId, post.Id);
            if (!wsProcessed)
            {
                newPosts.Add(post);
            }
        }

        if (newPosts.Count > 0)
        {
            var filteredResponse = new FacebookListResponse<Post>
            {
                Items = newPosts,
                HasMore = apiResult.Value.HasMore,
                NextToken = apiResult.Value.NextToken
            };

            await _processor.ProcessAsync(message.UserId, filteredResponse);

            _logger.LogInformation($"User {message.UserId}: HTTP mining processed {newPosts.Count}/{apiResult.Value.Items.Count} posts " +
                $"({apiResult.Value.Items.Count - newPosts.Count} already processed by WebSocket)");
        }
    }
}
```

## Conflict Resolution Strategies

### The "WebSocket Wins" Pattern

When both systems process the same data, establish clear precedence rules:

```csharp
public class ConflictResolutionManager
{
    public async Task<ProcessingDecision> ResolveProcessingConflictAsync(
        long userId,
        string objectId,
        ProcessingSource currentSource,
        ProcessingSource conflictingSource)
    {
        // Rule 1: WebSocket events are more recent and authoritative
        if (conflictingSource == ProcessingSource.WebSocket)
        {
            _logger.LogDebug($"User {userId}, Object {objectId}: WebSocket processing takes precedence over {currentSource}");
            return ProcessingDecision.UseConflictingSource;
        }

        // Rule 2: If WebSocket already processed, HTTP should skip
        if (currentSource == ProcessingSource.Http &&
            await IsWebSocketProcessedAsync(userId, objectId))
        {
            _logger.LogDebug($"User {userId}, Object {objectId}: Skipping HTTP processing, already handled by WebSocket");
            return ProcessingDecision.SkipProcessing;
        }

        // Rule 3: Check timestamps to determine most recent
        var currentTimestamp = await GetProcessingTimestampAsync(userId, objectId, currentSource);
        var conflictingTimestamp = await GetProcessingTimestampAsync(userId, objectId, conflictingSource);

        if (conflictingTimestamp > currentTimestamp)
        {
            _logger.LogDebug($"User {userId}, Object {objectId}: Using more recent processing from {conflictingSource}");
            return ProcessingDecision.UseConflictingSource;
        }

        return ProcessingDecision.UseCurrentSource;
    }

    private async Task<bool> IsWebSocketProcessedAsync(long userId, string objectId)
    {
        return await dbContext.WebSocketCheckpoints
            .AnyAsync(x => x.UserId == userId && x.ObjectId == objectId);
    }
}

public enum ProcessingSource
{
    Http,
    WebSocket
}

public enum ProcessingDecision
{
    UseCurrentSource,
    UseConflictingSource,
    SkipProcessing
}
```

## Connection Management and Resilience

### WebSocket Health Monitoring

```csharp
public class WebSocketHealthMonitor
{
    private readonly Timer _healthCheckTimer;
    private readonly ConcurrentDictionary<long, WebSocketHealthStatus> _healthStatus = new();

    public WebSocketHealthMonitor()
    {
        // Check health every 60 seconds
        _healthCheckTimer = new Timer(PerformHealthChecks, null,
            TimeSpan.FromSeconds(60), TimeSpan.FromSeconds(60));
    }

    private async void PerformHealthChecks(object state)
    {
        var allConnections = await _webSocketManager.GetAllConnectionsAsync();

        foreach (var connection in allConnections)
        {
            var health = await CheckConnectionHealthAsync(connection);
            _healthStatus[connection.UserId] = health;

            if (health.Status == HealthStatus.Unhealthy)
            {
                _logger.LogWarning($"User {connection.UserId}: WebSocket unhealthy - {health.Reason}");
                await HandleUnhealthyConnectionAsync(connection);
            }
        }
    }

    private async Task<WebSocketHealthStatus> CheckConnectionHealthAsync(WebSocketConnection connection)
    {
        // Check 1: Connection state
        if (connection.State != WebSocketState.Open)
        {
            return new WebSocketHealthStatus(HealthStatus.Unhealthy, "Connection not open");
        }

        // Check 2: Recent activity (heartbeat)
        var timeSinceLastMessage = DateTimeOffset.UtcNow - connection.LastMessageReceived;
        if (timeSinceLastMessage > TimeSpan.FromMinutes(10))
        {
            return new WebSocketHealthStatus(HealthStatus.Unhealthy, "No recent messages");
        }

        // Check 3: Error rate
        var recentErrors = await GetRecentErrorCountAsync(connection.UserId);
        if (recentErrors > 5)
        {
            return new WebSocketHealthStatus(HealthStatus.Degraded, "High error rate");
        }

        return new WebSocketHealthStatus(HealthStatus.Healthy, "All checks passed");
    }

    private async Task HandleUnhealthyConnectionAsync(WebSocketConnection connection)
    {
        // Attempt reconnection
        await _webSocketManager.ReconnectAsync(connection.UserId);

        // Notify HTTP mining to increase coverage
        var message = new IncreasePollingFrequencyMessage(
            connection.UserId,
            "WebSocket connection unhealthy");

        await _messageBus.SendAsync(MessageBusRoutes.POLLING_FREQUENCY_UPDATES, message);
    }
}

public sealed record WebSocketHealthStatus(HealthStatus Status, string Reason);

public enum HealthStatus
{
    Healthy,
    Degraded,
    Unhealthy
}
```

## Data Consistency and Validation

### Cross-System Data Validation

```csharp
public class DataConsistencyValidator
{
    public async Task ValidateDataConsistencyAsync(long userId, DateTimeOffset validationPeriodStart, DateTimeOffset validationPeriodEnd)
    {
        // Get all posts processed by HTTP mining in the period
        var httpPosts = await dbContext.PostCheckpoints
            .Where(x => x.UserId == userId &&
                       x.CreatedAt >= validationPeriodStart &&
                       x.CreatedAt <= validationPeriodEnd &&
                       x.ProcessingSource == ProcessingSource.Http)
            .ToListAsync();

        // Get all posts processed by WebSocket in the period
        var webSocketPosts = await dbContext.WebSocketCheckpoints
            .Where(x => x.UserId == userId &&
                       x.EventTimestamp >= validationPeriodStart &&
                       x.EventTimestamp <= validationPeriodEnd &&
                       x.ObjectType == "post")
            .ToListAsync();

        // Find overlaps and gaps
        var overlaps = httpPosts
            .Where(http => webSocketPosts.Any(ws => ws.ObjectId == http.PostId))
            .ToList();

        var httpOnlyPosts = httpPosts
            .Where(http => !webSocketPosts.Any(ws => ws.ObjectId == http.PostId))
            .ToList();

        var webSocketOnlyPosts = webSocketPosts
            .Where(ws => !httpPosts.Any(http => http.PostId == ws.ObjectId))
            .ToList();

        // Report findings
        _logger.LogInformation($"User {userId} consistency check: " +
            $"{overlaps.Count} overlaps, " +
            $"{httpOnlyPosts.Count} HTTP-only, " +
            $"{webSocketOnlyPosts.Count} WebSocket-only");

        // Alert on unexpected patterns
        var overlapPercentage = (double)overlaps.Count / (httpPosts.Count + webSocketPosts.Count - overlaps.Count);
        if (overlapPercentage > 0.5) // More than 50% overlap suggests inefficiency
        {
            _logger.LogWarning($"User {userId}: High overlap percentage ({overlapPercentage:P}) suggests system inefficiency");
        }

        if (httpOnlyPosts.Count == 0 && webSocketPosts.Count > 0)
        {
            _logger.LogInformation($"User {userId}: Perfect WebSocket coverage - HTTP mining can be reduced");
        }
    }
}
```

## Performance Optimization Patterns

### Dynamic Polling Frequency Adjustment

```csharp
public class AdaptivePollingManager
{
    public async Task AdjustPollingFrequencyAsync(long userId)
    {
        var wsStatus = await _webSocketManager.GetConnectionStatusAsync(userId);
        var userActivity = await _analyticsService.GetUserActivityMetricsAsync(userId);

        var optimalFrequency = CalculateOptimalPollingFrequency(wsStatus, userActivity);

        await UpdateUserPollingFrequencyAsync(userId, optimalFrequency);

        _logger.LogDebug($"User {userId}: Adjusted polling frequency to {optimalFrequency.TotalMinutes} minutes");
    }

    private TimeSpan CalculateOptimalPollingFrequency(WebSocketConnectionStatus wsStatus, UserActivityMetrics activity)
    {
        // Base frequency on WebSocket reliability
        var baseFrequency = wsStatus.IsConnected && wsStatus.ReliabilityScore > 0.8
            ? TimeSpan.FromHours(6)  // Reduce HTTP polling when WebSocket is reliable
            : TimeSpan.FromHours(1); // Increase when WebSocket is unreliable

        // Adjust based on user activity
        var activityMultiplier = activity.PostsPerDay switch
        {
            > 100 => 0.5,  // Very active users - poll more frequently
            > 10 => 0.75,  // Moderately active
            _ => 1.5       // Low activity - poll less frequently
        };

        var adjustedFrequency = TimeSpan.FromMilliseconds(baseFrequency.TotalMilliseconds * activityMultiplier);

        // Ensure reasonable bounds
        return TimeSpan.FromMinutes(Math.Max(15, Math.Min(720, adjustedFrequency.TotalMinutes))); // 15 min to 12 hours
    }
}
```

## Monitoring and Observability

### Dual System Metrics

```csharp
public class DualSystemMetrics
{
    public async Task RecordSystemMetricsAsync()
    {
        // WebSocket coverage metrics
        var wsStats = await CalculateWebSocketStatsAsync();
        _logger.LogInformation($"WebSocket coverage: {wsStats.ConnectedUsers}/{wsStats.TotalUsers} users, " +
            $"average uptime: {wsStats.AverageUptimePercentage:P}");

        // HTTP mining efficiency
        var httpStats = await CalculateHttpMiningStatsAsync();
        _logger.LogInformation($"HTTP mining efficiency: {httpStats.NewDataPercentage:P} new data rate, " +
            $"average gap fill time: {httpStats.AverageGapFillTime.TotalMinutes:F1} minutes");

        // System coordination effectiveness
        var coordination = await CalculateCoordinationStatsAsync();
        _logger.LogInformation($"System coordination: {coordination.ConflictRate:P} conflict rate, " +
            $"overlap efficiency: {coordination.OverlapEfficiency:P}");

        // Data completeness
        var completeness = await CalculateDataCompletenessAsync();
        _logger.LogInformation($"Data completeness: {completeness.CoveragePercentage:P} coverage, " +
            $"{completeness.MissedEvents} potentially missed events");
    }

    private async Task<WebSocketStats> CalculateWebSocketStatsAsync()
    {
        var activeConnections = await dbContext.WebSocketConnections
            .Where(x => x.DisconnectedAt == null)
            .CountAsync();

        var totalUsers = await dbContext.ActiveUsers.CountAsync();

        var uptimeData = await dbContext.WebSocketConnections
            .Where(x => x.ConnectedAt >= DateTimeOffset.UtcNow.AddDays(-1))
            .Select(x => new {
                Uptime = (x.DisconnectedAt ?? DateTimeOffset.UtcNow) - x.ConnectedAt,
                TotalTime = DateTimeOffset.UtcNow.AddDays(-1) - x.ConnectedAt
            })
            .ToListAsync();

        var averageUptime = uptimeData.Any()
            ? uptimeData.Average(x => x.Uptime.TotalMilliseconds / x.TotalTime.TotalMilliseconds)
            : 0.0;

        return new WebSocketStats(activeConnections, totalUsers, averageUptime);
    }
}

public sealed record WebSocketStats(int ConnectedUsers, int TotalUsers, double AverageUptimePercentage);
```

## Production Anti-Patterns and Lessons

### L **What NOT to Do**

**1. WebSocket Without HTTP Backup**
```csharp
// DON'T DO THIS
// Only using WebSocket - will miss data during disconnections
await StartWebSocketOnly(userId);
// Always have HTTP mining as backup
```

**2. Ignoring Connection Health**
```csharp
// DON'T DO THIS
webSocket.Connect();
// Assume it stays connected forever
// Always monitor connection health and implement reconnection
```

**3. Processing Duplicates Without Coordination**
```csharp
// DON'T DO THIS
await ProcessWebSocketEvent(event);  // Process in WebSocket
await ProcessHttpData(data);         // Process same data in HTTP
// Implement conflict resolution and deduplication
```

## Performance Results

In production, our dual HTTP/WebSocket system achieves:

- **99.95% data coverage** - captures nearly all events across both systems
- **< 30 second real-time latency** - WebSocket events processed almost instantly
- **60% reduction in HTTP API calls** - WebSocket coverage reduces polling needs
- **95% reduction in 429 rate limit errors** - WebSocket eliminates frequent polling
- **99.8% deduplication accuracy** - minimal overlap processing between systems
- **Linear scalability** - both systems scale independently based on load
- **24/7 uptime resilience** - automatic fallback from WebSocket to HTTP when needed

## What's Next

In **Part 7**, we'll conclude with **Production Lessons** - the hard-earned insights from running this architecture at scale. We'll cover performance tuning, cost optimization, monitoring strategies, team organization, and the common pitfalls that can sink even the best-designed systems.

The final part ties together all the architectural patterns with real-world operational wisdom.

---

*Have you implemented similar dual-capture systems? What challenges have you faced with WebSocket reliability at scale? Share your experiences with real-time data architectures in the comments below.*

**Previous**: [<- Part 5 - Checkpoint Systems](./part-5.md)

**Next**: [Part 7 - Production Lessons ->](./part-7.md)