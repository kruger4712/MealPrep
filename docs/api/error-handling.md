# API Error Handling Guide

## Overview
Comprehensive guide for error handling in the MealPrep API, covering error codes, response formats, debugging strategies, and best practices for handling errors gracefully in the AI-powered meal planning system.

## Error Response Format

### Standard Error Response Structure
All API errors follow a consistent JSON structure to ensure predictable error handling across client applications.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more validation errors occurred",
    "details": "The Recipe name field is required",
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_12345-67890-abcdef",
    "path": "/api/recipes",
    "method": "POST"
  },
  "validationErrors": [
    {
      "field": "name",
      "message": "Recipe name is required",
      "code": "REQUIRED_FIELD"
    },
    {
      "field": "ingredients",
      "message": "At least one ingredient is required",
      "code": "MIN_ITEMS_REQUIRED"
    }
  ],
  "supportInfo": {
    "documentation": "https://docs.mealprep.com/api/errors#VALIDATION_ERROR",
    "contact": "support@mealprep.com"
  }
}
```

### Error Response Fields
```yaml
Error Response Schema:
  error:
    code: string           # Unique error identifier (see error codes below)
    message: string        # Human-readable error description
    details: string        # Additional technical details (optional)
    timestamp: string      # ISO 8601 timestamp when error occurred
    requestId: string      # Unique request identifier for tracking
    path: string          # API endpoint that generated the error
    method: string        # HTTP method used in the request
    
  validationErrors:       # Array of field-specific validation errors (optional)
    - field: string       # Field name that failed validation
      message: string     # Validation error message
      code: string        # Specific validation error code
      
  supportInfo:           # Information for getting help (optional)
    documentation: string # Link to relevant documentation
    contact: string      # Support contact information
```

---

## HTTP Status Codes

### Client Error Responses (4xx)

#### 400 Bad Request
**When to use:** Invalid request syntax, malformed request data, or business logic violations.

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": {
    "code": "INVALID_REQUEST_FORMAT",
    "message": "Request body contains invalid JSON",
    "details": "Unexpected token '}' at position 47",
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_12345",
    "path": "/api/recipes",
    "method": "POST"
  }
}
```

**Common scenarios:**
- Invalid JSON format
- Missing required parameters
- Invalid parameter values
- Business rule violations

#### 401 Unauthorized
**When to use:** Authentication is required but missing or invalid.

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json
WWW-Authenticate: Bearer realm="MealPrep API"

{
  "error": {
    "code": "AUTHENTICATION_REQUIRED",
    "message": "Authentication credentials are required",
    "details": "No Authorization header provided",
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_12345",
    "path": "/api/recipes",
    "method": "GET"
  },
  "supportInfo": {
    "documentation": "https://docs.mealprep.com/api/authentication",
    "authEndpoint": "/auth/login"
  }
}
```

**Common scenarios:**
- Missing Authorization header
- Invalid or expired JWT token
- Malformed authentication credentials

#### 403 Forbidden
**When to use:** User is authenticated but lacks permission for the requested resource.

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": {
    "code": "INSUFFICIENT_PERMISSIONS",
    "message": "You do not have permission to access this resource",
    "details": "Required permission: recipes:delete, User has: recipes:read",
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_12345",
    "path": "/api/recipes/123",
    "method": "DELETE"
  },
  "requiredPermissions": ["recipes:delete"],
  "userPermissions": ["recipes:read", "recipes:write"]
}
```

**Common scenarios:**
- User trying to access another user's private data
- Insufficient role-based permissions
- Family member access restrictions

#### 404 Not Found
**When to use:** Requested resource does not exist.

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "The requested recipe was not found",
    "details": "Recipe with ID 999 does not exist or has been deleted",
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_12345",
    "path": "/api/recipes/999",
    "method": "GET"
  },
  "suggestion": "Check the recipe ID and ensure it exists"
}
```

**Common scenarios:**
- Recipe, user, or family member not found
- Deleted resources
- Invalid endpoint paths

#### 409 Conflict
**When to use:** Request conflicts with current state of the resource.

```http
HTTP/1.1 409 Conflict
Content-Type: application/json

