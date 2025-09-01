# API Security

## Overview
Comprehensive API security implementation guide for the MealPrep application, covering authentication, authorization, input validation, rate limiting, data protection, and security monitoring for REST APIs and real-time communications.

## API Security Architecture

### Security Layers
```
???????????????????????????????????????????????????????????????
?                     Client Application                      ?
???????????????????????????????????????????????????????????????
                      ? HTTPS/TLS 1.3
???????????????????????????????????????????????????????????????
?                   Load Balancer/CDN                        ?
?                  • DDoS Protection                         ?
?                  • Geographic Filtering                    ?
???????????????????????????????????????????????????????????????
                      ?
???????????????????????????????????????????????????????????????
?                   API Gateway                              ?
?                  • Rate Limiting                           ?
?                  • Request Validation                      ?
?                  • API Key Management                      ?
???????????????????????????????????????????????????????????????
                      ?
???????????????????????????????????????????????????????????????
?                  MealPrep API                              ?
?                  • JWT Authentication                       ?
?                  • RBAC Authorization                      ?
?                  • Input Sanitization                      ?
?                  • Output Filtering                        ?
???????????????????????????????????????????????????????????????
                      ?
???????????????????????????????????????????????????????????????
?                    Database                                ?
?                  • Row-Level Security                      ?
?                  • Encrypted at Rest                       ?
?                  • Audit Logging                           ?
???????????????????????????????????????????????????????????????
```

## Authentication Security

### JWT Token Security Implementation
```csharp
// Security/JwtSecurityService.cs
public class JwtSecurityService : IJwtSecurityService
{
    private readonly IConfiguration _configuration;
    private readonly ITokenBlacklistService _blacklistService;
    private readonly ISecurityEventLogger _securityLogger;

    public async Task<JwtSecurityValidationResult> ValidateTokenSecurityAsync(
        ClaimsPrincipal principal, 
        SecurityToken validatedToken)
    {
        var result = new JwtSecurityValidationResult();
        
        try
        {
            // 1. Check token blacklist
            var jti = principal.FindFirst(JwtRegisteredClaimNames.Jti)?.Value;
            if (!string.IsNullOrEmpty(jti))
            {
                var isBlacklisted = await _blacklistService.IsTokenBlacklistedAsync(jti);
                if (isBlacklisted)
                {
                    result.IsValid = false;
                    result.FailureReason = "Token has been revoked";
                    await _securityLogger.LogSecurityEventAsync(SecurityEventType.RevokedTokenUsed, 
                        new { TokenId = jti });
                    return result;
                }
            }

            // 2. Validate token timing
            var tokenValidation = ValidateTokenTiming(validatedToken);
            if (!tokenValidation.IsValid)
            {
                result.IsValid = false;
                result.FailureReason = tokenValidation.Reason;
                return result;
            }

            // 3. Check for suspicious patterns
            var suspiciousCheck = await CheckSuspiciousActivityAsync(principal);
            if (suspiciousCheck.IsSuspicious)
            {
                result.IsValid = false;
                result.FailureReason = "Suspicious activity detected";
                result.RequiresAdditionalVerification = true;
                await _securityLogger.LogSecurityEventAsync(SecurityEventType.SuspiciousTokenUsage, 
                    suspiciousCheck.Details);
                return result;
            }

            // 4. Validate family context
            var familyValidation = await ValidateFamilyContextAsync(principal);
            if (!familyValidation.IsValid)
            {
                result.IsValid = false;
                result.FailureReason = "Family context validation failed";
                return result;
            }

            result.IsValid = true;
            return result;
        }
        catch (Exception ex)
        {
            await _securityLogger.LogSecurityEventAsync(SecurityEventType.TokenValidationError, 
                new { Error = ex.Message });
            result.IsValid = false;
            result.FailureReason = "Token validation error";
            return result;
        }
    }

    private TokenTimingValidation ValidateTokenTiming(SecurityToken token)
    {
        var jwtToken = token as JwtSecurityToken;
        if (jwtToken == null)
        {
            return new TokenTimingValidation { IsValid = false, Reason = "Invalid token format" };
        }

        // Check issued time (not too far in past or future)
        var issuedAt = jwtToken.IssuedAt;
        var now = DateTime.UtcNow;
        var maxAge = TimeSpan.FromHours(24); // Maximum token age
        var clockSkewTolerance = TimeSpan.FromMinutes(5);

        if (issuedAt > now.Add(clockSkewTolerance))
        {
            return new TokenTimingValidation { IsValid = false, Reason = "Token issued in future" };
        }

        if (issuedAt < now.Subtract(maxAge))
        {
            return new TokenTimingValidation { IsValid = false, Reason = "Token too old" };
        }

        return new TokenTimingValidation { IsValid = true };
    }

    private async Task<SuspiciousActivityCheck> CheckSuspiciousActivityAsync(ClaimsPrincipal principal)
    {
        var userId = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        var deviceId = principal.FindFirst("device_id")?.Value;
        
        if (string.IsNullOrEmpty(userId) || string.IsNullOrEmpty(deviceId))
        {
            return new SuspiciousActivityCheck 
            { 
                IsSuspicious = true, 
                Details = new { Reason = "Missing user or device information" }
            };
        }

        // Check for multiple concurrent sessions from different devices
        var activeSessions = await GetActiveSessionsAsync(userId);
        if (activeSessions.Count(s => s.DeviceId != deviceId) > 3)
        {
            return new SuspiciousActivityCheck 
            { 
                IsSuspicious = true, 
                Details = new { Reason = "Too many concurrent sessions", SessionCount = activeSessions.Count }
            };
        }

        // Check for unusual geographic activity
        var currentLocation = await GetLocationFromRequestAsync();
        var recentLocations = await GetRecentLocationsAsync(userId);
        
        if (recentLocations.Any() && IsUnusualGeographicActivity(currentLocation, recentLocations))
        {
            return new SuspiciousActivityCheck 
            { 
                IsSuspicious = true, 
                Details = new { Reason = "Unusual geographic activity", CurrentLocation = currentLocation }
            };
        }

        return new SuspiciousActivityCheck { IsSuspicious = false };
    }
}
```

