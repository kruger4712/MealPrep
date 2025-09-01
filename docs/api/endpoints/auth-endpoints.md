# Authentication Endpoints

## Overview
Authentication endpoints for user registration, login, password management, and JWT token handling in the MealPrep application.

## Base URL
```
https://api.mealprep.com/v1/auth
```

## Endpoints

### Register User

**Endpoint:** `POST /register`

**Description:** Create a new user account with email verification.

**Request Body:**
```json
{
  "firstName": "John",
  "lastName": "Doe", 
  "email": "john.doe@example.com",
  "password": "SecurePassword123!",
  "confirmPassword": "SecurePassword123!",
  "acceptTerms": true,
  "newsletterOptIn": false
}
```

**Request Validation:**
```csharp
public class RegisterRequest
{
    [Required]
    [StringLength(50, MinimumLength = 2)]
    public string FirstName { get; set; } = string.Empty;

    [Required]
    [StringLength(50, MinimumLength = 2)]
    public string LastName { get; set; } = string.Empty;

    [Required]
    [EmailAddress]
    [StringLength(256)]
    public string Email { get; set; } = string.Empty;

    [Required]
    [StringLength(100, MinimumLength = 8)]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$",
        ErrorMessage = "Password must contain at least 8 characters, including uppercase, lowercase, number, and special character")]
    public string Password { get; set; } = string.Empty;

    [Required]
    [Compare(nameof(Password), ErrorMessage = "Passwords do not match")]
    public string ConfirmPassword { get; set; } = string.Empty;

    [Required]
    [Range(typeof(bool), "true", "true", ErrorMessage = "You must accept the terms and conditions")]
    public bool AcceptTerms { get; set; }

    public bool NewsletterOptIn { get; set; } = false;
}
```

