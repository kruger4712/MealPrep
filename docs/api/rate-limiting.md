# API Rate Limiting Guide

## Overview
Comprehensive guide for API rate limiting in the MealPrep application, covering rate limit policies, implementation strategies, monitoring, and best practices for managing API usage across different user tiers and endpoints.

## Rate Limiting Strategy

### Rate Limiting Philosophy
- **Fair Usage**: Ensure equal access to API resources for all users
- **Service Protection**: Prevent abuse and maintain system stability
- **Tiered Access**: Provide different limits based on subscription levels
- **Gradual Enforcement**: Implement progressive restrictions with warnings
- **Transparent Communication**: Clear rate limit information in responses

### Rate Limiting Dimensions
```yaml
Rate Limiting Dimensions:
  By User:
    - Individual user quotas
    - Subscription tier limits
    - Account status considerations
    
  By Endpoint:
    - Endpoint-specific limits
    - Resource-intensive operations
    - AI service usage controls
    
  By Time Window:
    - Per minute limits (burst protection)
    - Per hour limits (sustained usage)
    - Per day limits (quota management)
    
  By Client:
    - IP address throttling
    - API key restrictions
    - Device-based limits
```

---

## Rate Limit Policies

### User Tier-Based Limits

#### Free Tier Users
```yaml
Free Tier Limits:
  General API:
    requests_per_minute: 10
    requests_per_hour: 200
    requests_per_day: 1000
    
  Recipe Operations:
    create_per_day: 5
    update_per_day: 10
    delete_per_day: 5
    search_per_minute: 5
    
  AI Services:
    suggestions_per_hour: 3
    suggestions_per_day: 10
    persona_operations_per_day: 2
    
  Family Management:
    members_per_family: 4
    updates_per_day: 10
    
  Menu Planning:
    plans_per_day: 2
    plan_updates_per_day: 5
```

#### Premium Tier Users
```yaml
Premium Tier Limits:
  General API:
    requests_per_minute: 50
    requests_per_hour: 2000
    requests_per_day: 20000
    
  Recipe Operations:
    create_per_day: 50
    update_per_day: 100
    delete_per_day: 25
    search_per_minute: 20
    bulk_import_per_day: 3
    
  AI Services:
    suggestions_per_hour: 25
    suggestions_per_day: 100
    persona_operations_per_day: 20
    custom_personas_per_family: 10
    
  Family Management:
    members_per_family: 10
    updates_per_day: 50
    
  Menu Planning:
    plans_per_day: 20
    plan_updates_per_day: 50
    advanced_planning_per_day: 10
```

#### Enterprise Tier Users
```yaml
Enterprise Tier Limits:
  General API:
    requests_per_minute: 200
    requests_per_hour: 10000
    requests_per_day: 100000
    
  Recipe Operations:
    create_per_day: 500
    update_per_day: 1000
    delete_per_day: 250
    search_per_minute: 100
    bulk_import_per_day: 20
    
  AI Services:
    suggestions_per_hour: 100
    suggestions_per_day: 500
    persona_operations_per_day: 100
    custom_personas_per_family: 50
    
  Family Management:
    members_per_family: 50
    updates_per_day: 500
    
  Menu Planning:
    plans_per_day: 100
    plan_updates_per_day: 500
    advanced_planning_per_day: 50
```

### Endpoint-Specific Rate Limits

#### Authentication Endpoints
```yaml
Authentication Rate Limits:
  /auth/login:
    global_per_ip_per_minute: 5
    global_per_ip_per_hour: 20
    per_email_per_hour: 10
    lockout_threshold: 5
    lockout_duration: 300  # 5 minutes
    
  /auth/register:
    global_per_ip_per_minute: 2
    global_per_ip_per_hour: 10
    global_per_ip_per_day: 20
    
  /auth/refresh:
    per_user_per_minute: 5
    per_user_per_hour: 60
    
  /auth/logout:
    per_user_per_minute: 10
    
  /auth/password-reset:
    per_email_per_hour: 3
    per_ip_per_hour: 10
```