### API Key Management
```csharp
// Security/ApiKeyService.cs
public class ApiKeyService : IApiKeyService
{
    private readonly IApiKeyRepository _repository;
    private readonly ISecurityEventLogger _securityLogger;

    public async Task<ApiKeyValidationResult> ValidateApiKeyAsync(string apiKey, string endpoint)
    {
        var result = new ApiKeyValidationResult();
        
        try
        {
            // Hash the API key for lookup
            var hashedKey = HashApiKey(apiKey);
            var keyRecord = await _repository.GetApiKeyAsync(hashedKey);
            
            if (keyRecord == null)
            {
                result.IsValid = false;
                result.Reason = "Invalid API key";
                await _securityLogger.LogSecurityEventAsync(SecurityEventType.InvalidApiKey, 
                    new { HashedKey = hashedKey, Endpoint = endpoint });
                return result;
            }

            // Check if key is active
            if (!keyRecord.IsActive || keyRecord.ExpiresAt < DateTime.UtcNow)
            {
                result.IsValid = false;
                result.Reason = "API key expired or inactive";
                return result;
            }

            // Check rate limits
            var rateLimitCheck = await CheckApiKeyRateLimitAsync(keyRecord.Id, endpoint);
            if (!rateLimitCheck.IsWithinLimit)
            {
                result.IsValid = false;
                result.Reason = "API key rate limit exceeded";
                result.RetryAfter = rateLimitCheck.RetryAfter;
                return result;
            }

            // Check endpoint permissions
            if (!HasEndpointPermission(keyRecord, endpoint))
            {
                result.IsValid = false;
                result.Reason = "Insufficient permissions for endpoint";
                await _securityLogger.LogSecurityEventAsync(SecurityEventType.InsufficientApiKeyPermissions, 
                    new { KeyId = keyRecord.Id, Endpoint = endpoint });
                return result;
            }

            // Update usage statistics
            await UpdateApiKeyUsageAsync(keyRecord.Id, endpoint);

            result.IsValid = true;
            result.KeyRecord = keyRecord;
            return result;
        }
        catch (Exception ex)
        {
            await _securityLogger.LogSecurityEventAsync(SecurityEventType.ApiKeyValidationError, 
                new { Error = ex.Message, Endpoint = endpoint });
            
            result.IsValid = false;
            result.Reason = "API key validation error";
            return result;
        }
    }

    private string HashApiKey(string apiKey)
    {
        using var sha256 = SHA256.Create();
        var hashedBytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(apiKey));
        return Convert.ToBase64String(hashedBytes);
    }

    private bool HasEndpointPermission(ApiKey keyRecord, string endpoint)
    {
        // Check if API key has permission for specific endpoint
        if (keyRecord.Permissions == null || !keyRecord.Permissions.Any())
        {
            return false; // Default deny
        }

        return keyRecord.Permissions.Any(permission => 
            endpoint.StartsWith(permission.EndpointPattern, StringComparison.OrdinalIgnoreCase));
    }
}
```

