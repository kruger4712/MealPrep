# API Authentication and Authorization Guide

## Overview
Comprehensive guide for authentication and authorization in the MealPrep API, covering JWT tokens, OAuth flows, permission management, and security best practices for accessing the AI-powered meal planning endpoints.

## Authentication Architecture

### Authentication Flow Overview
The MealPrep API uses JWT (JSON Web Tokens) for stateless authentication with role-based access control (RBAC) for fine-grained authorization.

```mermaid
graph TD
    A[Client Application] --> B[POST /auth/login]
    B --> C[Authentication Service]
    C --> D[User Repository]
    D --> E[Password Verification]
    E --> F{Valid Credentials?}
    F -->|Yes| G[Generate JWT Token]
    F -->|No| H[Return 401 Unauthorized]
    G --> I[Return Access Token + Refresh Token]
    I --> J[Client Stores Tokens]
    J --> K[API Requests with Authorization Header]
    K --> L[Token Validation Middleware]
    L --> M{Valid Token?}
    M -->|Yes| N[Extract User Claims]
    M -->|No| O[Return 401 Unauthorized]
    N --> P[Authorization Check]
    P --> Q{Sufficient Permissions?}
    Q -->|Yes| R[Process Request]
    Q -->|No| S[Return 403 Forbidden]
```

### Token Structure
```yaml
JWT Token Claims:
  Standard Claims:
    iss: "https://api.mealprep.com"        # Issuer
    sub: "user_12345"                       # Subject (User ID)
    aud: "mealprep-api"                     # Audience
    exp: 1640995200                         # Expiration Time
    iat: 1640991600                         # Issued At
    jti: "token_uuid_12345"                 # JWT ID
    
  Custom Claims:
    userId: 12345                           # User Identifier
    email: "user@example.com"               # User Email
    roles: ["user", "premium"]              # User Roles
    familyId: 67890                         # Family Context
    permissions: ["recipes:read", "ai:use"] # Specific Permissions
    sessionId: "session_uuid_67890"         # Session Identifier
```

---

## Authentication Endpoints

### POST /auth/login
Authenticate user credentials and receive access tokens.

**Request:**
```http
POST /auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePassword123!",
  "rememberMe": false,
  "deviceInfo": {
    "deviceId": "device_uuid_12345",
    "deviceName": "iPhone 14",
    "platform": "iOS",
    "appVersion": "1.2.0"
  }
}
```

**Response (Success):**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IlJGVCJ9...",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "refreshExpiresIn": 2592000,
  "user": {
    "id": 12345,
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "roles": ["user", "premium"],
    "familyId": 67890,
    "preferences": {
      "theme": "light",
      "language": "en-US",
      "notifications": true
    }
  },
  "permissions": [
    "recipes:read",
    "recipes:write",
    "recipes:delete",
    "family:read",
    "family:write",
    "ai:use",
    "menu:read",
    "menu:write"
  ]
}
```

**Response (Error):**
```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "error": "invalid_credentials",
  "message": "Invalid email or password",
  "details": {
    "attempts": 3,
    "lockoutTime": null,
    "nextAttemptIn": null
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "requestId": "req_12345"
}
```

### POST /auth/refresh
Refresh an expired access token using a valid refresh token.

**Request:**
```http
POST /auth/refresh
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IlJGVCJ9...

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IlJGVCJ9..."
}
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IlJGVCJ9...",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "refreshExpiresIn": 2592000
}
```

### POST /auth/logout
Invalidate current session and tokens.

**Request:**
```http
POST /auth/logout
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IlJGVCJ9...",
  "logoutAllSessions": false
}
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "message": "Successfully logged out",
  "sessionsClosed": 1,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### POST /auth/register
Create a new user account.

**Request:**
```http
POST /auth/register
Content-Type: application/json

{
  "email": "newuser@example.com",
  "password": "SecurePassword123!",
  "firstName": "Jane",
  "lastName": "Smith",
  "agreeToTerms": true,
  "emailVerificationRequired": true,
  "marketingConsent": false
}
```

**Response:**
```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "user": {
    "id": 12346,
    "email": "newuser@example.com",
    "firstName": "Jane",
    "lastName": "Smith",
    "emailVerified": false,
    "createdAt": "2024-01-15T10:30:00Z"
  },
  "message": "Account created successfully. Please check your email for verification.",
  "verificationRequired": true
}
```