#### AI Service Endpoints
```yaml
AI Service Rate Limits:
  /ai/suggestions:
    free_tier_per_hour: 3
    premium_tier_per_hour: 25
    enterprise_tier_per_hour: 100
    concurrent_requests: 2
    timeout_seconds: 30
    
  /ai/personas:
    free_tier_per_day: 2
    premium_tier_per_day: 20
    enterprise_tier_per_day: 100
    
  /ai/bulk-suggestions:
    premium_tier_per_day: 5
    enterprise_tier_per_day: 50
    max_batch_size: 10
    
  /ai/custom-prompts:
    premium_tier_per_day: 10
    enterprise_tier_per_day: 100
```

#### Data-Intensive Endpoints
```yaml
Data Intensive Rate Limits:
  /recipes/search:
    free_tier_per_minute: 5
    premium_tier_per_minute: 20
    enterprise_tier_per_minute: 100
    
  /recipes/export:
    free_tier_per_day: 1
    premium_tier_per_day: 10
    enterprise_tier_per_day: 50
    
  /analytics/reports:
    premium_tier_per_hour: 5
    enterprise_tier_per_hour: 25
    
  /backup/create:
    per_user_per_day: 3
    per_user_per_hour: 1
```

---

## Rate Limiting Implementation

### Rate Limiting Middleware
```csharp
public class RateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IRateLimitService _rateLimitService;
    private readonly IRateLimitConfiguration _configuration;
    private readonly ILogger<RateLimitingMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        var rateLimitContext = await BuildRateLimitContextAsync(context);
        var rateLimitResult = await _rateLimitService.CheckRateLimitAsync(rateLimitContext);

        // Add rate limit headers to response
        AddRateLimitHeaders(context.Response, rateLimitResult);

        if (!rateLimitResult.IsAllowed)
        {
            await HandleRateLimitExceededAsync(context, rateLimitResult);
            return;
        }

        // Record the request
        await _rateLimitService.RecordRequestAsync(rateLimitContext);

        await _next(context);
    }

    private async Task<RateLimitContext> BuildRateLimitContextAsync(HttpContext context)
    {
        var user = context.GetCurrentUser();
        var endpoint = GetNormalizedEndpoint(context.Request.Path);
        var clientIdentifier = GetClientIdentifier(context);

        return new RateLimitContext
        {
            UserId = user?.UserId,
            UserTier = user?.SubscriptionTier ?? UserTier.Free,
            Endpoint = endpoint,
            Method = context.Request.Method,
            ClientIdentifier = clientIdentifier,
            IpAddress = GetClientIpAddress(context),
            UserAgent = context.Request.Headers["User-Agent"].ToString(),
            Timestamp = DateTime.UtcNow
        };
    }

    private void AddRateLimitHeaders(HttpResponse response, RateLimitResult result)
    {
        response.Headers.Add("X-RateLimit-Limit", result.Limit.ToString());
        response.Headers.Add("X-RateLimit-Remaining", result.Remaining.ToString());
        response.Headers.Add("X-RateLimit-Reset", result.ResetTime.ToString());
        response.Headers.Add("X-RateLimit-Reset-After", result.ResetAfter.ToString());

        if (result.RetryAfter.HasValue)
        {
            response.Headers.Add("Retry-After", result.RetryAfter.Value.ToString());
        }
    }

    private async Task HandleRateLimitExceededAsync(HttpContext context, RateLimitResult result)
    {
        context.Response.StatusCode = 429; // Too Many Requests

        var errorResponse = new
        {
            error = new
            {
                code = "RATE_LIMIT_EXCEEDED",
                message = GetRateLimitMessage(result),
                details = $"Rate limit exceeded for {result.LimitType}",
                timestamp = DateTime.UtcNow.ToString("O"),
                requestId = context.TraceIdentifier
            },
            rateLimitInfo = new
            {
                limit = result.Limit,
                remaining = result.Remaining,
                resetTime = result.ResetTime.ToString("O"),
                retryAfter = result.RetryAfter,
                limitType = result.LimitType.ToString(),
                window = result.WindowDescription
            },
            upgradeInfo = GetUpgradeInfo(result)
        };

        context.Response.ContentType = "application/json";
        await context.Response.WriteAsync(JsonSerializer.Serialize(errorResponse));

        // Log rate limit exceeded event
        _logger.LogWarning("Rate limit exceeded: {UserId}, {Endpoint}, {LimitType}", 
            result.UserId, result.Endpoint, result.LimitType);
    }

    private string GetRateLimitMessage(RateLimitResult result)
    {
        return result.LimitType switch
        {
            RateLimitType.PerMinute => "Too many requests per minute. Please slow down.",
            RateLimitType.PerHour => "Hourly request limit exceeded. Please try again later.",
            RateLimitType.PerDay => "Daily request limit exceeded. Limit resets at midnight UTC.",
            RateLimitType.Concurrent => "Too many concurrent requests. Please wait for current requests to complete.",
            RateLimitType.AiQuota => "AI service quota exceeded. Consider upgrading your plan.",
            _ => "Rate limit exceeded. Please try again later."
        };
    }
}
```