## Input Validation and Sanitization

### Request Validation Middleware
```csharp
// Middleware/InputValidationMiddleware.cs
public class InputValidationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IInputValidator _validator;
    private readonly ISecurityEventLogger _securityLogger;

    public async Task InvokeAsync(HttpContext context)
    {
        // Skip validation for certain endpoints
        if (ShouldSkipValidation(context.Request.Path))
        {
            await _next(context);
            return;
        }

        // Validate request headers
        var headerValidation = await ValidateRequestHeadersAsync(context.Request);
        if (!headerValidation.IsValid)
        {
            await HandleValidationFailureAsync(context, headerValidation.ErrorMessage);
            return;
        }

        // Validate and sanitize request body
        if (context.Request.ContentLength > 0)
        {
            var bodyValidation = await ValidateRequestBodyAsync(context);
            if (!bodyValidation.IsValid)
            {
                await HandleValidationFailureAsync(context, bodyValidation.ErrorMessage);
                return;
            }
        }

        // Validate query parameters
        var queryValidation = ValidateQueryParameters(context.Request.Query);
        if (!queryValidation.IsValid)
        {
            await HandleValidationFailureAsync(context, queryValidation.ErrorMessage);
            return;
        }

        await _next(context);
    }

    private async Task<ValidationResult> ValidateRequestBodyAsync(HttpContext context)
    {
        try
        {
            // Read and validate body size
            if (context.Request.ContentLength > MAX_BODY_SIZE)
            {
                return new ValidationResult 
                { 
                    IsValid = false, 
                    ErrorMessage = "Request body too large" 
                };
            }

            // Read body content
            var body = await ReadRequestBodyAsync(context.Request);
            
            // Validate JSON structure if applicable
            if (context.Request.ContentType?.Contains("application/json") == true)
            {
                var jsonValidation = ValidateJsonStructure(body);
                if (!jsonValidation.IsValid)
                {
                    return jsonValidation;
                }
            }

            // Check for malicious patterns
            var securityValidation = await ValidateForSecurityThreatsAsync(body);
            if (!securityValidation.IsValid)
            {
                await _securityLogger.LogSecurityEventAsync(SecurityEventType.MaliciousInputDetected, 
                    new { Body = body.Substring(0, Math.Min(body.Length, 500)) });
                return securityValidation;
            }

            return new ValidationResult { IsValid = true };
        }
        catch (Exception ex)
        {
            await _securityLogger.LogSecurityEventAsync(SecurityEventType.InputValidationError, 
                new { Error = ex.Message });
            
            return new ValidationResult 
            { 
                IsValid = false, 
                ErrorMessage = "Request validation failed" 
            };
        }
    }

    private async Task<ValidationResult> ValidateForSecurityThreatsAsync(string input)
    {
        // SQL Injection patterns
        var sqlInjectionPatterns = new[]
        {
            @"(\b(SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|EXEC|EXECUTE)\b)",
            @"(UNION\s+SELECT)",
            @"(;\s*DROP\s+TABLE)",
            @"(OR\s+1\s*=\s*1)",
            @"(';\s*--)",
        };

        foreach (var pattern in sqlInjectionPatterns)
        {
            if (Regex.IsMatch(input, pattern, RegexOptions.IgnoreCase))
            {
                return new ValidationResult 
                { 
                    IsValid = false, 
                    ErrorMessage = "Potential SQL injection detected" 
                };
            }
        }

        // XSS patterns
        var xssPatterns = new[]
        {
            @"<script[^>]*>.*?</script>",
            @"javascript:",
            @"on\w+\s*=",
            @"<iframe[^>]*>",
            @"<object[^>]*>",
        };

        foreach (var pattern in xssPatterns)
        {
            if (Regex.IsMatch(input, pattern, RegexOptions.IgnoreCase))
            {
                return new ValidationResult 
                { 
                    IsValid = false, 
                    ErrorMessage = "Potential XSS attack detected" 
                };
            }
        }

        // Path traversal patterns
        var pathTraversalPatterns = new[]
        {
            @"\.\./",
            @"\.\.\\",
            @"%2e%2e%2f",
            @"%2e%2e\\",
        };

        foreach (var pattern in pathTraversalPatterns)
        {
            if (input.Contains(pattern, StringComparison.OrdinalIgnoreCase))
            {
                return new ValidationResult 
                { 
                    IsValid = false, 
                    ErrorMessage = "Potential path traversal detected" 
                };
            }
        }

        return new ValidationResult { IsValid = true };
    }
}

// Model validation with custom attributes
[AttributeUsage(AttributeTargets.Property)]
public class NoMaliciousContentAttribute : ValidationAttribute
{
    public override bool IsValid(object value)
    {
        if (value is string stringValue)
        {
            // Check for suspicious patterns
            var suspiciousPatterns = new[]
            {
                "<script", "javascript:", "onload=", "onerror=",
                "SELECT ", "INSERT ", "UPDATE ", "DELETE ",
                "../", "..\\", "%2e%2e"
            };

            return !suspiciousPatterns.Any(pattern => 
                stringValue.Contains(pattern, StringComparison.OrdinalIgnoreCase));
        }

        return true;
    }

    public override string FormatErrorMessage(string name)
    {
        return $"The {name} field contains potentially malicious content.";
    }
}
```

