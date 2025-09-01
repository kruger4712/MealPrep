# Error Codes Reference

## Overview
Comprehensive reference for all error codes returned by the MealPrep API. Each error includes the HTTP status code, error code, description, and suggested resolution.

## Error Response Format

All API errors follow this consistent format:

```json
{
  "success": false,
  "message": "Human-readable error description",
  "errors": [
    {
      "field": "fieldName",
      "message": "Field-specific error message"
    }
  ],
  "errorCode": "ERROR_CODE_CONSTANT",
  "timestamp": "2024-01-15T10:30:00Z",
  "requestId": "uuid-request-identifier",
  "details": {
    "additionalContext": "value"
  }
}
```

---

## Authentication Errors (1000-1099)

### 1001 - INVALID_CREDENTIALS
**HTTP Status:** 401 Unauthorized  
**Description:** Invalid email or password provided during login  
**Resolution:** Verify email and password are correct, check for typos

```json
{
  "success": false,
  "message": "Invalid email or password",
  "errorCode": "INVALID_CREDENTIALS",
  "errors": []
}
```

### 1002 - TOKEN_EXPIRED
**HTTP Status:** 401 Unauthorized  
**Description:** JWT token has expired  
**Resolution:** Refresh token or re-authenticate

```json
{
  "success": false,
  "message": "Authentication token has expired",
  "errorCode": "TOKEN_EXPIRED",
  "details": {
    "expiredAt": "2024-01-15T10:00:00Z"
  }
}
```

### 1003 - TOKEN_INVALID
**HTTP Status:** 401 Unauthorized  
**Description:** JWT token is malformed or invalid  
**Resolution:** Provide valid authentication token

```json
{
  "success": false,
  "message": "Invalid authentication token",
  "errorCode": "TOKEN_INVALID"
}
```

### 1004 - TOKEN_MISSING
**HTTP Status:** 401 Unauthorized  
**Description:** No authentication token provided  
**Resolution:** Include Authorization header with valid JWT token

```json
{
  "success": false,
  "message": "Authentication token required",
  "errorCode": "TOKEN_MISSING"
}
```

### 1005 - ACCOUNT_LOCKED
**HTTP Status:** 403 Forbidden  
**Description:** Account temporarily locked due to multiple failed login attempts  
**Resolution:** Wait for lockout period to expire or request password reset

```json
{
  "success": false,
  "message": "Account temporarily locked due to multiple failed login attempts",
  "errorCode": "ACCOUNT_LOCKED",
  "details": {
    "lockoutUntil": "2024-01-15T11:00:00Z",
    "attemptsRemaining": 0
  }
}
```

### 1006 - ACCOUNT_DISABLED
**HTTP Status:** 403 Forbidden  
**Description:** User account has been disabled  
**Resolution:** Contact support to reactivate account

### 1007 - EMAIL_NOT_VERIFIED
**HTTP Status:** 403 Forbidden  
**Description:** Email address has not been verified  
**Resolution:** Check email for verification link or request new verification email

### 1008 - REFRESH_TOKEN_INVALID
**HTTP Status:** 401 Unauthorized  
**Description:** Refresh token is invalid or expired  
**Resolution:** Re-authenticate with email and password

---

## Validation Errors (2000-2099)

### 2001 - VALIDATION_FAILED
**HTTP Status:** 422 Unprocessable Entity  
**Description:** Request data failed validation  
**Resolution:** Check errors array for specific field validation issues

```json
{
  "success": false,
  "message": "Validation failed",
  "errorCode": "VALIDATION_FAILED",
  "errors": [
    {
      "field": "email",
      "message": "Email address is required"
    },
    {
      "field": "password",
      "message": "Password must be at least 8 characters"
    }
  ]
}
```

### 2002 - REQUIRED_FIELD_MISSING
**HTTP Status:** 400 Bad Request  
**Description:** Required field is missing from request  
**Resolution:** Include all required fields

### 2003 - INVALID_EMAIL_FORMAT
**HTTP Status:** 400 Bad Request  
**Description:** Email address format is invalid  
**Resolution:** Provide valid email address format

### 2004 - PASSWORD_TOO_WEAK
**HTTP Status:** 400 Bad Request  
**Description:** Password does not meet security requirements  
**Resolution:** Use password with minimum 8 characters, including uppercase, lowercase, number, and special character

### 2005 - INVALID_DATE_FORMAT
**HTTP Status:** 400 Bad Request  
**Description:** Date field contains invalid format  
**Resolution:** Use ISO 8601 format (YYYY-MM-DDTHH:mm:ssZ)

