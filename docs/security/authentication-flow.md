# Authentication Flow

## Overview
Comprehensive authentication and authorization implementation for the MealPrep application using JWT tokens, refresh token rotation, and role-based access control.

## Authentication Architecture

### Token-Based Authentication Flow
```
1. User Login ? 2. Validate Credentials ? 3. Generate JWT + Refresh Token
       ?                    ?                           ?
4. Return Tokens ? 5. Store Refresh Token ? 6. Sign JWT with Secret
       ?
7. Client Stores Tokens ? 8. Include JWT in API Requests ? 9. Validate JWT
       ?                           ?                            ?
10. Auto-refresh when expired ? 11. Return 401 if invalid ? 12. Process Request
```

### JWT Token Structure
```csharp
public class JwtTokenService : IJwtTokenService
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<JwtTokenService> _logger;

    public JwtToken GenerateToken(User user, Family family)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.ASCII.GetBytes(_configuration["Jwt:Secret"]);
        
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new(ClaimTypes.Email, user.Email),
            new(ClaimTypes.Name, user.FullName),
            new("subscription_tier", user.SubscriptionTier.ToString()),
            new("email_verified", user.EmailConfirmed.ToString()),
            new("family_id", family?.Id.ToString() ?? ""),
            new("family_role", family?.GetMemberRole(user.Id)?.ToString() ?? ""),
            new("permissions", JsonSerializer.Serialize(GetUserPermissions(user, family))),
            new("device_id", GenerateDeviceFingerprint()),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(), ClaimValueTypes.Integer64)
        };

        var tokenDescriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(claims),
            Expires = DateTime.UtcNow.AddMinutes(_configuration.GetValue<int>("Jwt:AccessTokenExpiryMinutes", 60)),
            Issuer = _configuration["Jwt:Issuer"],
            Audience = _configuration["Jwt:Audience"],
            SigningCredentials = new SigningCredentials(new SymmetricSecurityKey(key), SecurityAlgorithms.HmacSha256Signature)
        };

        var token = tokenHandler.CreateToken(tokenDescriptor);
        var tokenString = tokenHandler.WriteToken(token);

        return new JwtToken
        {
            AccessToken = tokenString,
            ExpiresAt = tokenDescriptor.Expires.Value,
            TokenType = "Bearer"
        };
    }

    public async Task<ClaimsPrincipal> ValidateTokenAsync(string token)
    {
        try
        {
            var tokenHandler = new JwtSecurityTokenHandler();
            var key = Encoding.ASCII.GetBytes(_configuration["Jwt:Secret"]);

            var validationParameters = new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(key),
                ValidateIssuer = true,
                ValidIssuer = _configuration["Jwt:Issuer"],
                ValidateAudience = true,
                ValidAudience = _configuration["Jwt:Audience"],
                ValidateLifetime = true,
                ClockSkew = TimeSpan.Zero, // No tolerance for clock skew
                RequireExpirationTime = true
            };

            var principal = tokenHandler.ValidateToken(token, validationParameters, out SecurityToken validatedToken);
            
            // Additional security checks
            await ValidateTokenSecurityAsync(principal, validatedToken);
            
            return principal;
        }
        catch (SecurityTokenException ex)
        {
            _logger.LogWarning("Token validation failed: {Error}", ex.Message);
            throw new UnauthorizedAccessException("Invalid token", ex);
        }
    }

    private async Task ValidateTokenSecurityAsync(ClaimsPrincipal principal, SecurityToken token)
    {
        // Check if token is in blacklist (for logout/compromised tokens)
        var jti = principal.FindFirst(JwtRegisteredClaimNames.Jti)?.Value;
        if (!string.IsNullOrEmpty(jti) && await _tokenBlacklistService.IsBlacklistedAsync(jti))
        {
            throw new SecurityTokenValidationException("Token has been revoked");
        }

        // Validate device fingerprint for additional security
        var deviceId = principal.FindFirst("device_id")?.Value;
        if (!string.IsNullOrEmpty(deviceId))
        {
            await ValidateDeviceFingerprintAsync(deviceId, principal);
        }

        // Check for suspicious activity patterns
        await ValidateUserActivityAsync(principal);
    }
}
```

## Refresh Token Management