## Rate Limiting and Throttling

### Advanced Rate Limiting Implementation
```csharp
// Security/RateLimitingService.cs
public class RateLimitingService : IRateLimitingService
{
    private readonly IDistributedCache _cache;
    private readonly IConfiguration _configuration;
    private readonly ISecurityEventLogger _securityLogger;

    public async Task<RateLimitResult> CheckRateLimitAsync(RateLimitContext context)
    {
        try
        {
            // Get rate limit configuration for endpoint
            var config = GetRateLimitConfig(context.Endpoint, context.UserTier);
            
            // Create cache keys for different time windows
            var keys = CreateRateLimitKeys(context.ClientId, context.Endpoint);
            
            // Check multiple time windows (sliding window algorithm)
            var windowChecks = await Task.WhenAll(
                CheckWindow(keys.Minute, config.PerMinute, TimeSpan.FromMinutes(1)),
                CheckWindow(keys.Hour, config.PerHour, TimeSpan.FromHours(1)),
                CheckWindow(keys.Day, config.PerDay, TimeSpan.FromDays(1))
            );

            // Find the most restrictive limit
            var restrictiveCheck = windowChecks
                .Where(w => !w.IsWithinLimit)
                .OrderBy(w => w.RetryAfter)
                .FirstOrDefault();

            if (restrictiveCheck != null)
            {
                // Log rate limit violation
                await _securityLogger.LogSecurityEventAsync(SecurityEventType.RateLimitExceeded, new 
                {
                    ClientId = context.ClientId,
                    Endpoint = context.Endpoint,
                    WindowType = restrictiveCheck.WindowType,
                    CurrentCount = restrictiveCheck.CurrentCount,
                    Limit = restrictiveCheck.Limit
                });

                return new RateLimitResult
                {
                    IsAllowed = false,
                    RetryAfter = restrictiveCheck.RetryAfter,
                    RemainingRequests = 0,
                    ResetTime = DateTime.UtcNow.Add(restrictiveCheck.RetryAfter)
                };
            }

            // Increment counters
            await IncrementCountersAsync(keys);

            // Return success with remaining quota
            var remaining = windowChecks.Min(w => w.Remaining);
            return new RateLimitResult
            {
                IsAllowed = true,
                RemainingRequests = remaining,
                ResetTime = DateTime.UtcNow.AddMinutes(1) // Next minute boundary
            };
        }
        catch (Exception ex)
        {
            await _securityLogger.LogSecurityEventAsync(SecurityEventType.RateLimitError, new 
            {
                Error = ex.Message,
                ClientId = context.ClientId,
                Endpoint = context.Endpoint
            });

            // Fail secure - deny request on error
            return new RateLimitResult
            {
                IsAllowed = false,
                RetryAfter = TimeSpan.FromMinutes(1)
            };
        }
    }

    private RateLimitConfig GetRateLimitConfig(string endpoint, UserTier userTier)
    {
        // Default limits
        var config = new RateLimitConfig
        {
            PerMinute = 60,
            PerHour = 1000,
            PerDay = 10000
        };

        // Adjust limits based on user tier
        var multiplier = userTier switch
        {
            UserTier.Free => 1.0,
            UserTier.Basic => 2.0,
            UserTier.Premium => 5.0,
            UserTier.Enterprise => 10.0,
            _ => 1.0
        };

        config.PerMinute = (int)(config.PerMinute * multiplier);
        config.PerHour = (int)(config.PerHour * multiplier);
        config.PerDay = (int)(config.PerDay * multiplier);

        // Special limits for AI endpoints
        if (endpoint.Contains("/api/ai/"))
        {
            config.PerMinute = Math.Min(config.PerMinute, userTier switch
            {
                UserTier.Free => 5,
                UserTier.Basic => 20,
                UserTier.Premium => 60,
                UserTier.Enterprise => 200,
                _ => 5
            });
        }

        return config;
    }

    private async Task<WindowCheck> CheckWindow(string key, int limit, TimeSpan window)
    {
        var current = await GetCurrentCountAsync(key);
        
        return new WindowCheck
        {
            IsWithinLimit = current < limit,
            CurrentCount = current,
            Limit = limit,
            Remaining = Math.Max(0, limit - current),
            RetryAfter = window,
            WindowType = GetWindowType(window)
        };
    }

    private async Task<int> GetCurrentCountAsync(string key)
    {
        var value = await _cache.GetStringAsync(key);
        return int.TryParse(value, out var count) ? count : 0;
    }

    private async Task IncrementCountersAsync(RateLimitKeys keys)
    {
        var options = new DistributedCacheEntryOptions();
        
        // Increment minute counter
        await IncrementCounterAsync(keys.Minute, TimeSpan.FromMinutes(1));
        
        // Increment hour counter
        await IncrementCounterAsync(keys.Hour, TimeSpan.FromHours(1));
        
        // Increment day counter
        await IncrementCounterAsync(keys.Day, TimeSpan.FromDays(1));
    }

    private async Task IncrementCounterAsync(string key, TimeSpan expiry)
    {
        var current = await GetCurrentCountAsync(key);
        var newValue = current + 1;
        
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = expiry
        };
        
        await _cache.SetStringAsync(key, newValue.ToString(), options);
    }
}
```