---

## Authorization and Permissions

### Role-Based Access Control (RBAC)

#### User Roles
```yaml
User Roles:
  guest:
    description: "Unauthenticated user"
    permissions:
      - "recipes:browse_public"
      - "auth:login"
      - "auth:register"
    
  user:
    description: "Authenticated standard user"
    inherits: ["guest"]
    permissions:
      - "recipes:read"
      - "recipes:write_own"
      - "recipes:delete_own"
      - "family:read_own"
      - "family:write_own"
      - "menu:read_own"
      - "menu:write_own"
      - "ai:basic_suggestions"
      - "profile:read_own"
      - "profile:write_own"
    
  premium:
    description: "Premium subscription user"
    inherits: ["user"]
    permissions:
      - "ai:advanced_suggestions"
      - "ai:custom_personas"
      - "recipes:import_bulk"
      - "menu:advanced_planning"
      - "analytics:personal"
      - "export:personal_data"
    
  family_admin:
    description: "Family group administrator"
    inherits: ["premium"]
    permissions:
      - "family:admin"
      - "family:invite_members"
      - "family:remove_members"
      - "menu:manage_family"
      - "recipes:share_family"
    
  moderator:
    description: "Content moderation role"
    inherits: ["premium"]
    permissions:
      - "recipes:moderate"
      - "content:review"
      - "users:view_reports"
      - "comments:moderate"
    
  admin:
    description: "System administrator"
    inherits: ["moderator"]
    permissions:
      - "users:read"
      - "users:write"
      - "users:delete"
      - "system:configuration"
      - "analytics:global"
      - "audit:read"
```

#### Permission Structure
```csharp
public static class Permissions
{
    // Recipe Permissions
    public const string RECIPES_READ = "recipes:read";
    public const string RECIPES_WRITE = "recipes:write";
    public const string RECIPES_DELETE = "recipes:delete";
    public const string RECIPES_MODERATE = "recipes:moderate";
    public const string RECIPES_SHARE = "recipes:share";
    
    // AI Permissions
    public const string AI_BASIC_SUGGESTIONS = "ai:basic_suggestions";
    public const string AI_ADVANCED_SUGGESTIONS = "ai:advanced_suggestions";
    public const string AI_CUSTOM_PERSONAS = "ai:custom_personas";
    public const string AI_BULK_GENERATION = "ai:bulk_generation";
    
    // Family Permissions
    public const string FAMILY_READ = "family:read";
    public const string FAMILY_WRITE = "family:write";
    public const string FAMILY_ADMIN = "family:admin";
    public const string FAMILY_INVITE = "family:invite_members";
    
    // Menu Planning Permissions
    public const string MENU_READ = "menu:read";
    public const string MENU_WRITE = "menu:write";
    public const string MENU_ADVANCED = "menu:advanced_planning";
    
    // System Permissions
    public const string SYSTEM_ADMIN = "system:admin";
    public const string USERS_MANAGE = "users:manage";
    public const string ANALYTICS_VIEW = "analytics:view";
}
```

### Authorization Middleware Implementation
```csharp
public class AuthorizationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IAuthorizationService _authorizationService;

    public async Task InvokeAsync(HttpContext context)
    {
        var endpoint = context.GetEndpoint();
        var authorizeAttribute = endpoint?.Metadata.GetMetadata<AuthorizeAttribute>();
        
        if (authorizeAttribute != null)
        {
            var authResult = await ValidateAuthorizationAsync(context, authorizeAttribute);
            
            if (!authResult.Succeeded)
            {
                await HandleUnauthorizedAsync(context, authResult);
                return;
            }
        }

        await _next(context);
    }

    private async Task<AuthorizationResult> ValidateAuthorizationAsync(
        HttpContext context, 
        AuthorizeAttribute attribute)
    {
        // Extract token from Authorization header
        var token = ExtractTokenFromHeader(context);
        if (string.IsNullOrEmpty(token))
        {
            return AuthorizationResult.Failed("No token provided");
        }

        // Validate JWT token
        var tokenValidation = await _authorizationService.ValidateTokenAsync(token);
        if (!tokenValidation.IsValid)
        {
            return AuthorizationResult.Failed(tokenValidation.Error);
        }

        // Extract user claims
        var userClaims = tokenValidation.Claims;
        context.Items["User"] = userClaims;

        // Check required permissions
        if (!string.IsNullOrEmpty(attribute.Policy))
        {
            var hasPermission = await _authorizationService.HasPermissionAsync(
                userClaims.UserId, 
                attribute.Policy
            );
            
            if (!hasPermission)
            {
                return AuthorizationResult.Failed($"Insufficient permissions: {attribute.Policy}");
            }
        }

        // Check required roles
        if (attribute.Roles != null)
        {
            var hasRole = userClaims.Roles.Any(role => 
                attribute.Roles.Split(',').Contains(role, StringComparer.OrdinalIgnoreCase)
            );
            
            if (!hasRole)
            {
                return AuthorizationResult.Failed($"Required role not found: {attribute.Roles}");
            }
        }

        return AuthorizationResult.Success();
    }
}
```