### Rate Limit Service Implementation
```csharp
public class RateLimitService : IRateLimitService
{
    private readonly IMemoryCache _cache;
    private readonly IRateLimitConfiguration _configuration;
    private readonly IDistributedCache _distributedCache;

    public async Task<RateLimitResult> CheckRateLimitAsync(RateLimitContext context)
    {
        var limits = await GetApplicableLimitsAsync(context);
        var results = new List<RateLimitResult>();

        foreach (var limit in limits)
        {
            var result = await CheckIndividualLimitAsync(context, limit);
            results.Add(result);

            if (!result.IsAllowed)
            {
                return result; // Return first exceeded limit
            }
        }

        // Return the most restrictive successful result
        return results.OrderBy(r => r.Remaining).First();
    }

    private async Task<RateLimitResult> CheckIndividualLimitAsync(
        RateLimitContext context, 
        RateLimitRule rule)
    {
        var key = BuildRateLimitKey(context, rule);
        var window = GetTimeWindow(rule.WindowType);
        var currentCount = await GetCurrentRequestCountAsync(key, window);

        var isAllowed = currentCount < rule.Limit;
        var remaining = Math.Max(0, rule.Limit - currentCount - 1);
        var resetTime = GetWindowResetTime(rule.WindowType);

        return new RateLimitResult
        {
            IsAllowed = isAllowed,
            Limit = rule.Limit,
            Remaining = remaining,
            ResetTime = resetTime,
            ResetAfter = (int)(resetTime - DateTime.UtcNow).TotalSeconds,
            RetryAfter = isAllowed ? null : (int?)GetRetryAfterSeconds(rule),
            LimitType = rule.Type,
            WindowDescription = GetWindowDescription(rule.WindowType),
            UserId = context.UserId,
            Endpoint = context.Endpoint
        };
    }

    private async Task<int> GetCurrentRequestCountAsync(string key, TimeSpan window)
    {
        // Implementation using Redis sliding window
        var script = @"
            local key = KEYS[1]
            local window = tonumber(ARGV[1])
            local now = tonumber(ARGV[2])
            
            -- Remove expired entries
            redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
            
            -- Count current entries
            local count = redis.call('ZCARD', key)
            
            -- Set expiration
            redis.call('EXPIRE', key, window)
            
            return count
        ";

        var redis = _distributedCache as IRedisCache;
        return await redis.EvaluateAsync<int>(script, 
            new RedisKey[] { key }, 
            new RedisValue[] { window.TotalSeconds, DateTimeOffset.UtcNow.ToUnixTimeSeconds() });
    }

    public async Task RecordRequestAsync(RateLimitContext context)
    {
        var limits = await GetApplicableLimitsAsync(context);

        foreach (var limit in limits)
        {
            var key = BuildRateLimitKey(context, limit);
            await RecordRequestForLimitAsync(key, limit);
        }
    }

    private async Task RecordRequestForLimitAsync(string key, RateLimitRule rule)
    {
        var script = @"
            local key = KEYS[1]
            local window = tonumber(ARGV[1])
            local now = tonumber(ARGV[2])
            local uuid = ARGV[3]
            
            -- Add current request
            redis.call('ZADD', key, now, uuid)
            
            -- Remove expired entries
            redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
            
            -- Set expiration
            redis.call('EXPIRE', key, window)
            
            return redis.call('ZCARD', key)
        ";

        var redis = _distributedCache as IRedisCache;
        await redis.EvaluateAsync(script,
            new RedisKey[] { key },
            new RedisValue[] { 
                GetTimeWindow(rule.WindowType).TotalSeconds,
                DateTimeOffset.UtcNow.ToUnixTimeSeconds(),
                Guid.NewGuid().ToString()
            });
    }
}
```

