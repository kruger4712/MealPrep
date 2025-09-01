# API Versioning Strategy

## Overview
Comprehensive API versioning strategy for the MealPrep application, covering versioning policies, implementation approaches, migration strategies, and best practices for maintaining backward compatibility while evolving the AI-powered meal planning API.

## Versioning Philosophy

### Core Principles
- **Backward Compatibility**: Minimize breaking changes to protect existing integrations
- **Predictable Evolution**: Clear versioning scheme that clients can rely on
- **Graceful Deprecation**: Gradual phase-out of old versions with adequate notice
- **Developer Experience**: Simple, intuitive versioning that reduces friction
- **Forward Compatibility**: Design current versions to accommodate future changes

### Versioning Strategy Overview
```yaml
Versioning Approach: Semantic Versioning (SemVer) adapted for APIs
Format: v{MAJOR}.{MINOR}.{PATCH}
Implementation: Header-based versioning with URL fallback
Default Behavior: Latest stable version for new clients
Deprecation Policy: 12-month minimum support for major versions
```

---

## Semantic Versioning for APIs

### Version Number Format
```
API Version: v{MAJOR}.{MINOR}.{PATCH}
Examples: v1.0.0, v1.2.1, v2.0.0
```

### Version Component Definitions

#### MAJOR Version (Breaking Changes)
**When to increment:**
- Removing endpoints or fields
- Changing response data structures
- Modifying authentication/authorization requirements
- Changing error response formats
- Altering fundamental API behavior

**Examples:**
```yaml
v1.x.x to v2.0.0:
  - Remove deprecated /recipes/bulk endpoint
  - Change authentication from API keys to JWT only
  - Restructure family member response format
  - Update AI suggestion response schema
```

#### MINOR Version (Backward Compatible Features)
**When to increment:**
- Adding new endpoints
- Adding optional fields to requests
- Adding fields to responses
- Introducing new optional headers
- Enhancing existing functionality without breaking changes

**Examples:**
```yaml
v1.2.x to v1.3.0:
  - Add /recipes/{id}/nutrition endpoint
  - Add optional 'tags' field to recipe creation
  - Include 'estimatedCost' in recipe responses
  - Add AI confidence scores to suggestions
```

#### PATCH Version (Bug Fixes)
**When to increment:**
- Fixing bugs without changing API contracts
- Performance improvements
- Security patches
- Documentation updates
- Internal refactoring

**Examples:**
```yaml
v1.2.1 to v1.2.2:
  - Fix ingredient quantity calculation bug
  - Improve AI suggestion response times
  - Security patch for JWT validation
  - Fix timezone handling in meal planning
```

---

## Version Implementation