---

## API Security Headers

### Required Security Headers
All API responses must include the following security headers:

```http
# Prevent clickjacking attacks
X-Frame-Options: DENY

# Enable XSS protection
X-XSS-Protection: 1; mode=block

# Prevent MIME type sniffing
X-Content-Type-Options: nosniff

# Referrer policy for privacy
Referrer-Policy: strict-origin-when-cross-origin

# Content Security Policy
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'

# HTTP Strict Transport Security (HTTPS only)
Strict-Transport-Security: max-age=31536000; includeSubDomains

# Remove server information
Server: MealPrep-API

# API versioning
API-Version: 1.0

# Rate limiting information
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```

### CORS Configuration
```csharp
public void ConfigureCors(IServiceCollection services)
{
    services.AddCors(options =>
    {
        options.AddPolicy("MealPrepCorsPolicy", builder =>
        {
            builder
                .WithOrigins(
                    "https://mealprep.com",
                    "https://app.mealprep.com",
                    "https://staging.mealprep.com"
                )
                .WithMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .WithHeaders(
                    "Authorization",
                    "Content-Type",
                    "X-Requested-With",
                    "X-API-Version",
                    "X-Client-Version"
                )
                .AllowCredentials()
                .SetPreflightMaxAge(TimeSpan.FromMinutes(5));
        });
    });
}
```

---

## Token Management

### Access Token Lifecycle
```csharp
public class TokenService
{
    private readonly IConfiguration _configuration;
    private readonly IUserRepository _userRepository;
    private readonly ISessionRepository _sessionRepository;

    public async Task<TokenResponse> GenerateTokensAsync(User user, LoginRequest request)
    {
        var claims = new List<Claim>
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim("userId", user.Id.ToString()),
            new Claim("familyId", user.FamilyId?.ToString() ?? ""),
            new Claim("sessionId", Guid.NewGuid().ToString())
        };

        // Add role claims
        foreach (var role in user.Roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }

        // Add permission claims
        var permissions = await GetUserPermissionsAsync(user.Id);
        foreach (var permission in permissions)
        {
            claims.Add(new Claim("permission", permission));
        }

        var accessToken = GenerateJwtToken(claims, TimeSpan.FromHours(1));
        var refreshToken = GenerateRefreshToken();

        // Store session information
        await _sessionRepository.CreateSessionAsync(new UserSession
        {
            UserId = user.Id,
            SessionId = claims.First(c => c.Type == "sessionId").Value,
            RefreshToken = refreshToken,
            ExpiresAt = DateTime.UtcNow.AddDays(30),
            DeviceInfo = request.DeviceInfo,
            IpAddress = GetClientIpAddress(),
            CreatedAt = DateTime.UtcNow
        });

        return new TokenResponse
        {
            AccessToken = accessToken,
            RefreshToken = refreshToken,
            TokenType = "Bearer",
            ExpiresIn = 3600,
            RefreshExpiresIn = 2592000
        };
    }

    public async Task<bool> ValidateTokenAsync(string token)
    {
        try
        {
            var tokenHandler = new JwtSecurityTokenHandler();
            var validationParameters = GetTokenValidationParameters();
            
            var principal = tokenHandler.ValidateToken(token, validationParameters, out var validatedToken);
            
            // Additional validation checks
            var jti = principal.FindFirst(JwtRegisteredClaimNames.Jti)?.Value;
            var sessionId = principal.FindFirst("sessionId")?.Value;
            
            // Check if token is blacklisted
            var isBlacklisted = await _sessionRepository.IsTokenBlacklistedAsync(jti);
            if (isBlacklisted)
            {
                return false;
            }

            // Check if session is still valid
            var session = await _sessionRepository.GetSessionAsync(sessionId);
            if (session == null || session.ExpiresAt < DateTime.UtcNow)
            {
                return false;
            }

            return true;
        }
        catch
        {
            return false;
        }
    }
}
```