### Rate Limit Configuration
```csharp
public class RateLimitConfiguration : IRateLimitConfiguration
{
    public async Task<List<RateLimitRule>> GetApplicableLimitsAsync(RateLimitContext context)
    {
        var rules = new List<RateLimitRule>();

        // Global IP-based limits
        rules.AddRange(GetGlobalIpLimits(context));

        // User-based limits
        if (context.UserId.HasValue)
        {
            rules.AddRange(GetUserLimits(context));
        }

        // Endpoint-specific limits
        rules.AddRange(GetEndpointLimits(context));

        // Special handling for sensitive endpoints
        if (IsSensitiveEndpoint(context.Endpoint))
        {
            rules.AddRange(GetSensitiveEndpointLimits(context));
        }

        return rules;
    }

    private List<RateLimitRule> GetUserLimits(RateLimitContext context)
    {
        var limits = context.UserTier switch
        {
            UserTier.Free => GetFreeTierLimits(),
            UserTier.Premium => GetPremiumTierLimits(),
            UserTier.Enterprise => GetEnterpriseTierLimits(),
            _ => GetFreeTierLimits()
        };

        return limits.Select(l => l with { 
            Key = $"user:{context.UserId}:{l.Key}" 
        }).ToList();
    }

    private List<RateLimitRule> GetFreeTierLimits()
    {
        return new List<RateLimitRule>
        {
            new RateLimitRule
            {
                Key = "general",
                Type = RateLimitType.PerMinute,
                WindowType = TimeWindowType.Minute,
                Limit = 10,
                Description = "General API requests per minute"
            },
            new RateLimitRule
            {
                Key = "general",
                Type = RateLimitType.PerHour,
                WindowType = TimeWindowType.Hour,
                Limit = 200,
                Description = "General API requests per hour"
            },
            new RateLimitRule
            {
                Key = "ai",
                Type = RateLimitType.AiQuota,
                WindowType = TimeWindowType.Hour,
                Limit = 3,
                Description = "AI suggestions per hour"
            }
        };
    }

    private List<RateLimitRule> GetEndpointLimits(RateLimitContext context)
    {
        return context.Endpoint switch
        {
            "/auth/login" => GetAuthLoginLimits(context),
            "/auth/register" => GetAuthRegisterLimits(context),
            "/ai/suggestions" => GetAiSuggestionLimits(context),
            "/recipes/search" => GetRecipeSearchLimits(context),
            _ => new List<RateLimitRule>()
        };
    }

    private List<RateLimitRule> GetAuthLoginLimits(RateLimitContext context)
    {
        return new List<RateLimitRule>
        {
            new RateLimitRule
            {
                Key = $"auth:login:ip:{context.IpAddress}",
                Type = RateLimitType.PerMinute,
                WindowType = TimeWindowType.Minute,
                Limit = 5,
                Description = "Login attempts per IP per minute"
            },
            new RateLimitRule
            {
                Key = $"auth:login:ip:{context.IpAddress}",
                Type = RateLimitType.PerHour,
                WindowType = TimeWindowType.Hour,
                Limit = 20,
                Description = "Login attempts per IP per hour"
            }
        };
    }
}
```