### Header-Based Versioning (Primary)
**Request Headers:**
```http
GET /api/recipes
Accept: application/json
API-Version: v1.2.0
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response Headers:**
```http
HTTP/1.1 200 OK
Content-Type: application/json
API-Version: v1.2.0
API-Supported-Versions: v1.0.0, v1.1.0, v1.2.0, v1.3.0
API-Latest-Version: v1.3.0
API-Deprecated-Versions: v1.0.0
```

### URL-Based Versioning (Fallback)
```http
GET /api/v1/recipes
GET /api/v1.2/recipes  # Specific minor version
GET /api/v2/recipes    # Major version
```

### Middleware Implementation
```csharp
public class ApiVersioningMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IApiVersionService _versionService;
    private readonly ILogger<ApiVersioningMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        var requestedVersion = ExtractApiVersion(context);
        var resolvedVersion = await _versionService.ResolveVersionAsync(requestedVersion);

        // Set version context for controllers
        context.Items["ApiVersion"] = resolvedVersion;
        
        // Add version headers to response
        context.Response.OnStarting(() =>
        {
            AddVersionHeaders(context.Response, resolvedVersion);
            return Task.CompletedTask;
        });

        // Check if version is deprecated
        if (resolvedVersion.IsDeprecated)
        {
            await LogDeprecatedVersionUsage(context, resolvedVersion);
            AddDeprecationHeaders(context.Response, resolvedVersion);
        }

        await _next(context);
    }

    private ApiVersion ExtractApiVersion(HttpContext context)
    {
        // Priority: Header > URL > Default
        var headerVersion = context.Request.Headers["API-Version"].FirstOrDefault();
        if (!string.IsNullOrEmpty(headerVersion))
        {
            return ApiVersion.Parse(headerVersion);
        }

        var urlVersion = ExtractVersionFromUrl(context.Request.Path);
        if (urlVersion != null)
        {
            return urlVersion;
        }

        return _versionService.GetDefaultVersion();
    }

    private void AddVersionHeaders(HttpResponse response, ResolvedApiVersion version)
    {
        response.Headers.Add("API-Version", version.RequestedVersion.ToString());
        response.Headers.Add("API-Resolved-Version", version.ActualVersion.ToString());
        response.Headers.Add("API-Supported-Versions", string.Join(", ", version.SupportedVersions));
        response.Headers.Add("API-Latest-Version", version.LatestVersion.ToString());
        
        if (version.DeprecatedVersions.Any())
        {
            response.Headers.Add("API-Deprecated-Versions", string.Join(", ", version.DeprecatedVersions));
        }
    }

    private void AddDeprecationHeaders(HttpResponse response, ResolvedApiVersion version)
    {
        response.Headers.Add("Deprecation", "true");
        response.Headers.Add("Sunset", version.SunsetDate?.ToString("R"));
        response.Headers.Add("Link", $"<{version.MigrationGuideUrl}>; rel=\"migration-guide\"");
        
        var warningMessage = $"API version {version.ActualVersion} is deprecated. " +
                           $"Please migrate to version {version.LatestVersion}. " +
                           $"Support ends on {version.SunsetDate:yyyy-MM-dd}.";
        
        response.Headers.Add("Warning", $"299 - \"{warningMessage}\"");
    }
}
```

### Version Resolution Service
```csharp
public class ApiVersionService : IApiVersionService
{
    private readonly List<ApiVersionInfo> _supportedVersions;
    private readonly ApiVersionInfo _defaultVersion;

    public ApiVersionService()
    {
        _supportedVersions = new List<ApiVersionInfo>
        {
            new ApiVersionInfo
            {
                Version = new ApiVersion(1, 0, 0),
                Status = VersionStatus.Deprecated,
                DeprecatedDate = new DateTime(2024, 6, 1),
                SunsetDate = new DateTime(2024, 12, 1),
                MigrationGuideUrl = "https://docs.mealprep.com/api/migrations/v1.0-to-v1.3"
            },
            new ApiVersionInfo
            {
                Version = new ApiVersion(1, 1, 0),
                Status = VersionStatus.Supported,
                ReleaseDate = new DateTime(2024, 3, 15)
            },
            new ApiVersionInfo
            {
                Version = new ApiVersion(1, 2, 0),
                Status = VersionStatus.Supported,
                ReleaseDate = new DateTime(2024, 6, 1)
            },
            new ApiVersionInfo
            {
                Version = new ApiVersion(1, 3, 0),
                Status = VersionStatus.Current,
                ReleaseDate = new DateTime(2024, 9, 1),
                IsDefault = true
            }
        };

        _defaultVersion = _supportedVersions.First(v => v.IsDefault);
    }

    public async Task<ResolvedApiVersion> ResolveVersionAsync(ApiVersion requestedVersion)
    {
        var exactMatch = _supportedVersions.FirstOrDefault(v => v.Version.Equals(requestedVersion));
        if (exactMatch != null)
        {
            return new ResolvedApiVersion
            {
                RequestedVersion = requestedVersion,
                ActualVersion = exactMatch.Version,
                VersionInfo = exactMatch,
                IsDeprecated = exactMatch.Status == VersionStatus.Deprecated,
                SupportedVersions = _supportedVersions.Select(v => v.Version).ToList(),
                LatestVersion = GetLatestVersion(),
                DeprecatedVersions = GetDeprecatedVersions()
            };
        }

        // Find compatible version (same major, highest compatible minor)
        var compatibleVersion = FindCompatibleVersion(requestedVersion);
        if (compatibleVersion != null)
        {
            return new ResolvedApiVersion
            {
                RequestedVersion = requestedVersion,
                ActualVersion = compatibleVersion.Version,
                VersionInfo = compatibleVersion,
                IsDeprecated = compatibleVersion.Status == VersionStatus.Deprecated,
                SupportedVersions = _supportedVersions.Select(v => v.Version).ToList(),
                LatestVersion = GetLatestVersion(),
                DeprecatedVersions = GetDeprecatedVersions()
            };
        }

        throw new UnsupportedApiVersionException(
            $"API version {requestedVersion} is not supported. " +
            $"Supported versions: {string.Join(", ", _supportedVersions.Select(v => v.Version))}"
        );
    }