### 2006 - INVALID_ENUM_VALUE
**HTTP Status:** 400 Bad Request  
**Description:** Field contains value not in allowed enum list  
**Resolution:** Use one of the allowed values

```json
{
  "success": false,
  "message": "Invalid difficulty level",
  "errorCode": "INVALID_ENUM_VALUE",
  "errors": [
    {
      "field": "difficulty",
      "message": "Must be one of: Easy, Medium, Hard"
    }
  ]
}
```

### 2007 - INVALID_RANGE_VALUE
**HTTP Status:** 400 Bad Request  
**Description:** Numeric field is outside allowed range  
**Resolution:** Provide value within specified range

### 2008 - INVALID_FILE_FORMAT
**HTTP Status:** 400 Bad Request  
**Description:** Uploaded file format not supported  
**Resolution:** Upload file in supported format (JPG, PNG, GIF)

### 2009 - FILE_TOO_LARGE
**HTTP Status:** 413 Payload Too Large  
**Description:** Uploaded file exceeds maximum size limit  
**Resolution:** Reduce file size or compress image

---

## Resource Errors (3000-3099)

### 3001 - RESOURCE_NOT_FOUND
**HTTP Status:** 404 Not Found  
**Description:** Requested resource does not exist  
**Resolution:** Verify resource ID is correct

```json
{
  "success": false,
  "message": "Recipe not found",
  "errorCode": "RESOURCE_NOT_FOUND",
  "details": {
    "resourceType": "Recipe",
    "resourceId": 123
  }
}
```

### 3002 - RESOURCE_ALREADY_EXISTS
**HTTP Status:** 409 Conflict  
**Description:** Resource with same identifier already exists  
**Resolution:** Use different identifier or update existing resource

### 3003 - UNAUTHORIZED_ACCESS
**HTTP Status:** 403 Forbidden  
**Description:** User does not have permission to access resource  
**Resolution:** Ensure user owns resource or has appropriate permissions

### 3004 - RESOURCE_IN_USE
**HTTP Status:** 409 Conflict  
**Description:** Resource cannot be deleted because it's referenced by other resources  
**Resolution:** Remove references before deleting

### 3005 - RECIPE_NOT_FOUND
**HTTP Status:** 404 Not Found  
**Description:** Specific recipe not found  
**Resolution:** Check recipe ID exists and user has access

### 3006 - FAMILY_MEMBER_NOT_FOUND
**HTTP Status:** 404 Not Found  
**Description:** Family member not found  
**Resolution:** Verify family member ID and user ownership

### 3007 - MENU_NOT_FOUND
**HTTP Status:** 404 Not Found  
**Description:** Menu plan not found  
**Resolution:** Check menu ID exists and user has access

### 3008 - INGREDIENT_NOT_FOUND
**HTTP Status:** 404 Not Found  
**Description:** Ingredient not found in database  
**Resolution:** Verify ingredient ID or create new ingredient

---

## Business Logic Errors (4000-4099)

### 4001 - DUPLICATE_EMAIL
**HTTP Status:** 409 Conflict  
**Description:** Email address already registered  
**Resolution:** Use different email or login with existing account

```json
{
  "success": false,
  "message": "Email address already registered",
  "errorCode": "DUPLICATE_EMAIL",
  "details": {
    "email": "user@example.com"
  }
}
```

### 4002 - INSUFFICIENT_PERMISSIONS
**HTTP Status:** 403 Forbidden  
**Description:** User lacks required permissions for action  
**Resolution:** Request appropriate permissions or contact administrator

### 4003 - FAMILY_SIZE_LIMIT_EXCEEDED
**HTTP Status:** 400 Bad Request  
**Description:** Maximum number of family members reached  
**Resolution:** Remove family member or upgrade account

### 4004 - RECIPE_LIMIT_EXCEEDED
**HTTP Status:** 400 Bad Request  
**Description:** Maximum number of recipes reached for account type  
**Resolution:** Delete unused recipes or upgrade account

### 4005 - AI_QUOTA_EXCEEDED
**HTTP Status:** 429 Too Many Requests  
**Description:** AI API usage quota exceeded  
**Resolution:** Wait for quota reset or upgrade account

### 4006 - INVALID_FAMILY_COMPOSITION
**HTTP Status:** 400 Bad Request  
**Description:** Family member configuration is invalid for operation  
**Resolution:** Ensure at least one active family member exists

### 4007 - CONFLICTING_DIETARY_RESTRICTIONS
**HTTP Status:** 400 Bad Request  
**Description:** Dietary restrictions conflict with recipe requirements  
**Resolution:** Modify restrictions or choose different recipe