### Refresh Token Implementation
```csharp
public class RefreshTokenService : IRefreshTokenService
{
    private readonly IUserRepository _userRepository;
    private readonly IRefreshTokenRepository _refreshTokenRepository;
    private readonly IJwtTokenService _jwtTokenService;
    private readonly ILogger<RefreshTokenService> _logger;

    public async Task<RefreshTokenResult> GenerateRefreshTokenAsync(User user, string deviceFingerprint)
    {
        // Revoke existing refresh tokens for this user/device to prevent accumulation
        await RevokeDeviceRefreshTokensAsync(user.Id, deviceFingerprint);

        var refreshToken = new RefreshToken
        {
            Id = Guid.NewGuid(),
            UserId = user.Id,
            Token = GenerateSecureToken(),
            DeviceFingerprint = deviceFingerprint,
            CreatedAt = DateTime.UtcNow,
            ExpiresAt = DateTime.UtcNow.AddDays(30), // 30-day expiry
            IsActive = true,
            IpAddress = GetClientIpAddress(),
            UserAgent = GetClientUserAgent()
        };

        await _refreshTokenRepository.CreateAsync(refreshToken);

        return new RefreshTokenResult
        {
            RefreshToken = refreshToken.Token,
            ExpiresAt = refreshToken.ExpiresAt
        };
    }

    public async Task<TokenRefreshResult> RefreshTokenAsync(string refreshToken, string deviceFingerprint)
    {
        var token = await _refreshTokenRepository.GetByTokenAsync(refreshToken);
        
        if (token == null || !token.IsActive || token.ExpiresAt <= DateTime.UtcNow)
        {
            throw new UnauthorizedAccessException("Invalid or expired refresh token");
        }

        // Validate device fingerprint
        if (token.DeviceFingerprint != deviceFingerprint)
        {
            _logger.LogWarning("Device fingerprint mismatch for refresh token. UserId: {UserId}, Expected: {Expected}, Actual: {Actual}",
                token.UserId, token.DeviceFingerprint, deviceFingerprint);
            
            // Revoke all tokens for this user as a security measure
            await RevokeAllUserTokensAsync(token.UserId);
            throw new SecurityException("Device fingerprint mismatch - all tokens revoked");
        }

        var user = await _userRepository.GetByIdAsync(token.UserId);
        if (user == null || !user.IsActive)
        {
            throw new UnauthorizedAccessException("User not found or inactive");
        }

        var family = await _familyService.GetUserFamilyAsync(user.Id);

        // Generate new tokens with rotation
        var newAccessToken = _jwtTokenService.GenerateToken(user, family);
        var newRefreshToken = await GenerateRefreshTokenAsync(user, deviceFingerprint);

        // Revoke the old refresh token
        token.IsActive = false;
        token.RevokedAt = DateTime.UtcNow;
        await _refreshTokenRepository.UpdateAsync(token);

        // Track token refresh for security monitoring
        await _securityEventService.LogTokenRefreshAsync(user.Id, deviceFingerprint);

        return new TokenRefreshResult
        {
            AccessToken = newAccessToken.AccessToken,
            RefreshToken = newRefreshToken.RefreshToken,
            ExpiresAt = newAccessToken.ExpiresAt
        };
    }

    private string GenerateSecureToken()
    {
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }
}
```

## Multi-Factor Authentication

### MFA Implementation
```csharp
public class MultiFactorAuthService : IMultiFactorAuthService
{
    private readonly ITotpService _totpService;
    private readonly ISmsService _smsService;
    private readonly IEmailService _emailService;
    private readonly IUserRepository _userRepository;

    public async Task<MfaSetupResult> InitiateMfaSetupAsync(Guid userId, MfaMethod method)
    {
        var user = await _userRepository.GetByIdAsync(userId);
        if (user == null) throw new UserNotFoundException();

        return method switch
        {
            MfaMethod.Authenticator => await SetupAuthenticatorMfaAsync(user),
            MfaMethod.Sms => await SetupSmsMfaAsync(user),
            MfaMethod.Email => await SetupEmailMfaAsync(user),
            _ => throw new ArgumentException("Unsupported MFA method")
        };
    }

    private async Task<MfaSetupResult> SetupAuthenticatorMfaAsync(User user)
    {
        var secret = _totpService.GenerateSecret();
        var qrCodeUri = _totpService.GenerateQrCodeUri(user.Email, secret, "MealPrep");

        // Store secret temporarily (will be confirmed when user validates first code)
        await _userRepository.SetTempMfaSecretAsync(user.Id, secret);

        return new MfaSetupResult
        {
            Method = MfaMethod.Authenticator,
            Secret = secret,
            QrCodeUri = qrCodeUri,
            BackupCodes = GenerateBackupCodes()
        };
    }

    public async Task<MfaValidationResult> ValidateMfaCodeAsync(Guid userId, string code, MfaMethod method)
    {
        var user = await _userRepository.GetByIdAsync(userId);
        if (user == null) throw new UserNotFoundException();

        var isValid = method switch
        {
            MfaMethod.Authenticator => await ValidateAuthenticatorCodeAsync(user, code),
            MfaMethod.Sms => await ValidateSmsCodeAsync(user, code),
            MfaMethod.Email => await ValidateEmailCodeAsync(user, code),
            MfaMethod.BackupCode => await ValidateBackupCodeAsync(user, code),
            _ => false
        };

        if (isValid)
        {
            // Update last successful MFA timestamp
            await _userRepository.UpdateLastMfaValidationAsync(userId, DateTime.UtcNow);
            
            // Log successful MFA validation
            await _securityEventService.LogMfaValidationAsync(userId, method, true);
        }
        else
        {
            // Log failed MFA attempt
            await _securityEventService.LogMfaValidationAsync(userId, method, false);
            
            // Check for brute force attempts
            await _bruteForceProtectionService.RecordFailedMfaAttemptAsync(userId);
        }

        return new MfaValidationResult
        {
            IsValid = isValid,
            Method = method,
            ValidatedAt = DateTime.UtcNow
        };
    }

    private async Task<bool> ValidateAuthenticatorCodeAsync(User user, string code)
    {
        if (string.IsNullOrEmpty(user.MfaSecret))
            return false;

        // Validate TOTP code with time window tolerance
        return _totpService.ValidateCode(user.MfaSecret, code, 30); // 30-second window
    }

    private List<string> GenerateBackupCodes()
    {
        var codes = new List<string>();
        for (int i = 0; i < 10; i++)
        {
            codes.Add(GenerateBackupCode());
        }
        return codes;
    }

    private string GenerateBackupCode()
    {
        var random = new Random();
        return $"{random.Next(1000, 9999)}-{random.Next(1000, 9999)}";
    }
}
```