---

## Rate Limit Headers

### Standard Headers
All API responses include rate limiting information in HTTP headers:

```http
HTTP/1.1 200 OK
Content-Type: application/json

# Current rate limit for the user/endpoint
X-RateLimit-Limit: 1000

# Remaining requests in current window
X-RateLimit-Remaining: 999

# Unix timestamp when the rate limit resets
X-RateLimit-Reset: 1640995200

# Seconds until rate limit resets
X-RateLimit-Reset-After: 3600

# Rate limit window type (minute, hour, day)
X-RateLimit-Window: hour

# User's subscription tier
X-RateLimit-Tier: premium
```

### Rate Limit Exceeded Headers
When rate limit is exceeded (HTTP 429):

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json

# Standard rate limit headers (all showing 0 remaining)
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640995200
X-RateLimit-Reset-After: 3600

# Retry guidance
Retry-After: 3600

# Type of limit that was exceeded
X-RateLimit-Limit-Type: hourly

# Upgrade information
X-RateLimit-Upgrade-Available: true
```

---

## Rate Limit Monitoring

### Monitoring Dashboard
```typescript
interface RateLimitMetrics {
  totalRequests: number;
  rateLimitedRequests: number;
  rateLimitedPercentage: number;
  topRateLimitedEndpoints: Array<{
    endpoint: string;
    count: number;
    percentage: number;
  }>;
  topRateLimitedUsers: Array<{
    userId: string;
    tier: string;
    count: number;
  }>;
  rateLimitTrends: Array<{
    timestamp: string;
    requests: number;
    rateLimited: number;
  }>;
}

export class RateLimitMonitoringService {
  async collectRateLimitMetrics(timeRange: TimeRange): Promise<RateLimitMetrics> {
    const metrics = await Promise.all([
      this.getTotalRequests(timeRange),
      this.getRateLimitedRequests(timeRange),
      this.getTopRateLimitedEndpoints(timeRange),
      this.getTopRateLimitedUsers(timeRange),
      this.getRateLimitTrends(timeRange)
    ]);

    return {
      totalRequests: metrics[0],
      rateLimitedRequests: metrics[1],
      rateLimitedPercentage: (metrics[1] / metrics[0]) * 100,
      topRateLimitedEndpoints: metrics[2],
      topRateLimitedUsers: metrics[3],
      rateLimitTrends: metrics[4]
    };
  }

  async generateRateLimitReport(): Promise<RateLimitReport> {
    const last24Hours = await this.collectRateLimitMetrics({ hours: 24 });
    const last7Days = await this.collectRateLimitMetrics({ days: 7 });
    const last30Days = await this.collectRateLimitMetrics({ days: 30 });

    return {
      summary: {
        current24h: last24Hours,
        current7d: last7Days,
        current30d: last30Days
      },
      recommendations: this.generateRecommendations(last24Hours, last7Days),
      alerts: await this.checkForAlerts(last24Hours)
    };
  }