{
  "error": {
    "code": "RESOURCE_CONFLICT",
    "message": "Recipe name already exists",
    "details": "A recipe named 'Chicken Parmesan' already exists in your collection",
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_12345",
    "path": "/api/recipes",
    "method": "POST"
  },
  "conflictingResource": {
    "id": 456,
    "name": "Chicken Parmesan",
    "createdAt": "2024-01-10T15:20:00Z"
  }
}
```

**Common scenarios:**
- Duplicate recipe names
- Concurrent modification conflicts
- Business rule violations

#### 422 Unprocessable Entity
**When to use:** Request is well-formed but contains semantic errors.

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Recipe validation failed",
    "details": "Recipe contains invalid data that cannot be processed",
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_12345",
    "path": "/api/recipes",
    "method": "POST"
  },
  "validationErrors": [
    {
      "field": "prepTimeMinutes",
      "message": "Preparation time cannot exceed 24 hours",
      "code": "VALUE_OUT_OF_RANGE",
      "rejectedValue": 1500
    },
    {
      "field": "ingredients[0].quantity",
      "message": "Ingredient quantity must be positive",
      "code": "POSITIVE_NUMBER_REQUIRED",
      "rejectedValue": -2
    }
  ]
}
```

#### 429 Too Many Requests
**When to use:** Rate limiting threshold exceeded.

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 60
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640995200

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "API rate limit exceeded",
    "details": "You have exceeded the allowed 1000 requests per hour",
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_12345",
    "path": "/api/ai/suggestions",
    "method": "POST"
  },
  "rateLimitInfo": {
    "limit": 1000,
    "remaining": 0,
    "resetTime": "2024-01-15T11:00:00Z",
    "retryAfter": 60
  }
}
```

### Server Error Responses (5xx)

#### 500 Internal Server Error
**When to use:** Unexpected server-side errors.

```http
HTTP/1.1 500 Internal Server Error
Content-Type: application/json

{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "An unexpected error occurred while processing your request",
    "details": "Please try again later or contact support if the problem persists",
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_12345",
    "path": "/api/recipes",
    "method": "POST"
  },
  "supportInfo": {
    "contact": "support@mealprep.com",
    "reportBug": "https://mealprep.com/support/bug-report"
  }
}
```

#### 502 Bad Gateway
**When to use:** Upstream service failures (AI service, database, etc.).

```http
HTTP/1.1 502 Bad Gateway
Content-Type: application/json

{
  "error": {
    "code": "UPSTREAM_SERVICE_ERROR",
    "message": "AI suggestion service is temporarily unavailable",
    "details": "The AI meal planning service is experiencing issues. Please try again later.",
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_12345",
    "path": "/api/ai/suggestions",
    "method": "POST"
  },
  "serviceStatus": {
    "service": "gemini-ai",
    "status": "degraded",
    "estimatedRecovery": "2024-01-15T11:00:00Z"
  }
}
```

#### 503 Service Unavailable
**When to use:** Service is temporarily down for maintenance.

```http
HTTP/1.1 503 Service Unavailable
Content-Type: application/json
Retry-After: 300

{
  "error": {
    "code": "SERVICE_MAINTENANCE",
    "message": "Service is temporarily unavailable for maintenance",
    "details": "We are performing scheduled maintenance. Service will be restored shortly.",
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_12345",
    "path": "/api/recipes",
    "method": "GET"
  },
  "maintenanceInfo": {
    "scheduledEnd": "2024-01-15T11:00:00Z",
    "retryAfter": 300
  }
}
```

---

## Error Codes Reference

### Authentication and Authorization Errors
```yaml
AUTHENTICATION_REQUIRED:
  httpStatus: 401
  description: "No authentication credentials provided"
  commonCauses:
    - Missing Authorization header
    - Empty or null token
  solution: "Include valid JWT token in Authorization header"

INVALID_CREDENTIALS:
  httpStatus: 401
  description: "Authentication credentials are invalid"
  commonCauses:
    - Wrong email/password combination
    - Expired or malformed JWT token
  solution: "Verify credentials and re-authenticate if needed"

TOKEN_EXPIRED:
  httpStatus: 401
  description: "JWT token has expired"
  commonCauses:
    - Token past expiration time
    - System clock skew
  solution: "Refresh token or re-authenticate"