    private ApiVersionInfo FindCompatibleVersion(ApiVersion requestedVersion)
    {
        // Find the highest supported version with the same major version
        // that is less than or equal to the requested version
        return _supportedVersions
            .Where(v => v.Version.Major == requestedVersion.Major && 
                       v.Version <= requestedVersion &&
                       v.Status != VersionStatus.Discontinued)
            .OrderByDescending(v => v.Version)
            .FirstOrDefault();
    }
}
```

---

## Version-Specific Response Handling

### Controller Versioning
```csharp
[ApiController]
[Route("api/recipes")]
public class RecipesController : ControllerBase
{
    private readonly IRecipeService _recipeService;
    private readonly IApiVersionContext _versionContext;

    [HttpGet("{id}")]
    public async Task<IActionResult> GetRecipe(int id)
    {
        var recipe = await _recipeService.GetRecipeByIdAsync(id);
        if (recipe == null)
        {
            return NotFound();
        }

        var apiVersion = _versionContext.GetCurrentVersion();
        var response = MapRecipeToResponse(recipe, apiVersion);
        
        return Ok(response);
    }

    private object MapRecipeToResponse(Recipe recipe, ApiVersion version)
    {
        return version.Major switch
        {
            1 => MapToV1Response(recipe, version),
            2 => MapToV2Response(recipe, version),
            _ => throw new UnsupportedApiVersionException($"Unsupported version: {version}")
        };
    }

    private object MapToV1Response(Recipe recipe, ApiVersion version)
    {
        var baseResponse = new
        {
            id = recipe.Id,
            name = recipe.Name,
            description = recipe.Description,
            instructions = recipe.Instructions,
            prepTimeMinutes = recipe.PrepTimeMinutes,
            cookTimeMinutes = recipe.CookTimeMinutes,
            servings = recipe.Servings,
            ingredients = recipe.Ingredients.Select(i => new
            {
                id = i.Id,
                name = i.Name,
                quantity = i.Quantity,
                unit = i.Unit
            }),
            createdAt = recipe.CreatedAt,
            updatedAt = recipe.UpdatedAt
        };

        // Version-specific enhancements
        if (version >= new ApiVersion(1, 1, 0))
        {
            return new
            {
                baseResponse.id,
                baseResponse.name,
                baseResponse.description,
                baseResponse.instructions,
                baseResponse.prepTimeMinutes,
                baseResponse.cookTimeMinutes,
                baseResponse.servings,
                baseResponse.ingredients,
                baseResponse.createdAt,
                baseResponse.updatedAt,
                // Added in v1.1.0
                difficulty = recipe.Difficulty.ToString(),
                cuisine = recipe.Cuisine
            };
        }

        if (version >= new ApiVersion(1, 2, 0))
        {
            return new
            {
                baseResponse.id,
                baseResponse.name,
                baseResponse.description,
                baseResponse.instructions,
                baseResponse.prepTimeMinutes,
                baseResponse.cookTimeMinutes,
                baseResponse.servings,
                baseResponse.ingredients,
                baseResponse.createdAt,
                baseResponse.updatedAt,
                difficulty = recipe.Difficulty.ToString(),
                cuisine = recipe.Cuisine,
                // Added in v1.2.0
                estimatedCost = recipe.EstimatedCost,
                nutritionInfo = recipe.NutritionInfo,
                tags = recipe.Tags
            };
        }