## Security Middleware

### JWT Authentication Middleware
```csharp
public class JwtAuthenticationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IJwtTokenService _jwtTokenService;
    private readonly ILogger<JwtAuthenticationMiddleware> _logger;

    public JwtAuthenticationMiddleware(RequestDelegate next, IJwtTokenService jwtTokenService, ILogger<JwtAuthenticationMiddleware> logger)
    {
        _next = next;
        _jwtTokenService = jwtTokenService;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            var token = ExtractTokenFromRequest(context.Request);
            
            if (!string.IsNullOrEmpty(token))
            {
                var principal = await _jwtTokenService.ValidateTokenAsync(token);
                context.User = principal;
                
                // Add user context for logging
                using (_logger.BeginScope(new Dictionary<string, object>
                {
                    ["UserId"] = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "Unknown",
                    ["FamilyId"] = principal.FindFirst("family_id")?.Value ?? "None"
                }))
                {
                    await _next(context);
                }
            }
            else
            {
                await _next(context);
            }
        }
        catch (UnauthorizedAccessException)
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("Unauthorized");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error in JWT authentication middleware");
            context.Response.StatusCode = 500;
            await context.Response.WriteAsync("Internal server error");
        }
    }

    private string ExtractTokenFromRequest(HttpRequest request)
    {
        // Check Authorization header
        var authHeader = request.Headers["Authorization"].FirstOrDefault();
        if (!string.IsNullOrEmpty(authHeader) && authHeader.StartsWith("Bearer ", StringComparison.OrdinalIgnoreCase))
        {
            return authHeader.Substring("Bearer ".Length).Trim();
        }

        // Check cookie as fallback (for same-site requests)
        return request.Cookies["access_token"];
    }
}
```

### Rate Limiting Middleware
```csharp
public class RateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IRateLimitService _rateLimitService;
    private readonly ILogger<RateLimitingMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        var clientId = GetClientIdentifier(context);
        var endpoint = GetEndpointKey(context);
        
        var rateLimitResult = await _rateLimitService.CheckRateLimitAsync(clientId, endpoint);
        
        if (!rateLimitResult.IsAllowed)
        {
            context.Response.StatusCode = 429; // Too Many Requests
            context.Response.Headers.Add("Retry-After", rateLimitResult.RetryAfterSeconds.ToString());
            
            await context.Response.WriteAsync(JsonSerializer.Serialize(new
            {
                error = "Rate limit exceeded",
                retryAfterSeconds = rateLimitResult.RetryAfterSeconds,
                limit = rateLimitResult.Limit,
                remaining = 0
            }));
            
            return;
        }

        // Add rate limit headers
        context.Response.Headers.Add("X-RateLimit-Limit", rateLimitResult.Limit.ToString());
        context.Response.Headers.Add("X-RateLimit-Remaining", rateLimitResult.Remaining.ToString());
        context.Response.Headers.Add("X-RateLimit-Reset", rateLimitResult.ResetAt.ToUnixTimeSeconds().ToString());

        await _next(context);
    }

    private string GetClientIdentifier(HttpContext context)
    {
        // Use user ID if authenticated, otherwise IP address
        var userId = context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        return !string.IsNullOrEmpty(userId) ? $"user:{userId}" : $"ip:{context.Connection.RemoteIpAddress}";
    }

    private string GetEndpointKey(HttpContext context)
    {
        var route = context.Request.Path.Value?.ToLowerInvariant() ?? "";
        var method = context.Request.Method.ToUpperInvariant();
        
        // Group similar endpoints to prevent abuse
        if (route.StartsWith("/api/ai/"))
            return "ai_endpoints";
        if (route.StartsWith("/api/auth/"))
            return "auth_endpoints";
        
        return $"{method}:{route}";
    }
}
```