### Refresh Token Rotation
```csharp
public async Task<TokenResponse> RefreshTokenAsync(string refreshToken)
{
    // Validate refresh token
    var session = await _sessionRepository.GetSessionByRefreshTokenAsync(refreshToken);
    if (session == null || session.ExpiresAt < DateTime.UtcNow)
    {
        throw new UnauthorizedException("Invalid or expired refresh token");
    }

    // Get user information
    var user = await _userRepository.GetByIdAsync(session.UserId);
    if (user == null || !user.IsActive)
    {
        throw new UnauthorizedException("User account not found or inactive");
    }

    // Rotate refresh token (invalidate old, create new)
    await _sessionRepository.InvalidateRefreshTokenAsync(refreshToken);
    
    // Generate new tokens
    var newTokens = await GenerateTokensAsync(user, new LoginRequest 
    { 
        DeviceInfo = session.DeviceInfo 
    });

    // Update session with new refresh token
    session.RefreshToken = newTokens.RefreshToken;
    session.LastUsedAt = DateTime.UtcNow;
    await _sessionRepository.UpdateSessionAsync(session);

    return newTokens;
}
```

---

## API Rate Limiting

### Rate Limiting Configuration
```yaml
Rate Limiting Rules:
  Authentication Endpoints:
    /auth/login:
      - 5 requests per minute per IP
      - 20 requests per hour per IP
      - Sliding window counter
      
    /auth/register:
      - 2 requests per minute per IP
      - 10 requests per hour per IP
      
    /auth/refresh:
      - 10 requests per minute per user
      - 100 requests per hour per user
      
  API Endpoints:
    Standard Users:
      - 1000 requests per hour
      - 10000 requests per day
      - 50 requests per minute burst
      
    Premium Users:
      - 5000 requests per hour
      - 50000 requests per day
      - 100 requests per minute burst
      
  AI Endpoints:
    /ai/suggestions:
      Standard: 10 requests per hour
      Premium: 100 requests per hour
      
    /ai/personas:
      Standard: 5 requests per hour
      Premium: 50 requests per hour
```

### Rate Limiting Implementation
```csharp
public class RateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IRateLimitService _rateLimitService;

    public async Task InvokeAsync(HttpContext context)
    {
        var endpoint = context.Request.Path.Value;
        var user = context.Items["User"] as UserClaims;
        var clientId = GetClientIdentifier(context, user);

        var rateLimitResult = await _rateLimitService.CheckRateLimitAsync(
            endpoint, 
            clientId, 
            user?.Roles
        );

        if (!rateLimitResult.IsAllowed)
        {
            context.Response.StatusCode = 429; // Too Many Requests
            context.Response.Headers.Add("Retry-After", rateLimitResult.RetryAfter.ToString());
            context.Response.Headers.Add("X-RateLimit-Limit", rateLimitResult.Limit.ToString());
            context.Response.Headers.Add("X-RateLimit-Remaining", "0");
            context.Response.Headers.Add("X-RateLimit-Reset", rateLimitResult.ResetTime.ToString());

            await context.Response.WriteAsync(JsonSerializer.Serialize(new
            {
                error = "rate_limit_exceeded",
                message = "Too many requests. Please try again later.",
                retryAfter = rateLimitResult.RetryAfter,
                limit = rateLimitResult.Limit
            }));
            
            return;
        }

        // Add rate limit headers to successful responses
        context.Response.OnStarting(() =>
        {
            context.Response.Headers.Add("X-RateLimit-Limit", rateLimitResult.Limit.ToString());
            context.Response.Headers.Add("X-RateLimit-Remaining", rateLimitResult.Remaining.ToString());
            context.Response.Headers.Add("X-RateLimit-Reset", rateLimitResult.ResetTime.ToString());
            return Task.CompletedTask;
        });

        await _next(context);
    }
}
```

---

## Security Best Practices