### 4008 - MENU_DATE_CONFLICT
**HTTP Status:** 409 Conflict  
**Description:** Menu already exists for specified date range  
**Resolution:** Choose different dates or update existing menu

---

## AI Service Errors (5000-5099)

### 5001 - AI_SERVICE_UNAVAILABLE
**HTTP Status:** 503 Service Unavailable  
**Description:** AI service is temporarily unavailable  
**Resolution:** Try again later or use manual meal planning

```json
{
  "success": false,
  "message": "AI service temporarily unavailable",
  "errorCode": "AI_SERVICE_UNAVAILABLE",
  "details": {
    "retryAfter": 300
  }
}
```

### 5002 - AI_REQUEST_TIMEOUT
**HTTP Status:** 504 Gateway Timeout  
**Description:** AI service request timed out  
**Resolution:** Simplify request parameters and try again

### 5003 - AI_QUOTA_EXCEEDED
**HTTP Status:** 429 Too Many Requests  
**Description:** AI usage quota exceeded for user  
**Resolution:** Wait for quota reset or upgrade plan

### 5004 - INVALID_AI_PARAMETERS
**HTTP Status:** 400 Bad Request  
**Description:** AI request parameters are invalid  
**Resolution:** Check family member data and request constraints

### 5005 - AI_GENERATION_FAILED
**HTTP Status:** 500 Internal Server Error  
**Description:** AI failed to generate suggestions  
**Resolution:** Try with different parameters or use manual options

### 5006 - INSUFFICIENT_FAMILY_DATA
**HTTP Status:** 400 Bad Request  
**Description:** Not enough family member data for AI suggestions  
**Resolution:** Complete family member profiles with preferences

### 5007 - AI_CONTENT_FILTERED
**HTTP Status:** 400 Bad Request  
**Description:** AI response was filtered due to content policy  
**Resolution:** Modify request parameters and try again

---

## Rate Limiting Errors (6000-6099)

### 6001 - RATE_LIMIT_EXCEEDED
**HTTP Status:** 429 Too Many Requests  
**Description:** API rate limit exceeded  
**Resolution:** Wait before making additional requests

```json
{
  "success": false,
  "message": "Rate limit exceeded",
  "errorCode": "RATE_LIMIT_EXCEEDED",
  "details": {
    "limit": 100,
    "remaining": 0,
    "resetTime": "2024-01-15T11:00:00Z"
  }
}
```

### 6002 - CONCURRENT_REQUEST_LIMIT
**HTTP Status:** 429 Too Many Requests  
**Description:** Too many concurrent requests from user  
**Resolution:** Reduce concurrent request volume

### 6003 - AI_RATE_LIMIT_EXCEEDED
**HTTP Status:** 429 Too Many Requests  
**Description:** AI service rate limit exceeded  
**Resolution:** Wait longer between AI requests

---

## External Service Errors (7000-7099)

### 7001 - DATABASE_CONNECTION_FAILED
**HTTP Status:** 503 Service Unavailable  
**Description:** Database connection failed  
**Resolution:** Service issue - try again later

### 7002 - EXTERNAL_API_UNAVAILABLE
**HTTP Status:** 503 Service Unavailable  
**Description:** External service dependency unavailable  
**Resolution:** Service issue - try again later

### 7003 - EMAIL_SERVICE_UNAVAILABLE
**HTTP Status:** 503 Service Unavailable  
**Description:** Email service unavailable  
**Resolution:** Email will be sent when service recovers

### 7004 - IMAGE_STORAGE_UNAVAILABLE
**HTTP Status:** 503 Service Unavailable  
**Description:** Image storage service unavailable  
**Resolution:** Try uploading image later

---

## System Errors (8000-8099)

### 8001 - INTERNAL_SERVER_ERROR
**HTTP Status:** 500 Internal Server Error  
**Description:** Unexpected server error occurred  
**Resolution:** Contact support if problem persists