**Success Response (201 Created):**
```json
{
  "data": {
    "user": {
      "id": "12345",
      "firstName": "John",
      "lastName": "Doe",
      "email": "john.doe@example.com",
      "emailVerified": false,
      "createdAt": "2024-01-01T00:00:00Z"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "rt_abc123...",
    "expiresAt": "2024-01-01T01:00:00Z"
  },
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

**Error Response (400 Bad Request):**
```json
{
  "errors": [
    {
      "code": "VALIDATION_ERROR",
      "message": "Email is already registered",
      "field": "email",
      "value": "john.doe@example.com"
    }
  ],
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Login User

**Endpoint:** `POST /login`

**Description:** Authenticate user and receive JWT access token.

**Request Body:**
```json
{
  "email": "john.doe@example.com",
  "password": "SecurePassword123!",
  "rememberMe": true
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "user": {
      "id": "12345",
      "firstName": "John",
      "lastName": "Doe",
      "email": "john.doe@example.com",
      "emailVerified": true,
      "roles": ["User"],
      "lastLoginAt": "2024-01-01T00:00:00Z"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "rt_abc123...",
    "expiresAt": "2024-01-01T01:00:00Z"
  },
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

**Error Response (401 Unauthorized):**
```json
{
  "errors": [
    {
      "code": "AUTHENTICATION_FAILED",
      "message": "Invalid email or password"
    }
  ],
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Refresh Token

**Endpoint:** `POST /refresh`

**Description:** Get new access token using refresh token.

**Request Body:**
```json
{
  "refreshToken": "rt_abc123..."
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "rt_def456...",
    "expiresAt": "2024-01-01T01:00:00Z"
  },
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Logout

**Endpoint:** `POST /logout`

**Description:** Invalidate current refresh token and logout user.

**Headers:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Request Body:**
```json
{
  "refreshToken": "rt_abc123..."
}
```

**Success Response (204 No Content)**

### Verify Email

**Endpoint:** `POST /verify-email`

**Description:** Verify user email address using verification token.

**Request Body:**
```json
{
  "email": "john.doe@example.com",
  "verificationToken": "vt_abc123..."
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "message": "Email verified successfully",
    "emailVerified": true
  },
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Resend Verification Email

**Endpoint:** `POST /resend-verification`

**Description:** Resend email verification link.

**Request Body:**
```json
{
  "email": "john.doe@example.com"
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "message": "Verification email sent successfully"
  },
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Forgot Password

**Endpoint:** `POST /forgot-password`

**Description:** Send password reset link to user's email.

**Request Body:**
```json
{
  "email": "john.doe@example.com"
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "message": "Password reset link sent to your email"
  },
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Reset Password

**Endpoint:** `POST /reset-password`

**Description:** Reset user password using reset token.

**Request Body:**
```json
{
  "email": "john.doe@example.com",
  "resetToken": "rt_abc123...",
  "newPassword": "NewSecurePassword123!",
  "confirmPassword": "NewSecurePassword123!"
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "message": "Password reset successfully"
  },
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Change Password

**Endpoint:** `POST /change-password`

**Description:** Change password for authenticated user.

**Headers:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Request Body:**
```json
{
  "currentPassword": "CurrentPassword123!",
  "newPassword": "NewSecurePassword123!",
  "confirmPassword": "NewSecurePassword123!"
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "message": "Password changed successfully"
  },
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Get Current User

**Endpoint:** `GET /me`

**Description:** Get current authenticated user information.

**Headers:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Success Response (200 OK):**
```json
{
  "data": {
    "id": "12345",
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com",
    "emailVerified": true,
    "roles": ["User"],
    "preferences": {
      "notifications": {
        "email": true,
        "push": false
      },
      "privacy": {
        "shareRecipes": true,
        "showProfile": false
      }
    },
    "createdAt": "2024-01-01T00:00:00Z",
    "lastLoginAt": "2024-01-01T00:00:00Z"
  },
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Update User Profile

**Endpoint:** `PUT /me`

**Description:** Update current user profile information.

**Headers:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Request Body:**
```json
{
  "firstName": "John",
  "lastName": "Doe",
  "preferences": {
    "notifications": {
      "email": true,
      "push": true
    },
    "privacy": {
      "shareRecipes": true,
      "showProfile": true
    }
  }
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "id": "12345",
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com",
    "emailVerified": true,
    "preferences": {
      "notifications": {
        "email": true,
        "push": true
      },
      "privacy": {
        "shareRecipes": true,
        "showProfile": true
      }
    },
    "updatedAt": "2024-01-01T00:00:00Z"
  },
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Delete Account

**Endpoint:** `DELETE /me`

**Description:** Delete current user account (soft delete with 30-day recovery period).

**Headers:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Request Body:**
```json
{
  "password": "CurrentPassword123!",
  "reason": "No longer needed"
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "message": "Account deletion initiated. You have 30 days to recover your account.",
    "recoveryDeadline": "2024-01-31T00:00:00Z"
  },
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

## JWT Token Structure

### Access Token Claims
```json
{
  "sub": "12345",
  "email": "john.doe@example.com",
  "given_name": "John",
  "family_name": "Doe",
  "role": "User",
  "email_verified": true,
  "iat": 1640995200,
  "exp": 1641081600,
  "iss": "MealPrep.API",
  "aud": "MealPrep.App"
}
```

### Token Validation
- **Algorithm**: HS256
- **Expiration**: 1 hour for access tokens
- **Refresh Token**: 30 days (sliding expiration)
- **Issuer**: MealPrep.API
- **Audience**: MealPrep.App

## Rate Limiting

### Authentication Endpoints
- **Login**: 5 attempts per minute per IP
- **Register**: 3 attempts per minute per IP
- **Password Reset**: 2 attempts per minute per email
- **Refresh Token**: 10 requests per minute per user

### Rate Limit Headers
```
X-RateLimit-Limit: 5
X-RateLimit-Remaining: 3
X-RateLimit-Reset: 1640995260
Retry-After: 60
```

## Error Codes

### Authentication Errors
| Code | HTTP Status | Description |
|------|-------------|-------------|
| AUTH_001 | 400 | Invalid email format |
| AUTH_002 | 400 | Password does not meet requirements |
| AUTH_003 | 400 | Email already registered |
| AUTH_004 | 401 | Invalid email or password |
| AUTH_005 | 401 | Account not verified |
| AUTH_006 | 401 | Account locked due to too many failed attempts |
| AUTH_007 | 401 | Invalid or expired token |
| AUTH_008 | 401 | Invalid refresh token |
| AUTH_009 | 403 | Account suspended |
| AUTH_010 | 429 | Too many login attempts |

## Security Considerations

### Password Requirements
- Minimum 8 characters
- At least one uppercase letter
- At least one lowercase letter
- At least one number
- At least one special character
- Cannot be common passwords or personal information

### Account Security
- Account lockout after 5 failed login attempts
- Email verification required for new accounts
- Password reset tokens expire in 1 hour
- Refresh tokens are invalidated on password change
- Session management with secure token storage

### Additional Security Headers
```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

## Implementation Example

### Controller Implementation
```csharp
[ApiController]
[Route("api/v1/auth")]
public class AuthController : ControllerBase
{
    private readonly IAuthService _authService;
    private readonly ILogger<AuthController> _logger;

    public AuthController(IAuthService authService, ILogger<AuthController> logger)
    {
        _authService = authService;
        _logger = logger;
    }

    /// <summary>
    /// Register a new user account
    /// </summary>
    [HttpPost("register")]
    [ProducesResponseType(typeof(ApiResponse<AuthResult>), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status400BadRequest)]
    [RateLimit(PerMinute = 3)]
    public async Task<ActionResult<ApiResponse<AuthResult>>> Register([FromBody] RegisterRequest request)
    {
        try
        {
            var result = await _authService.RegisterAsync(request);
            return CreatedAtAction(nameof(GetCurrentUser), new { }, new ApiResponse<AuthResult>
            {
                Data = result,
                Metadata = new RequestMetadata
                {
                    Timestamp = DateTime.UtcNow,
                    RequestId = HttpContext.TraceIdentifier
                }
            });
        }
        catch (ValidationException ex)
        {
            return BadRequest(CreateErrorResponse(ex));
        }
        catch (DuplicateEmailException ex)
        {
            return BadRequest(CreateErrorResponse("AUTH_003", ex.Message, "email", request.Email));
        }
    }

    /// <summary>
    /// Authenticate user and get JWT token
    /// </summary>
    [HttpPost("login")]
    [ProducesResponseType(typeof(ApiResponse<AuthResult>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status401Unauthorized)]
    [RateLimit(PerMinute = 5)]
    public async Task<ActionResult<ApiResponse<AuthResult>>> Login([FromBody] LoginRequest request)
    {
        try
        {
            var result = await _authService.LoginAsync(request);
            return Ok(new ApiResponse<AuthResult>
            {
                Data = result,
                Metadata = new RequestMetadata
                {
                    Timestamp = DateTime.UtcNow,
                    RequestId = HttpContext.TraceIdentifier
                }
            });
        }
        catch (InvalidCredentialsException)
        {
            return Unauthorized(CreateErrorResponse("AUTH_004", "Invalid email or password"));
        }
        catch (AccountNotVerifiedException)
        {
            return Unauthorized(CreateErrorResponse("AUTH_005", "Account not verified"));
        }
        catch (AccountLockedException ex)
        {
            return Unauthorized(CreateErrorResponse("AUTH_006", ex.Message));
        }
    }
}
```

---

These authentication endpoints provide secure, comprehensive user management capabilities for the MealPrep application with proper validation, error handling, and security measures.

*This documentation should be updated when authentication requirements change or new security features are added.*