### Input Validation
```csharp
[HttpPost]
[Authorize(Policy = Permissions.RECIPES_WRITE)]
public async Task<IActionResult> CreateRecipe([FromBody] CreateRecipeRequest request)
{
    // Input validation
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }

    // Additional security validation
    var user = HttpContext.GetCurrentUser();
    
    // Check ownership for family recipes
    if (request.IsSharedWithFamily && user.FamilyId.HasValue)
    {
        var hasPermission = await _authorizationService.CanShareWithFamilyAsync(user.Id, user.FamilyId.Value);
        if (!hasPermission)
        {
            return Forbid("Insufficient permissions to share with family");
        }
    }

    // Sanitize input
    request.Name = _inputSanitizer.SanitizeHtml(request.Name);
    request.Description = _inputSanitizer.SanitizeHtml(request.Description);
    request.Instructions = _inputSanitizer.SanitizeHtml(request.Instructions);

    // Business logic validation
    if (request.Ingredients == null || !request.Ingredients.Any())
    {
        return BadRequest("At least one ingredient is required");
    }

    var recipe = await _recipeService.CreateRecipeAsync(request, user.Id);
    return CreatedAtAction(nameof(GetRecipe), new { id = recipe.Id }, recipe);
}
```

### Audit Logging
```csharp
public class SecurityAuditLogger
{
    public async Task LogAuthenticationEventAsync(AuthenticationEvent evt)
    {
        var auditEntry = new SecurityAuditEntry
        {
            EventType = evt.Type,
            UserId = evt.UserId,
            IpAddress = evt.IpAddress,
            UserAgent = evt.UserAgent,
            Success = evt.Success,
            FailureReason = evt.FailureReason,
            Timestamp = DateTime.UtcNow,
            AdditionalData = new Dictionary<string, object>
            {
                ["DeviceInfo"] = evt.DeviceInfo,
                ["SessionId"] = evt.SessionId,
                ["Geolocation"] = evt.Geolocation
            }
        };

        await _auditRepository.CreateAuditEntryAsync(auditEntry);

        // Alert on suspicious activities
        if (ShouldTriggerSecurityAlert(evt))
        {
            await _securityAlertService.TriggerAlertAsync(evt);
        }
    }

    private bool ShouldTriggerSecurityAlert(AuthenticationEvent evt)
    {
        return evt.Type == AuthEventType.FailedLogin && 
               evt.ConsecutiveFailures >= 5 ||
               evt.Type == AuthEventType.SuspiciousLocation ||
               evt.Type == AuthEventType.TokenReplay;
    }
}
```

---

## Client Integration Examples

### JavaScript/TypeScript Client
```typescript
class MealPrepApiClient {
    private baseUrl: string;
    private accessToken: string | null = null;
    private refreshToken: string | null = null;

    constructor(baseUrl: string) {
        this.baseUrl = baseUrl;
        this.loadTokensFromStorage();
    }

    async login(email: string, password: string, deviceInfo?: DeviceInfo): Promise<LoginResponse> {
        const response = await fetch(`${this.baseUrl}/auth/login`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                email,
                password,
                deviceInfo: deviceInfo || this.getDeviceInfo()
            })
        });

        if (!response.ok) {
            const error = await response.json();
            throw new ApiError(error.message, response.status, error);
        }

        const loginData = await response.json();
        this.setTokens(loginData.accessToken, loginData.refreshToken);
        
        return loginData;
    }

    async makeAuthenticatedRequest<T>(
        endpoint: string, 
        options: RequestInit = {}
    ): Promise<T> {
        const headers = {
            'Content-Type': 'application/json',
            ...options.headers
        };

        if (this.accessToken) {
            headers['Authorization'] = `Bearer ${this.accessToken}`;
        }

        let response = await fetch(`${this.baseUrl}${endpoint}`, {
            ...options,
            headers
        });

        // Handle token expiration
        if (response.status === 401 && this.refreshToken) {
            const refreshed = await this.refreshAccessToken();
            if (refreshed) {
                headers['Authorization'] = `Bearer ${this.accessToken}`;
                response = await fetch(`${this.baseUrl}${endpoint}`, {
                    ...options,
                    headers
                });
            }
        }

        if (!response.ok) {
            const error = await response.json();
            throw new ApiError(error.message, response.status, error);
        }

        return response.json();
    }

    private async refreshAccessToken(): Promise<boolean> {
        try {
            const response = await fetch(`${this.baseUrl}/auth/refresh`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${this.accessToken}`
                },
                body: JSON.stringify({
                    refreshToken: this.refreshToken
                })
            });

            if (response.ok) {
                const tokens = await response.json();
                this.setTokens(tokens.accessToken, tokens.refreshToken);
                return true;
            }
        } catch (error) {
            console.error('Token refresh failed:', error);
        }

        this.clearTokens();
        return false;
    }

    private setTokens(accessToken: string, refreshToken: string): void {
        this.accessToken = accessToken;
        this.refreshToken = refreshToken;
        localStorage.setItem('mealprep_access_token', accessToken);
        localStorage.setItem('mealprep_refresh_token', refreshToken);
    }
}
```

### Mobile Client (React Native)
```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';
import { Platform } from 'react-native';