```json
{
  "success": false,
  "message": "An unexpected error occurred",
  "errorCode": "INTERNAL_SERVER_ERROR",
  "requestId": "req_12345",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### 8002 - DATABASE_ERROR
**HTTP Status:** 500 Internal Server Error  
**Description:** Database operation failed  
**Resolution:** Service issue - try again later

### 8003 - CONFIGURATION_ERROR
**HTTP Status:** 500 Internal Server Error  
**Description:** Server configuration error  
**Resolution:** Service issue - contact support

### 8004 - MAINTENANCE_MODE
**HTTP Status:** 503 Service Unavailable  
**Description:** System is in maintenance mode  
**Resolution:** Wait for maintenance to complete

---

## Error Handling Best Practices

### Client-Side Error Handling

```typescript
async function handleApiRequest<T>(request: Promise<T>): Promise<T> {
  try {
    return await request;
  } catch (error) {
    if (error.response) {
      const { status, data } = error.response;
      
      switch (data.errorCode) {
        case 'TOKEN_EXPIRED':
          // Attempt token refresh
          await refreshAuthToken();
          return handleApiRequest(request); // Retry
          
        case 'RATE_LIMIT_EXCEEDED':
          // Wait and retry
          const retryAfter = data.details?.retryAfter || 60;
          await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
          return handleApiRequest(request);
          
        case 'VALIDATION_FAILED':
          // Handle validation errors
          displayValidationErrors(data.errors);
          throw error;
          
        default:
          // Log error and show user-friendly message
          console.error('API Error:', data);
          showErrorMessage(data.message);
          throw error;
      }
    }
    
    // Network or other errors
    console.error('Network Error:', error);
    showErrorMessage('Network error occurred. Please try again.');
    throw error;
  }
}
```

### Retry Logic

```typescript
async function apiWithRetry<T>(
  request: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await request();
    } catch (error) {
      const isRetryable = [
        'AI_SERVICE_UNAVAILABLE',
        'DATABASE_CONNECTION_FAILED',
        'RATE_LIMIT_EXCEEDED'
      ].includes(error.response?.data?.errorCode);
      
      if (!isRetryable || attempt === maxRetries) {
        throw error;
      }
      
      // Exponential backoff
      const delay = baseDelay * Math.pow(2, attempt - 1);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

### Error Categorization

| Category | Error Codes | User Action Required | Retry Recommended |
|----------|-------------|---------------------|-------------------|
| **Authentication** | 1000-1099 | Yes | No |
| **Validation** | 2000-2099 | Yes | No |
| **Resource** | 3000-3099 | Yes | No |
| **Business Logic** | 4000-4099 | Yes | No |
| **AI Service** | 5000-5099 | Sometimes | Yes |
| **Rate Limiting** | 6000-6099 | No | Yes (with delay) |
| **External Services** | 7000-7099 | No | Yes |
| **System** | 8000-8099 | No | Yes |

### Logging Recommendations

```typescript
function logError(error: ApiError, context: any) {
  const logData = {
    timestamp: new Date().toISOString(),
    errorCode: error.errorCode,
    requestId: error.requestId,
    userId: getCurrentUserId(),
    endpoint: context.endpoint,
    method: context.method,
    statusCode: error.statusCode,
    message: error.message,
    details: error.details
  };
  
  // Send to logging service
  analytics.track('API_Error', logData);
  
  // Console log in development
  if (process.env.NODE_ENV === 'development') {
    console.error('API Error:', logData);
  }
}
```

---

## Common Error Scenarios

### Authentication Flow Errors
1. **Invalid Login** ? 1001 INVALID_CREDENTIALS
2. **Expired Session** ? 1002 TOKEN_EXPIRED ? Auto-refresh
3. **Account Locked** ? 1005 ACCOUNT_LOCKED ? Wait or reset

### Recipe Management Errors
1. **Recipe Not Found** ? 3005 RECIPE_NOT_FOUND
2. **Unauthorized Access** ? 3003 UNAUTHORIZED_ACCESS
3. **Validation Failed** ? 2001 VALIDATION_FAILED

### AI Service Errors
1. **Service Down** ? 5001 AI_SERVICE_UNAVAILABLE ? Retry later
2. **Quota Exceeded** ? 5003 AI_QUOTA_EXCEEDED ? Wait or upgrade
3. **Insufficient Data** ? 5006 INSUFFICIENT_FAMILY_DATA ? Complete profiles

### Family Management Errors
1. **Member Not Found** ? 3006 FAMILY_MEMBER_NOT_FOUND
2. **Size Limit** ? 4003 FAMILY_SIZE_LIMIT_EXCEEDED ? Upgrade
3. **Invalid Configuration** ? 4006 INVALID_FAMILY_COMPOSITION

---

## Error Response Examples by HTTP Status

### 400 Bad Request Examples
```json
// Missing required field
{
  "success": false,
  "message": "Recipe name is required",
  "errorCode": "REQUIRED_FIELD_MISSING",
  "errors": [{"field": "name", "message": "Recipe name cannot be empty"}]
}

// Invalid enum value
{
  "success": false,
  "message": "Invalid difficulty level",
  "errorCode": "INVALID_ENUM_VALUE",
  "errors": [{"field": "difficulty", "message": "Must be Easy, Medium, or Hard"}]
}
```

### 401 Unauthorized Examples
```json
// Expired token
{
  "success": false,
  "message": "Authentication token has expired",
  "errorCode": "TOKEN_EXPIRED",
  "details": {"expiredAt": "2024-01-15T10:00:00Z"}
}

// Missing token
{
  "success": false,
  "message": "Authentication required",
  "errorCode": "TOKEN_MISSING"
}
```

### 403 Forbidden Examples
```json
// Insufficient permissions
{
  "success": false,
  "message": "You don't have permission to access this recipe",
  "errorCode": "UNAUTHORIZED_ACCESS",
  "details": {"resourceType": "Recipe", "resourceId": 123}
}

// Account locked
{
  "success": false,
  "message": "Account locked due to failed login attempts",
  "errorCode": "ACCOUNT_LOCKED",
  "details": {"unlockTime": "2024-01-15T11:00:00Z"}
}
```

### 404 Not Found Examples
```json
// Resource not found
{
  "success": false,
  "message": "Recipe not found",
  "errorCode": "RECIPE_NOT_FOUND",
  "details": {"recipeId": 999}
}
```

### 409 Conflict Examples
```json
// Duplicate resource
{
  "success": false,
  "message": "Email already registered",
  "errorCode": "DUPLICATE_EMAIL",
  "details": {"email": "user@example.com"}
}

// Menu date conflict
{
  "success": false,
  "message": "Menu already exists for this date range",
  "errorCode": "MENU_DATE_CONFLICT",
  "details": {"startDate": "2024-01-15", "endDate": "2024-01-21"}
}
```

### 429 Too Many Requests Examples
```json
// Rate limit exceeded
{
  "success": false,
  "message": "Too many requests",
  "errorCode": "RATE_LIMIT_EXCEEDED",
  "details": {
    "limit": 100,
    "remaining": 0,
    "resetTime": "2024-01-15T11:00:00Z"
  }
}

// AI quota exceeded
{
  "success": false,
  "message": "AI usage quota exceeded",
  "errorCode": "AI_QUOTA_EXCEEDED",
  "details": {
    "quotaLimit": 50,
    "quotaUsed": 50,
    "resetTime": "2024-01-16T00:00:00Z"
  }
}
```

### 500 Internal Server Error Examples
```json
// General server error
{
  "success": false,
  "message": "An unexpected error occurred",
  "errorCode": "INTERNAL_SERVER_ERROR",
  "requestId": "req_abc123",
  "timestamp": "2024-01-15T10:30:00Z"
}

// Database error
{
  "success": false,
  "message": "Database operation failed",
  "errorCode": "DATABASE_ERROR",
  "requestId": "req_def456"
}
```

### 503 Service Unavailable Examples
```json
// AI service down
{
  "success": false,
  "message": "AI service temporarily unavailable",
  "errorCode": "AI_SERVICE_UNAVAILABLE",
  "details": {"retryAfter": 300}
}

// Maintenance mode
{
  "success": false,
  "message": "System under maintenance",
  "errorCode": "MAINTENANCE_MODE",
  "details": {
    "estimatedDuration": 1800,
    "maintenanceEnd": "2024-01-15T12:00:00Z"
  }
}
```

---

## Error Code Quick Reference

| Code | Error | HTTP Status | Category |
|------|-------|-------------|----------|
| 1001 | INVALID_CREDENTIALS | 401 | Authentication |
| 1002 | TOKEN_EXPIRED | 401 | Authentication |
| 1003 | TOKEN_INVALID | 401 | Authentication |
| 1004 | TOKEN_MISSING | 401 | Authentication |
| 1005 | ACCOUNT_LOCKED | 403 | Authentication |
| 2001 | VALIDATION_FAILED | 422 | Validation |
| 2008 | INVALID_FILE_FORMAT | 400 | Validation |
| 2009 | FILE_TOO_LARGE | 413 | Validation |
| 3001 | RESOURCE_NOT_FOUND | 404 | Resource |
| 3003 | UNAUTHORIZED_ACCESS | 403 | Resource |
| 3005 | RECIPE_NOT_FOUND | 404 | Resource |
| 4001 | DUPLICATE_EMAIL | 409 | Business Logic |
| 4003 | FAMILY_SIZE_LIMIT_EXCEEDED | 400 | Business Logic |
| 5001 | AI_SERVICE_UNAVAILABLE | 503 | AI Service |
| 5003 | AI_QUOTA_EXCEEDED | 429 | AI Service |
| 6001 | RATE_LIMIT_EXCEEDED | 429 | Rate Limiting |
| 8001 | INTERNAL_SERVER_ERROR | 500 | System |

---

*Last Updated: December 2024*  
*Error codes are continuously updated as the API evolves*