INSUFFICIENT_PERMISSIONS:
  httpStatus: 403
  description: "User lacks required permissions"
  commonCauses:
    - User role doesn't include required permission
    - Attempting to access other user's private data
  solution: "Contact admin for permission upgrade or access own resources"

ACCOUNT_SUSPENDED:
  httpStatus: 403
  description: "User account has been suspended"
  commonCauses:
    - Policy violations
    - Administrative action
  solution: "Contact support to resolve account issues"
```

### Validation Errors
```yaml
VALIDATION_ERROR:
  httpStatus: 422
  description: "Request data failed validation"
  commonCauses:
    - Missing required fields
    - Invalid data formats
    - Business rule violations
  solution: "Review validation errors and correct request data"

REQUIRED_FIELD:
  httpStatus: 422
  description: "Required field is missing or empty"
  commonCauses:
    - Null or undefined values
    - Empty strings for required text fields
  solution: "Provide valid value for the required field"

INVALID_FORMAT:
  httpStatus: 422
  description: "Field value has invalid format"
  commonCauses:
    - Invalid email format
    - Malformed URLs
    - Invalid date/time strings
  solution: "Ensure field value matches expected format"

VALUE_OUT_OF_RANGE:
  httpStatus: 422
  description: "Numeric value is outside acceptable range"
  commonCauses:
    - Negative values where positive required
    - Values exceeding maximum limits
  solution: "Provide value within acceptable range"

DUPLICATE_VALUE:
  httpStatus: 409
  description: "Value already exists where uniqueness required"
  commonCauses:
    - Duplicate recipe names
    - Duplicate email addresses
  solution: "Use a unique value for the field"
```

### Resource Errors
```yaml
RESOURCE_NOT_FOUND:
  httpStatus: 404
  description: "Requested resource does not exist"
  commonCauses:
    - Invalid resource ID
    - Resource has been deleted
    - User lacks access to resource
  solution: "Verify resource ID and user permissions"

RESOURCE_CONFLICT:
  httpStatus: 409
  description: "Operation conflicts with resource state"
  commonCauses:
    - Concurrent modifications
    - State transition not allowed
  solution: "Refresh resource state and retry operation"

RESOURCE_LOCKED:
  httpStatus: 423
  description: "Resource is temporarily locked"
  commonCauses:
    - Another operation in progress
    - Administrative lock
  solution: "Wait and retry, or contact support if persistent"
```

### AI Service Errors
```yaml
AI_SERVICE_UNAVAILABLE:
  httpStatus: 502
  description: "AI service is temporarily unavailable"
  commonCauses:
    - Gemini API service down
    - Network connectivity issues
    - Service rate limits exceeded
  solution: "Try again later or use fallback suggestions"

AI_QUOTA_EXCEEDED:
  httpStatus: 429
  description: "AI service quota has been exceeded"
  commonCauses:
    - Monthly AI usage limit reached
    - Subscription plan limits
  solution: "Upgrade plan or wait for quota reset"

AI_PROCESSING_ERROR:
  httpStatus: 500
  description: "Error processing AI request"
  commonCauses:
    - Invalid prompt data
    - AI model processing failure
  solution: "Retry with different parameters or contact support"

INVALID_PERSONA_DATA:
  httpStatus: 422
  description: "Family persona data is invalid for AI processing"
  commonCauses:
    - Incomplete family member profiles
    - Conflicting dietary restrictions
  solution: "Complete family member profiles with valid data"
```

### Rate Limiting Errors
```yaml
RATE_LIMIT_EXCEEDED:
  httpStatus: 429
  description: "API rate limit exceeded"
  commonCauses:
    - Too many requests in time window
    - Burst limits exceeded
  solution: "Implement exponential backoff and respect rate limits"

CONCURRENT_REQUEST_LIMIT:
  httpStatus: 429
  description: "Too many concurrent requests"
  commonCauses:
    - Multiple simultaneous API calls
    - Poorly designed client retry logic
  solution: "Limit concurrent requests and implement proper queuing"