        if (version >= new ApiVersion(1, 3, 0))
        {
            return new
            {
                baseResponse.id,
                baseResponse.name,
                baseResponse.description,
                baseResponse.instructions,
                baseResponse.prepTimeMinutes,
                baseResponse.cookTimeMinutes,
                baseResponse.servings,
                baseResponse.ingredients,
                baseResponse.createdAt,
                baseResponse.updatedAt,
                difficulty = recipe.Difficulty.ToString(),
                cuisine = recipe.Cuisine,
                estimatedCost = recipe.EstimatedCost,
                nutritionInfo = recipe.NutritionInfo,
                tags = recipe.Tags,
                // Added in v1.3.0
                ratings = new
                {
                    average = recipe.AverageRating,
                    count = recipe.RatingCount
                },
                familyFitScore = recipe.FamilyFitScore
            };
        }

        return baseResponse;
    }
}
```

### AI Suggestions Versioning
```csharp
[HttpPost("suggestions")]
public async Task<IActionResult> GenerateAiSuggestions([FromBody] AiSuggestionRequest request)
{
    var suggestions = await _aiService.GenerateSuggestionsAsync(request);
    var apiVersion = _versionContext.GetCurrentVersion();
    
    var response = MapSuggestionsToResponse(suggestions, apiVersion);
    return Ok(response);
}

private object MapSuggestionsToResponse(List<AiSuggestion> suggestions, ApiVersion version)
{
    if (version.Major == 1)
    {
        if (version < new ApiVersion(1, 2, 0))
        {
            // v1.0.x and v1.1.x - Basic suggestion format
            return new
            {
                suggestions = suggestions.Select(s => new
                {
                    recipeId = s.RecipeId,
                    recipeName = s.RecipeName,
                    description = s.Description,
                    prepTime = s.PrepTimeMinutes,
                    servings = s.Servings,
                    familyFit = s.FamilyFitScore
                })
            };
        }
        else if (version < new ApiVersion(1, 3, 0))
        {
            // v1.2.x - Added confidence and reasoning
            return new
            {
                suggestions = suggestions.Select(s => new
                {
                    recipeId = s.RecipeId,
                    recipeName = s.RecipeName,
                    description = s.Description,
                    prepTime = s.PrepTimeMinutes,
                    servings = s.Servings,
                    familyFit = s.FamilyFitScore,
                    // Added in v1.2.0
                    confidence = s.ConfidenceScore,
                    reasoning = s.ReasoningText
                })
            };
        }
        else
        {
            // v1.3.x - Added personalization metadata
            return new
            {
                suggestions = suggestions.Select(s => new
                {
                    recipeId = s.RecipeId,
                    recipeName = s.RecipeName,
                    description = s.Description,
                    prepTime = s.PrepTimeMinutes,
                    servings = s.Servings,
                    familyFit = s.FamilyFitScore,
                    confidence = s.ConfidenceScore,
                    reasoning = s.ReasoningText,
                    // Added in v1.3.0
                    personalization = new
                    {
                        matchedPreferences = s.MatchedPreferences,
                        avoidedRestrictions = s.AvoidedRestrictions,
                        familyMemberScores = s.FamilyMemberScores
                    }
                }),
                // Added in v1.3.0
                metadata = new
                {
                    generationTime = suggestions.First().GenerationTime,
                    modelVersion = "gemini-1.5-pro",
                    personaVersion = "v2.1"
                }
            };
        }
    }
    
    throw new UnsupportedApiVersionException($"Version {version} not supported for AI suggestions");
}
```

---

## Deprecation and Migration Strategy

### Deprecation Timeline
```yaml
Deprecation Process:
  Announcement: 6 months before deprecation
  Deprecation: Version marked as deprecated
  Warning Period: 6 months with warnings
  Sunset: Version no longer supported
  
Example Timeline for v1.0.0:
  2024-03-01: Deprecation announced
  2024-06-01: Version marked deprecated (warnings added)
  2024-12-01: Version sunset (no longer supported)
```

### Migration Notifications
```csharp
public class DeprecationNotificationService
{
    public async Task NotifyClientsOfDeprecation(string version, DateTime sunsetDate)
    {
        var clients = await GetActiveClientsUsingVersion(version);
        
        foreach (var client in clients)
        {
            await SendDeprecationNotification(client, new DeprecationNotice
            {
                DeprecatedVersion = version,
                RecommendedVersion = GetRecommendedMigrationVersion(version),
                SunsetDate = sunsetDate,
                MigrationGuideUrl = GetMigrationGuideUrl(version),
                BreakingChanges = GetBreakingChanges(version),
                ContactInfo = "api-support@mealprep.com"
            });
        }
    }