  private generateRecommendations(
    recent: RateLimitMetrics, 
    weekly: RateLimitMetrics
  ): string[] {
    const recommendations: string[] = [];

    if (recent.rateLimitedPercentage > 5) {
      recommendations.push("High rate limiting detected. Consider reviewing limits or scaling resources.");
    }

    if (recent.topRateLimitedEndpoints.length > 0) {
      const topEndpoint = recent.topRateLimitedEndpoints[0];
      recommendations.push(`Endpoint ${topEndpoint.endpoint} has high rate limiting. Review limits or optimize endpoint.`);
    }

    if (weekly.rateLimitedPercentage > recent.rateLimitedPercentage * 1.5) {
      recommendations.push("Rate limiting increasing over time. Monitor user growth and adjust limits accordingly.");
    }

    return recommendations;
  }
}
```

### Alert Configuration
```yaml
Rate Limit Alerts:
  High Rate Limiting:
    condition: "rate_limited_percentage > 10%"
    severity: "warning"
    notification: "slack, email"
    description: "More than 10% of requests are being rate limited"
    
  Excessive AI Usage:
    condition: "ai_requests_per_hour > ai_limit * 0.9"
    severity: "info"
    notification: "slack"
    description: "AI usage approaching limit"
    
  Authentication Attacks:
    condition: "failed_logins_per_ip > 20"
    severity: "critical"
    notification: "slack, email, pagerduty"
    description: "Potential brute force attack detected"
    
  Endpoint Overload:
    condition: "endpoint_rate_limited > 50% for 5 minutes"
    severity: "critical"
    notification: "slack, email"
    description: "Specific endpoint experiencing high rate limiting"
```

---

## Client-Side Rate Limit Handling

### JavaScript Client Implementation
```typescript
export class RateLimitAwareApiClient {
    private requestQueue: Map<string, Date> = new Map();
    private retryDelays: Map<string, number> = new Map();

    async makeRequest<T>(
        endpoint: string, 
        options: RequestInit = {}
    ): Promise<T> {
        // Check if we should delay this request
        await this.enforceClientSideRateLimit(endpoint);

        try {
            const response = await fetch(endpoint, {
                ...options,
                headers: {
                    ...options.headers,
                    'X-Client-Rate-Limit-Aware': 'true'
                }
            });

            // Update rate limit info from headers
            this.updateRateLimitInfo(endpoint, response);

            if (response.status === 429) {
                return this.handleRateLimitExceeded(endpoint, response, options);
            }

            if (!response.ok) {
                throw new Error(`Request failed: ${response.status}`);
            }

            // Reset retry delay on success
            this.retryDelays.delete(endpoint);

            return await response.json();
        } catch (error) {
            throw error;
        }
    }

    private async handleRateLimitExceeded<T>(
        endpoint: string,
        response: Response,
        originalOptions: RequestInit
    ): Promise<T> {
        const retryAfter = parseInt(response.headers.get('Retry-After') || '60');
        const rateLimitType = response.headers.get('X-RateLimit-Limit-Type');
        
        // Parse error response for additional info
        const errorData = await response.json();
        
        console.warn(`Rate limit exceeded for ${endpoint}:`, {
            retryAfter,
            limitType: rateLimitType,
            upgradeAvailable: errorData.upgradeInfo?.available
        });

        // Show user-friendly notification
        this.showRateLimitNotification(errorData);

        // Implement exponential backoff
        const backoffDelay = this.calculateBackoffDelay(endpoint, retryAfter);
        
        await this.delay(backoffDelay * 1000);
        
        // Retry the request
        return this.makeRequest(endpoint, originalOptions);
    }

    private calculateBackoffDelay(endpoint: string, retryAfter: number): number {
        const previousDelay = this.retryDelays.get(endpoint) || 1;
        const newDelay = Math.min(previousDelay * 2, retryAfter);
        this.retryDelays.set(endpoint, newDelay);
        return newDelay;
    }

    private showRateLimitNotification(errorData: any): void {
        const message = errorData.error?.message || 'Rate limit exceeded';
        const upgradeInfo = errorData.upgradeInfo;

        if (upgradeInfo?.available) {
            // Show upgrade prompt
            this.showUpgradePrompt(upgradeInfo);
        } else {
            // Show rate limit notification
            this.showTemporaryNotification(message);
        }
    }