```

---

## Error Handling Implementation

### Backend Error Handling Middleware
```csharp
public class ErrorHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ErrorHandlingMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var requestId = context.TraceIdentifier;
        var path = context.Request.Path.Value;
        var method = context.Request.Method;

        _logger.LogError(exception, "Unhandled exception occurred. RequestId: {RequestId}", requestId);

        var errorResponse = exception switch
        {
            ValidationException validationEx => CreateValidationErrorResponse(validationEx, requestId, path, method),
            UnauthorizedException _ => CreateUnauthorizedErrorResponse(requestId, path, method),
            ForbiddenException forbiddenEx => CreateForbiddenErrorResponse(forbiddenEx, requestId, path, method),
            NotFoundException notFoundEx => CreateNotFoundErrorResponse(notFoundEx, requestId, path, method),
            ConflictException conflictEx => CreateConflictErrorResponse(conflictEx, requestId, path, method),
            RateLimitExceededException rateLimitEx => CreateRateLimitErrorResponse(rateLimitEx, requestId, path, method),
            AiServiceException aiEx => CreateAiServiceErrorResponse(aiEx, requestId, path, method),
            _ => CreateInternalServerErrorResponse(exception, requestId, path, method)
        };

        context.Response.StatusCode = errorResponse.StatusCode;
        context.Response.ContentType = "application/json";

        var jsonResponse = JsonSerializer.Serialize(errorResponse.Body, new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        });

        await context.Response.WriteAsync(jsonResponse);
    }

    private ErrorResponse CreateValidationErrorResponse(ValidationException exception, string requestId, string path, string method)
    {
        return new ErrorResponse
        {
            StatusCode = 422,
            Body = new
            {
                error = new
                {
                    code = "VALIDATION_ERROR",
                    message = "One or more validation errors occurred",
                    details = exception.Message,
                    timestamp = DateTime.UtcNow.ToString("O"),
                    requestId,
                    path,
                    method
                },
                validationErrors = exception.ValidationErrors.Select(ve => new
                {
                    field = ve.PropertyName,
                    message = ve.ErrorMessage,
                    code = ve.ErrorCode,
                    rejectedValue = ve.AttemptedValue
                }),
                supportInfo = new
                {
                    documentation = "https://docs.mealprep.com/api/errors#VALIDATION_ERROR"
                }
            }
        };
    }

    private ErrorResponse CreateAiServiceErrorResponse(AiServiceException exception, string requestId, string path, string method)
    {
        var (errorCode, statusCode) = exception.ServiceError switch
        {
            AiServiceError.ServiceUnavailable => ("AI_SERVICE_UNAVAILABLE", 502),
            AiServiceError.QuotaExceeded => ("AI_QUOTA_EXCEEDED", 429),
            AiServiceError.ProcessingError => ("AI_PROCESSING_ERROR", 500),
            AiServiceError.InvalidPersonaData => ("INVALID_PERSONA_DATA", 422),
            _ => ("AI_SERVICE_ERROR", 500)
        };

        return new ErrorResponse
        {
            StatusCode = statusCode,
            Body = new
            {
                error = new
                {
                    code = errorCode,
                    message = exception.Message,
                    details = exception.Details,
                    timestamp = DateTime.UtcNow.ToString("O"),
                    requestId,
                    path,
                    method
                },
                serviceStatus = new
                {
                    service = "gemini-ai",
                    status = exception.ServiceStatus,
                    estimatedRecovery = exception.EstimatedRecovery?.ToString("O")
                }
            }
        };
    }
}
```

### Custom Exception Classes
```csharp
public class ValidationException : Exception
{
    public IEnumerable<ValidationError> ValidationErrors { get; }

    public ValidationException(IEnumerable<ValidationError> validationErrors)
        : base("One or more validation errors occurred")
    {
        ValidationErrors = validationErrors;
    }
}

public class AiServiceException : Exception
{
    public AiServiceError ServiceError { get; }
    public string Details { get; }
    public string ServiceStatus { get; }
    public DateTime? EstimatedRecovery { get; }

    public AiServiceException(
        AiServiceError serviceError, 
        string message, 
        string details = null,
        string serviceStatus = "unavailable",
        DateTime? estimatedRecovery = null) 
        : base(message)
    {
        ServiceError = serviceError;
        Details = details;
        ServiceStatus = serviceStatus;
        EstimatedRecovery = estimatedRecovery;
    }
}

public enum AiServiceError
{
    ServiceUnavailable,
    QuotaExceeded,
    ProcessingError,
    InvalidPersonaData
}
```

---

## Client-Side Error Handling

### JavaScript/TypeScript Error Handling
```typescript
export class ApiError extends Error {
    public readonly statusCode: number;
    public readonly errorCode: string;
    public readonly requestId: string;
    public readonly validationErrors?: ValidationError[];
    public readonly retryAfter?: number;