    private async Task SendDeprecationNotification(ApiClient client, DeprecationNotice notice)
    {
        var emailContent = $@"
        Important: API Version {notice.DeprecatedVersion} Deprecation Notice
        
        Dear {client.OrganizationName},
        
        We're writing to inform you that API version {notice.DeprecatedVersion} 
        will be discontinued on {notice.SunsetDate:yyyy-MM-dd}.
        
        Recommended Action:
        Please migrate to version {notice.RecommendedVersion} by the sunset date.
        
        Migration Resources:
        - Migration Guide: {notice.MigrationGuideUrl}
        - Breaking Changes: {string.Join(", ", notice.BreakingChanges)}
        - Support Contact: {notice.ContactInfo}
        
        We're here to help with your migration. Please don't hesitate to reach out.
        
        Best regards,
        MealPrep API Team
        ";

        await _emailService.SendEmailAsync(client.ContactEmail, 
            $"API Version {notice.DeprecatedVersion} Deprecation Notice", 
            emailContent);
    }
}
```

### Migration Guides
```markdown
# Migration Guide: v1.0 to v1.3

## Overview
This guide helps you migrate from API v1.0 to v1.3, including all breaking changes and new features.

## Breaking Changes

### 1. Authentication Changes
**v1.0 (Deprecated):**
```http
GET /api/recipes
Authorization: APIKey your-api-key-here
```

**v1.3 (Current):**
```http
GET /api/recipes
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
API-Version: v1.3.0
```

**Migration Steps:**
1. Obtain JWT tokens via `/auth/login` endpoint
2. Replace `APIKey` header with `Bearer` token
3. Add `API-Version` header to all requests

### 2. Recipe Response Format Changes
**v1.0 Response:**
```json
{
  "id": 123,
  "name": "Chicken Parmesan",
  "description": "Classic Italian dish",
  "cookTime": 45,
  "ingredients": ["chicken", "cheese", "sauce"]
}
```

**v1.3 Response:**
```json
{
  "id": 123,
  "name": "Chicken Parmesan",
  "description": "Classic Italian dish",
  "prepTimeMinutes": 15,
  "cookTimeMinutes": 30,
  "servings": 4,
  "ingredients": [
    {
      "id": 1,
      "name": "chicken breast",
      "quantity": 1,
      "unit": "lb"
    }
  ],
  "estimatedCost": 18.50,
  "nutritionInfo": {
    "calories": 520,
    "protein": 45,
    "carbs": 25,
    "fat": 22
  },
  "familyFitScore": 8.5
}
```

**Migration Steps:**
1. Update response parsing to handle new field names
2. Handle structured ingredient objects instead of strings
3. Utilize new fields like `estimatedCost` and `familyFitScore`

## New Features in v1.3

### 1. Enhanced AI Suggestions
- Family fit scoring based on member preferences
- Confidence scores for suggestions
- Detailed reasoning for recommendations

### 2. Nutrition Information
- Comprehensive nutrition data for all recipes
- Dietary restriction compliance indicators
- Calorie and macro tracking

### 3. Cost Estimation
- Real-time ingredient cost calculation
- Budget-based meal planning
- Cost per serving breakdown

## Code Examples

### JavaScript/TypeScript Migration
```typescript
// v1.0 Implementation (Deprecated)
class MealPrepApiV1 {
    private apiKey: string;
    
    constructor(apiKey: string) {
        this.apiKey = apiKey;
    }
    
    async getRecipes(): Promise<any[]> {
        const response = await fetch('/api/recipes', {
            headers: {
                'Authorization': `APIKey ${this.apiKey}`
            }
        });
        return response.json();
    }
}

// v1.3 Implementation (Current)
class MealPrepApiV1_3 {
    private accessToken: string;
    
    constructor(accessToken: string) {
        this.accessToken = accessToken;
    }
    
    async getRecipes(): Promise<Recipe[]> {
        const response = await fetch('/api/recipes', {
            headers: {
                'Authorization': `Bearer ${this.accessToken}`,
                'API-Version': 'v1.3.0',
                'Content-Type': 'application/json'
            }
        });
        
        if (!response.ok) {
            throw new Error(`API error: ${response.status}`);
        }
        
        return response.json();
    }
}
```