## Output Security and Data Protection

### Response Filtering and Sanitization
```csharp
// Security/ResponseSecurityService.cs
public class ResponseSecurityService : IResponseSecurityService
{
    private readonly IDataClassificationService _dataClassification;
    private readonly IUserContextService _userContext;

    public async Task<object> SecureResponseDataAsync(object responseData, ClaimsPrincipal user)
    {
        if (responseData == null) return null;

        // Get user permissions and context
        var userPermissions = await GetUserPermissionsAsync(user);
        var familyContext = await GetFamilyContextAsync(user);

        // Apply row-level security
        var filteredData = await ApplyRowLevelSecurityAsync(responseData, familyContext);

        // Remove sensitive fields based on user permissions
        var sanitizedData = SanitizeResponseData(filteredData, userPermissions);

        // Apply data masking where necessary
        var maskedData = ApplyDataMasking(sanitizedData, userPermissions);

        return maskedData;
    }

    private object SanitizeResponseData(object data, UserPermissions permissions)
    {
        if (data == null) return null;

        var json = JsonSerializer.Serialize(data);
        var jsonDoc = JsonDocument.Parse(json);
        var result = new Dictionary<string, object>();

        foreach (var property in jsonDoc.RootElement.EnumerateObject())
        {
            var fieldName = property.Name;
            var fieldValue = property.Value;

            // Check if user has permission to view this field
            if (CanUserAccessField(fieldName, permissions))
            {
                // Apply additional sanitization based on field type
                var sanitizedValue = SanitizeFieldValue(fieldName, fieldValue, permissions);
                result[fieldName] = sanitizedValue;
            }
        }

        return result;
    }

    private bool CanUserAccessField(string fieldName, UserPermissions permissions)
    {
        // Define sensitive fields that require special permissions
        var sensitiveFields = new Dictionary<string, string[]>
        {
            ["email"] = new[] { "ViewUserDetails" },
            ["phoneNumber"] = new[] { "ViewUserDetails" },
            ["address"] = new[] { "ViewUserDetails" },
            ["paymentInfo"] = new[] { "ViewPaymentDetails" },
            ["apiKey"] = new[] { "ViewApiKeys" },
            ["internalNotes"] = new[] { "ViewInternalData" },
        };

        if (sensitiveFields.ContainsKey(fieldName.ToLowerInvariant()))
        {
            var requiredPermissions = sensitiveFields[fieldName.ToLowerInvariant()];
            return requiredPermissions.Any(perm => permissions.HasPermission(perm));
        }

        return true; // Allow access to non-sensitive fields by default
    }

    private object ApplyDataMasking(object data, UserPermissions permissions)
    {
        if (data is IDictionary<string, object> dict)
        {
            var maskedDict = new Dictionary<string, object>();
            
            foreach (var kvp in dict)
            {
                var fieldName = kvp.Key.ToLowerInvariant();
                var fieldValue = kvp.Value;

                // Apply masking based on field type and user permissions
                var maskedValue = fieldName switch
                {
                    "email" when !permissions.HasPermission("ViewFullEmail") => 
                        MaskEmail(fieldValue?.ToString()),
                    "phonenumber" when !permissions.HasPermission("ViewFullPhone") => 
                        MaskPhoneNumber(fieldValue?.ToString()),
                    "creditcard" when !permissions.HasPermission("ViewFullPayment") => 
                        MaskCreditCard(fieldValue?.ToString()),
                    _ => fieldValue
                };

                maskedDict[kvp.Key] = maskedValue;
            }

            return maskedDict;
        }

        return data;
    }

    private string MaskEmail(string email)
    {
        if (string.IsNullOrEmpty(email) || !email.Contains('@'))
            return email;

        var parts = email.Split('@');
        var localPart = parts[0];
        var domain = parts[1];

        if (localPart.Length <= 2)
            return $"**@{domain}";

        return $"{localPart[0]}***{localPart[^1]}@{domain}";
    }

    private string MaskPhoneNumber(string phone)
    {
        if (string.IsNullOrEmpty(phone) || phone.Length < 4)
            return "***";

        return $"***-***-{phone[^4..]}";
    }

    private string MaskCreditCard(string creditCard)
    {
        if (string.IsNullOrEmpty(creditCard) || creditCard.Length < 4)
            return "****";

        return $"****-****-****-{creditCard[^4..]}";
    }
}
```