    constructor(
        message: string,
        statusCode: number,
        errorData: any
    ) {
        super(message);
        this.name = 'ApiError';
        this.statusCode = statusCode;
        this.errorCode = errorData.error?.code || 'UNKNOWN_ERROR';
        this.requestId = errorData.error?.requestId;
        this.validationErrors = errorData.validationErrors;
        this.retryAfter = errorData.rateLimitInfo?.retryAfter;
    }

    public isRetryable(): boolean {
        return this.statusCode >= 500 || 
               this.statusCode === 429 || 
               this.errorCode === 'AI_SERVICE_UNAVAILABLE';
    }

    public isAuthenticationError(): boolean {
        return this.statusCode === 401;
    }

    public isValidationError(): boolean {
        return this.statusCode === 422 && this.validationErrors?.length > 0;
    }

    public getRetryDelay(): number {
        if (this.retryAfter) {
            return this.retryAfter * 1000; // Convert to milliseconds
        }

        // Exponential backoff for server errors
        if (this.statusCode >= 500) {
            return Math.min(1000 * Math.pow(2, Math.random() * 3), 30000);
        }

        return 0;
    }
}

export class MealPrepApiClient {
    private async handleApiError(response: Response): Promise<never> {
        let errorData: any;
        
        try {
            errorData = await response.json();
        } catch {
            errorData = {
                error: {
                    code: 'PARSE_ERROR',
                    message: 'Failed to parse error response'
                }
            };
        }

        const apiError = new ApiError(
            errorData.error?.message || 'An error occurred',
            response.status,
            errorData
        );

        // Log error for debugging
        console.error('API Error:', {
            statusCode: apiError.statusCode,
            errorCode: apiError.errorCode,
            requestId: apiError.requestId,
            message: apiError.message
        });

        throw apiError;
    }

    private async makeRequestWithRetry<T>(
        url: string,
        options: RequestInit,
        maxRetries: number = 3
    ): Promise<T> {
        let lastError: ApiError;

        for (let attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                const response = await fetch(url, options);

                if (!response.ok) {
                    await this.handleApiError(response);
                }

                return await response.json();
            } catch (error) {
                if (error instanceof ApiError) {
                    lastError = error;

                    // Don't retry non-retryable errors
                    if (!error.isRetryable() || attempt === maxRetries) {
                        throw error;
                    }

                    // Handle authentication errors
                    if (error.isAuthenticationError() && this.refreshToken) {
                        const refreshed = await this.refreshAccessToken();
                        if (refreshed) {
                            // Update Authorization header and retry
                            options.headers = {
                                ...options.headers,
                                'Authorization': `Bearer ${this.accessToken}`
                            };
                            continue;
                        }
                    }

                    // Wait before retrying
                    const delay = error.getRetryDelay();
                    if (delay > 0) {
                        await new Promise(resolve => setTimeout(resolve, delay));
                    }
                } else {
                    throw error;
                }
            }
        }

        throw lastError;
    }
}
```

### React Error Boundary
```tsx
interface ErrorBoundaryState {
    hasError: boolean;
    error?: ApiError;
}

export class ApiErrorBoundary extends React.Component<
    React.PropsWithChildren<{}>,
    ErrorBoundaryState
> {
    constructor(props: React.PropsWithChildren<{}>) {
        super(props);
        this.state = { hasError: false };
    }

    static getDerivedStateFromError(error: Error): ErrorBoundaryState {
        if (error instanceof ApiError) {
            return { hasError: true, error };
        }
        return { hasError: true };
    }

    componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
        console.error('API Error caught by boundary:', error, errorInfo);
        
        // Report to error tracking service
        this.reportError(error, errorInfo);
    }

    private reportError(error: Error, errorInfo: React.ErrorInfo) {
        // Integration with error tracking service (e.g., Sentry)
        if (error instanceof ApiError) {
            // Report structured API error
            console.error('API Error Details:', {
                errorCode: error.errorCode,
                statusCode: error.statusCode,
                requestId: error.requestId,
                validationErrors: error.validationErrors
            });
        }
    }

    render() {
        if (this.state.hasError) {
            return <ErrorFallback error={this.state.error} />;
        }

        return this.props.children;
    }
}