class MealPrepMobileClient {
    private async getDeviceInfo(): Promise<DeviceInfo> {
        return {
            deviceId: await this.getDeviceId(),
            deviceName: await this.getDeviceName(),
            platform: Platform.OS,
            appVersion: '1.2.0'
        };
    }

    private async storeTokensSecurely(accessToken: string, refreshToken: string): Promise<void> {
        try {
            await AsyncStorage.multiSet([
                ['@mealprep:access_token', accessToken],
                ['@mealprep:refresh_token', refreshToken]
            ]);
        } catch (error) {
            console.error('Failed to store tokens:', error);
        }
    }

    private async loadTokensFromSecureStorage(): Promise<void> {
        try {
            const tokens = await AsyncStorage.multiGet([
                '@mealprep:access_token',
                '@mealprep:refresh_token'
            ]);
            
            this.accessToken = tokens[0][1];
            this.refreshToken = tokens[1][1];
        } catch (error) {
            console.error('Failed to load tokens:', error);
        }
    }
}
```

---

## Testing Authentication

### Unit Tests for Authentication Service
```csharp
[Test]
public async Task Login_WithValidCredentials_ReturnsTokens()
{
    // Arrange
    var loginRequest = new LoginRequest
    {
        Email = "test@example.com",
        Password = "ValidPassword123!",
        DeviceInfo = new DeviceInfo { DeviceId = "test_device" }
    };

    var user = new User
    {
        Id = 1,
        Email = "test@example.com",
        PasswordHash = BCrypt.Net.BCrypt.HashPassword("ValidPassword123!"),
        IsActive = true,
        Roles = new[] { "user" }
    };

    _userRepository.Setup(x => x.GetByEmailAsync("test@example.com"))
               .ReturnsAsync(user);

    // Act
    var result = await _authenticationService.LoginAsync(loginRequest);

    // Assert
    Assert.That(result.Success, Is.True);
    Assert.That(result.AccessToken, Is.Not.Null);
    Assert.That(result.RefreshToken, Is.Not.Null);
    Assert.That(result.User.Email, Is.EqualTo("test@example.com"));
}

[Test]
public async Task ValidateToken_WithExpiredToken_ReturnsFalse()
{
    // Arrange
    var expiredToken = GenerateExpiredToken();

    // Act
    var isValid = await _tokenService.ValidateTokenAsync(expiredToken);

    // Assert
    Assert.That(isValid, Is.False);
}
```

### Integration Tests for Authentication Flow
```csharp
[Test]
public async Task AuthenticationFlow_EndToEnd_Success()
{
    // Arrange
    var client = _factory.CreateClient();
    var loginData = new
    {
        email = "test@example.com",
        password = "ValidPassword123!",
        deviceInfo = new { deviceId = "test_device" }
    };

    // Act - Login
    var loginResponse = await client.PostAsJsonAsync("/auth/login", loginData);
    loginResponse.EnsureSuccessStatusCode();
    
    var loginResult = await loginResponse.Content.ReadFromJsonAsync<LoginResponse>();
    
    // Act - Use token for authenticated request
    client.DefaultRequestHeaders.Authorization = 
        new AuthenticationHeaderValue("Bearer", loginResult.AccessToken);
    
    var recipesResponse = await client.GetAsync("/api/recipes");
    
    // Assert
    recipesResponse.EnsureSuccessStatusCode();
    Assert.That(loginResult.AccessToken, Is.Not.Null);
    Assert.That(loginResult.User.Email, Is.EqualTo("test@example.com"));
}
```

---

*Last Updated: December 2024*  
*Authentication guide continuously updated with security enhancements and best practices*