## Security Headers and CORS

### Security Headers Middleware
```csharp
// Middleware/SecurityHeadersMiddleware.cs
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context)
    {
        // Content Security Policy
        context.Response.Headers.Add("Content-Security-Policy", 
            "default-src 'self'; " +
            "script-src 'self' 'unsafe-inline' https://apis.google.com; " +
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; " +
            "img-src 'self' data: https:; " +
            "font-src 'self' https://fonts.gstatic.com; " +
            "connect-src 'self' https://api.mealprep.com; " +
            "frame-ancestors 'none'; " +
            "base-uri 'self'; " +
            "form-action 'self'");

        // Prevent clickjacking
        context.Response.Headers.Add("X-Frame-Options", "DENY");

        // Prevent MIME sniffing
        context.Response.Headers.Add("X-Content-Type-Options", "nosniff");

        // Enable XSS protection
        context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");

        // Force HTTPS
        context.Response.Headers.Add("Strict-Transport-Security", 
            "max-age=31536000; includeSubDomains; preload");

        // Referrer policy
        context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");

        // Permissions policy
        context.Response.Headers.Add("Permissions-Policy", 
            "geolocation=(), microphone=(), camera=()");

        // Remove server information
        context.Response.Headers.Remove("Server");
        context.Response.Headers.Remove("X-Powered-By");

        await _next(context);
    }
}

// CORS Configuration
public class CorsConfiguration
{
    public static void ConfigureSecureCors(IServiceCollection services, IConfiguration configuration)
    {
        services.AddCors(options =>
        {
            options.AddPolicy("MealPrepPolicy", builder =>
            {
                var allowedOrigins = configuration.GetSection("Cors:AllowedOrigins").Get<string[]>() 
                    ?? new[] { "https://localhost:3000" };

                builder
                    .WithOrigins(allowedOrigins)
                    .WithMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                    .WithHeaders("Content-Type", "Authorization", "X-Requested-With", "X-API-Key")
                    .AllowCredentials()
                    .SetPreflightMaxAge(TimeSpan.FromMinutes(10))
                    .WithExposedHeaders("X-RateLimit-Remaining", "X-RateLimit-Reset");
            });

            // Separate policy for health checks and monitoring
            options.AddPolicy("HealthCheckPolicy", builder =>
            {
                builder
                    .WithOrigins("https://monitor.mealprep.com")
                    .WithMethods("GET")
                    .WithHeaders("User-Agent");
            });
        });
    }
}
```

This comprehensive API security guide provides enterprise-level protection for the MealPrep application APIs, covering authentication, authorization, input validation, rate limiting, and data protection strategies.

*This API security guide should be regularly updated to address emerging threats and security best practices.*