const ErrorFallback: React.FC<{ error?: ApiError }> = ({ error }) => {
    const getErrorMessage = () => {
        if (!error) return 'An unexpected error occurred';

        switch (error.errorCode) {
            case 'AUTHENTICATION_REQUIRED':
            case 'TOKEN_EXPIRED':
                return 'Please log in again to continue';
            case 'INSUFFICIENT_PERMISSIONS':
                return 'You don\'t have permission to access this feature';
            case 'AI_SERVICE_UNAVAILABLE':
                return 'AI suggestions are temporarily unavailable. Please try again later';
            case 'RATE_LIMIT_EXCEEDED':
                return 'Too many requests. Please wait before trying again';
            default:
                return error.message;
        }
    };

    const getActionButton = () => {
        switch (error?.errorCode) {
            case 'AUTHENTICATION_REQUIRED':
            case 'TOKEN_EXPIRED':
                return (
                    <button onClick={() => window.location.href = '/login'}>
                        Log In
                    </button>
                );
            case 'RATE_LIMIT_EXCEEDED':
                const retryAfter = error.retryAfter || 60;
                return (
                    <button onClick={() => window.location.reload()}>
                        Retry in {retryAfter}s
                    </button>
                );
            default:
                return (
                    <button onClick={() => window.location.reload()}>
                        Try Again
                    </button>
                );
        }
    };

    return (
        <div className="error-fallback">
            <h2>Something went wrong</h2>
            <p>{getErrorMessage()}</p>
            {error?.requestId && (
                <p className="request-id">Request ID: {error.requestId}</p>
            )}
            {getActionButton()}
        </div>
    );
};
```

---

## Debugging and Troubleshooting

### Request ID Tracking
Every API response includes a unique `requestId` that can be used to trace the request through logs and debugging tools.

```http
GET /api/recipes/123
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

HTTP/1.1 200 OK
X-Request-ID: req_67890-abcdef-123456
Content-Type: application/json

{
  "id": 123,
  "name": "Chicken Parmesan",
  // ... recipe data
}
```

### Logging Integration
```csharp
public class ApiLoggingService
{
    private readonly ILogger<ApiLoggingService> _logger;

    public void LogApiError(HttpContext context, Exception exception, string requestId)
    {
        var logData = new
        {
            RequestId = requestId,
            Method = context.Request.Method,
            Path = context.Request.Path.Value,
            QueryString = context.Request.QueryString.Value,
            UserAgent = context.Request.Headers["User-Agent"].ToString(),
            UserId = context.User?.FindFirst("userId")?.Value,
            IPAddress = GetClientIPAddress(context),
            Exception = new
            {
                Type = exception.GetType().Name,
                Message = exception.Message,
                StackTrace = exception.StackTrace
            }
        };

        _logger.LogError(exception, "API Error: {LogData}", JsonSerializer.Serialize(logData));
    }
}
```

### Health Check Endpoints
```csharp
[HttpGet("/health")]
public async Task<IActionResult> HealthCheck()
{
    var healthChecks = new Dictionary<string, object>
    {
        ["status"] = "healthy",
        ["timestamp"] = DateTime.UtcNow.ToString("O"),
        ["version"] = GetType().Assembly.GetName().Version?.ToString()
    };

    try
    {
        // Check database connectivity
        var dbHealthy = await _healthCheckService.CheckDatabaseAsync();
        healthChecks["database"] = dbHealthy ? "healthy" : "unhealthy";

        // Check AI service connectivity
        var aiHealthy = await _healthCheckService.CheckAiServiceAsync();
        healthChecks["aiService"] = aiHealthy ? "healthy" : "degraded";

        // Check overall status
        var overallHealthy = dbHealthy && aiHealthy;
        healthChecks["status"] = overallHealthy ? "healthy" : "degraded";

        return Ok(healthChecks);
    }
    catch (Exception ex)
    {
        healthChecks["status"] = "unhealthy";
        healthChecks["error"] = ex.Message;

        return StatusCode(503, healthChecks);
    }
}
```

---

*Last Updated: December 2024*  
*Error handling guide continuously updated with new error scenarios and best practices*