## Testing Your Migration

### 1. Parallel Testing
Run both v1.0 and v1.3 implementations side by side to compare results.

### 2. Response Validation
Validate that your application correctly handles the new response format.

### 3. Error Handling
Test error scenarios with the new authentication and response formats.

## Support and Timeline

- **Migration Deadline**: December 1, 2024
- **Support Contact**: api-support@mealprep.com
- **Documentation**: https://docs.mealprep.com/api/v1.3
- **Migration Tools**: Available at https://migrate.mealprep.com
```

---

## Version Documentation Structure

### API Documentation Organization
```
docs/api/
??? v1.0/                    # Deprecated - sunset Dec 2024
?   ??? authentication.md
?   ??? endpoints/
?   ??? migration-to-v1.3.md
??? v1.1/                    # Supported
?   ??? authentication.md
?   ??? endpoints/
?   ??? changelog.md
??? v1.2/                    # Supported
?   ??? authentication.md
?   ??? endpoints/
?   ??? changelog.md
??? v1.3/                    # Current
?   ??? authentication.md
?   ??? endpoints/
?   ??? ai-suggestions.md
?   ??? changelog.md
??? latest/                  # Symlink to current version
    ??? authentication.md
    ??? endpoints/
    ??? getting-started.md
```

### Version-Specific Changelogs
```yaml
# docs/api/v1.3/changelog.md
Version 1.3.0 (2024-09-01):
  Added:
    - Family fit scoring for AI suggestions
    - Nutrition information in recipe responses
    - Cost estimation for recipes and meal plans
    - Enhanced personalization metadata
    
  Changed:
    - Improved AI suggestion confidence scoring
    - Enhanced error messages with more detail
    
  Deprecated:
    - None
    
  Security:
    - Enhanced JWT token validation
    - Improved rate limiting for AI endpoints

Version 1.2.0 (2024-06-01):
  Added:
    - AI suggestion confidence scores
    - Recipe difficulty ratings
    - Batch recipe operations
    
  Changed:
    - Improved search performance
    - Enhanced ingredient matching
    
  Deprecated:
    - Legacy bulk import format (use /recipes/batch)
```

---

## Client Libraries and SDKs

### Version-Aware SDK Design
```typescript
// TypeScript SDK with version support
export class MealPrepApiClient {
    private baseUrl: string;
    private version: string;
    private accessToken: string;

    constructor(config: {
        baseUrl: string;
        accessToken: string;
        version?: string;
    }) {
        this.baseUrl = config.baseUrl;
        this.accessToken = config.accessToken;
        this.version = config.version || 'v1.3.0';
    }

    // Version-aware request method
    private async request<T>(
        endpoint: string, 
        options: RequestInit = {}
    ): Promise<T> {
        const url = `${this.baseUrl}/api${endpoint}`;
        
        const response = await fetch(url, {
            ...options,
            headers: {
                'Authorization': `Bearer ${this.accessToken}`,
                'API-Version': this.version,
                'Content-Type': 'application/json',
                ...options.headers
            }
        });

        // Check for version-related warnings
        const deprecationWarning = response.headers.get('Warning');
        if (deprecationWarning) {
            console.warn('API Version Warning:', deprecationWarning);
        }

        if (!response.ok) {
            throw new ApiError(await response.json(), response.status);
        }

        return response.json();
    }

    // Version-specific methods
    async getRecipes(): Promise<Recipe[]> {
        return this.request<Recipe[]>('/recipes');
    }

    // Method with version-specific behavior
    async generateAiSuggestions(
        request: AiSuggestionRequest
    ): Promise<AiSuggestionResponse> {
        const response = await this.request<any>('/ai/suggestions', {
            method: 'POST',
            body: JSON.stringify(request)
        });

        // Handle version-specific response formats
        if (this.version.startsWith('v1.0') || this.version.startsWith('v1.1')) {
            // Transform legacy format to current format
            return this.transformLegacyAiResponse(response);
        }

        return response;
    }