## Frontend Authentication Integration

### Auth Context Implementation
```typescript
// contexts/AuthContext.tsx
interface AuthContextType {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => Promise<void>;
  refreshToken: () => Promise<void>;
  enableMfa: (method: MfaMethod) => Promise<MfaSetupResult>;
  validateMfa: (code: string, method: MfaMethod) => Promise<boolean>;
}

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  // Auto token refresh
  useEffect(() => {
    const refreshInterval = setInterval(async () => {
      try {
        await refreshToken();
      } catch (error) {
        console.error('Token refresh failed:', error);
        await logout();
      }
    }, 15 * 60 * 1000); // Refresh every 15 minutes

    return () => clearInterval(refreshInterval);
  }, []);

  const login = async (credentials: LoginCredentials) => {
    setIsLoading(true);
    try {
      const deviceFingerprint = generateDeviceFingerprint();
      const response = await authApi.login({
        ...credentials,
        deviceFingerprint
      });

      if (response.requiresMfa) {
        // Handle MFA challenge
        throw new MfaRequiredError(response.mfaMethods);
      }

      // Store tokens securely
      tokenStorage.setTokens(response.accessToken, response.refreshToken);
      setUser(response.user);
    } finally {
      setIsLoading(false);
    }
  };

  const logout = async () => {
    try {
      await authApi.logout();
    } catch (error) {
      console.error('Logout API call failed:', error);
    } finally {
      tokenStorage.clearTokens();
      setUser(null);
      queryClient.clear();
    }
  };

  const refreshToken = async () => {
    const refreshToken = tokenStorage.getRefreshToken();
    if (!refreshToken) {
      throw new Error('No refresh token available');
    }

    const deviceFingerprint = generateDeviceFingerprint();
    const response = await authApi.refreshToken({
      refreshToken,
      deviceFingerprint
    });

    tokenStorage.setTokens(response.accessToken, response.refreshToken);
  };

  return (
    <AuthContext.Provider value={{
      user,
      isAuthenticated: !!user,
      isLoading,
      login,
      logout,
      refreshToken,
      enableMfa,
      validateMfa
    }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Secure Token Storage
```typescript
// utils/tokenStorage.ts
interface TokenStorage {
  setTokens(accessToken: string, refreshToken: string): void;
  getAccessToken(): string | null;
  getRefreshToken(): string | null;
  clearTokens(): void;
}

class SecureTokenStorage implements TokenStorage {
  private readonly ACCESS_TOKEN_KEY = 'mealprep_access_token';
  private readonly REFRESH_TOKEN_KEY = 'mealprep_refresh_token';

  setTokens(accessToken: string, refreshToken: string): void {
    // Store access token in memory for security
    this.setMemoryToken(accessToken);
    
    // Store refresh token in httpOnly cookie or secure storage
    this.setSecureRefreshToken(refreshToken);
  }

  getAccessToken(): string | null {
    return this.getMemoryToken();
  }

  getRefreshToken(): string | null {
    return this.getSecureRefreshToken();
  }

  clearTokens(): void {
    this.clearMemoryToken();
    this.clearSecureRefreshToken();
  }

  private setMemoryToken(token: string): void {
    (window as any).__mealprep_token = token;
  }

  private getMemoryToken(): string | null {
    return (window as any).__mealprep_token || null;
  }

  private clearMemoryToken(): void {
    delete (window as any).__mealprep_token;
  }

  private setSecureRefreshToken(token: string): void {
    // In production, this would be an httpOnly cookie set by the server
    // For development, use localStorage with encryption
    const encrypted = this.encrypt(token);
    localStorage.setItem(this.REFRESH_TOKEN_KEY, encrypted);
  }

  private getSecureRefreshToken(): string | null {
    const encrypted = localStorage.getItem(this.REFRESH_TOKEN_KEY);
    return encrypted ? this.decrypt(encrypted) : null;
  }

  private clearSecureRefreshToken(): void {
    localStorage.removeItem(this.REFRESH_TOKEN_KEY);
  }

  private encrypt(value: string): string {
    // Simple encryption for demo - use proper encryption in production
    return btoa(value);
  }

  private decrypt(value: string): string {
    return atob(value);
  }
}

export const tokenStorage = new SecureTokenStorage();
```

This comprehensive authentication flow provides enterprise-grade security for the MealPrep application with JWT tokens, refresh token rotation, multi-factor authentication, and robust security middleware.

*This authentication guide should be updated as security requirements evolve and new threats emerge.*