    private async enforceClientSideRateLimit(endpoint: string): Promise<void> {
        const lastRequest = this.requestQueue.get(endpoint);
        if (lastRequest) {
            const timeSinceLastRequest = Date.now() - lastRequest.getTime();
            const minInterval = this.getMinRequestInterval(endpoint);
            
            if (timeSinceLastRequest < minInterval) {
                const delayNeeded = minInterval - timeSinceLastRequest;
                await this.delay(delayNeeded);
            }
        }
        
        this.requestQueue.set(endpoint, new Date());
    }

    private getMinRequestInterval(endpoint: string): number {
        // Implement client-side rate limiting to reduce server load
        if (endpoint.includes('/ai/')) {
            return 2000; // 2 seconds between AI requests
        }
        if (endpoint.includes('/auth/')) {
            return 1000; // 1 second between auth requests
        }
        return 100; // 100ms for other endpoints
    }
}
```

### React Hook for Rate Limit Handling
```typescript
export function useRateLimitAwareApi() {
    const [rateLimitInfo, setRateLimitInfo] = useState<RateLimitInfo | null>(null);
    const [isRateLimited, setIsRateLimited] = useState(false);

    const makeRequest = useCallback(async <T>(
        endpoint: string,
        options?: RequestInit
    ): Promise<T> => {
        try {
            setIsRateLimited(false);
            const client = new RateLimitAwareApiClient();
            const result = await client.makeRequest<T>(endpoint, options);
            
            // Update rate limit info from response headers
            setRateLimitInfo(client.getLastRateLimitInfo());
            
            return result;
        } catch (error) {
            if (error instanceof RateLimitError) {
                setIsRateLimited(true);
                setRateLimitInfo(error.rateLimitInfo);
            }
            throw error;
        }
    }, []);

    return {
        makeRequest,
        rateLimitInfo,
        isRateLimited,
        canMakeRequest: rateLimitInfo?.remaining > 0
    };
}

// Usage in React component
function RecipeSearch() {
    const { makeRequest, rateLimitInfo, isRateLimited } = useRateLimitAwareApi();

    const searchRecipes = async (searchTerm: string) => {
        try {
            const results = await makeRequest('/api/recipes/search', {
                method: 'POST',
                body: JSON.stringify({ searchTerm })
            });
            setSearchResults(results);
        } catch (error) {
            // Handle error
        }
    };

    return (
        <div>
            {rateLimitInfo && (
                <div className="rate-limit-info">
                    {rateLimitInfo.remaining} requests remaining this hour
                </div>
            )}
            
            {isRateLimited && (
                <div className="rate-limit-warning">
                    Rate limit exceeded. Please wait before making more requests.
                </div>
            )}
            
            <SearchInput 
                onSearch={searchRecipes}
                disabled={isRateLimited || rateLimitInfo?.remaining === 0}
            />
        </div>
    );
}
```

---

## Best Practices and Recommendations

### For API Consumers
1. **Respect Rate Limits**: Always check rate limit headers before making requests
2. **Implement Backoff**: Use exponential backoff when rate limited
3. **Cache Responses**: Reduce API calls by caching responses locally
4. **Batch Requests**: Use batch endpoints when available
5. **Monitor Usage**: Track your API usage patterns

### For API Providers
1. **Clear Documentation**: Provide comprehensive rate limit documentation
2. **Gradual Enforcement**: Implement warnings before hard limits
3. **Flexible Limits**: Offer different tiers for different use cases
4. **Monitoring Tools**: Provide dashboards for users to monitor usage
5. **Grace Periods**: Allow temporary bursts for legitimate use cases

### Security Considerations
1. **IP-Based Limits**: Protect against distributed attacks
2. **User-Based Limits**: Prevent individual account abuse
3. **Endpoint-Specific Limits**: Protect resource-intensive operations
4. **Geographic Restrictions**: Consider regional rate limiting
5. **Anomaly Detection**: Monitor for unusual usage patterns

---

*Last Updated: December 2024*  
*Rate limiting guide continuously updated with new strategies and best practices*