    private transformLegacyAiResponse(legacyResponse: any): AiSuggestionResponse {
        // Transform old format to new format for backward compatibility
        return {
            suggestions: legacyResponse.suggestions.map((s: any) => ({
                ...s,
                confidence: s.confidence || 0.8, // Default for legacy responses
                reasoning: s.reasoning || 'Generated by AI',
                personalization: {
                    matchedPreferences: [],
                    avoidedRestrictions: [],
                    familyMemberScores: {}
                }
            })),
            metadata: {
                generationTime: new Date().toISOString(),
                modelVersion: 'legacy',
                personaVersion: 'v1.0'
            }
        };
    }
}
```

---

## Monitoring and Analytics

### Version Usage Tracking
```csharp
public class ApiVersionAnalytics
{
    private readonly IMetricsCollector _metrics;
    
    public async Task TrackVersionUsage(HttpContext context, ApiVersion version)
    {
        var endpoint = GetNormalizedEndpoint(context.Request.Path);
        var userAgent = context.Request.Headers["User-Agent"].ToString();
        var clientId = ExtractClientId(context);

        _metrics.Counter("api_requests_total")
            .WithTag("version", version.ToString())
            .WithTag("endpoint", endpoint)
            .WithTag("major_version", version.Major.ToString())
            .WithTag("client_id", clientId)
            .Increment();

        // Track deprecated version usage
        if (IsDeprecatedVersion(version))
        {
            _metrics.Counter("api_deprecated_version_usage")
                .WithTag("version", version.ToString())
                .WithTag("endpoint", endpoint)
                .WithTag("client_id", clientId)
                .Increment();
        }

        // Store detailed usage for analytics
        await _analytics.RecordApiUsage(new ApiUsageRecord
        {
            Version = version.ToString(),
            Endpoint = endpoint,
            ClientId = clientId,
            UserAgent = userAgent,
            Timestamp = DateTime.UtcNow,
            IsDeprecated = IsDeprecatedVersion(version)
        });
    }

    public async Task<VersionUsageReport> GenerateUsageReport(DateTime fromDate, DateTime toDate)
    {
        var usage = await _analytics.GetVersionUsage(fromDate, toDate);
        
        return new VersionUsageReport
        {
            TotalRequests = usage.Sum(u => u.RequestCount),
            VersionBreakdown = usage.GroupBy(u => u.Version)
                .ToDictionary(g => g.Key, g => g.Sum(u => u.RequestCount)),
            DeprecatedVersionUsage = usage.Where(u => u.IsDeprecated)
                .Sum(u => u.RequestCount),
            TopClients = usage.GroupBy(u => u.ClientId)
                .OrderByDescending(g => g.Sum(u => u.RequestCount))
                .Take(10)
                .ToDictionary(g => g.Key, g => g.Sum(u => u.RequestCount)),
            RecommendedActions = GenerateRecommendations(usage)
        };
    }
}
```

---

## Best Practices Summary

### For API Providers
1. **Plan for Evolution**: Design APIs with future changes in mind
2. **Clear Communication**: Provide detailed migration guides and timelines
3. **Gradual Deprecation**: Allow sufficient time for client migration
4. **Monitor Usage**: Track version adoption and deprecated version usage
5. **Automated Testing**: Test all supported versions continuously

### For API Consumers
1. **Version Pinning**: Specify exact versions in production
2. **Stay Updated**: Regularly review deprecation notices
3. **Test Early**: Test new versions in staging environments
4. **Handle Gracefully**: Implement proper error handling for version issues
5. **Monitor Headers**: Watch for deprecation warnings in responses

### Security Considerations
1. **Version-Specific Security**: Apply security patches across all supported versions
2. **Authentication Evolution**: Plan authentication method migrations carefully
3. **Data Protection**: Ensure version changes don't compromise data security
4. **Audit Logging**: Log version usage for security monitoring

---

*Last Updated: December 2024*  
*API versioning strategy continuously updated with industry best practices and platform evolution*