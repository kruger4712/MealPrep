# Google Gemini AI Integration Guide

## Overview
Comprehensive guide for integrating Google Gemini AI into the MealPrep application for intelligent meal suggestions, recipe personalization, and menu planning optimization.

## Architecture Overview

### AI Service Architecture
```
???????????????????    ???????????????????    ???????????????????
?   MealPrep API  ??????  AI Service     ?????? Google Cloud    ?
?                 ?    ?                 ?    ? Vertex AI       ?
? • Controllers   ?    ? • Prompt Eng.   ?    ?                 ?
? • Family Data   ?    ? • Response      ?    ? • Gemini Pro    ?
? • Preferences   ?    ?   Processing    ?    ? • Rate Limiting ?
?                 ?    ? • Caching       ?    ? • Cost Tracking ?
???????????????????    ???????????????????    ???????????????????
        ?                       ?                       ?
        ?                       ?                       ?
???????????????????    ???????????????????    ???????????????????
?   Database      ?    ?  Cache Layer    ?    ? Fallback        ?
?                 ?    ?                 ?    ? Strategies      ?
? • User Data     ?    ? • Prompt Cache  ?    ?                 ?
? • Family        ?    ? • Response      ?    ? • Recipe DB     ?
?   Personas      ?    ?   Cache         ?    ? • Rule-based    ?
? • History       ?    ? • Cost Tracking ?    ?   Suggestions   ?
???????????????????    ???????????????????    ???????????????????
```

## Google Cloud Setup

### Project Configuration
```bash
# 1. Create Google Cloud Project
gcloud projects create mealprep-ai-project --name="MealPrep AI"

# 2. Enable required APIs
gcloud services enable aiplatform.googleapis.com
gcloud services enable cloudresourcemanager.googleapis.com
gcloud services enable iam.googleapis.com

# 3. Set project as default
gcloud config set project mealprep-ai-project
```

### Service Account Setup
```bash
# Create service account
gcloud iam service-accounts create mealprep-ai-service \
    --display-name="MealPrep AI Service Account" \
    --description="Service account for MealPrep AI operations"

# Grant necessary permissions
gcloud projects add-iam-policy-binding mealprep-ai-project \
    --member="serviceAccount:mealprep-ai-service@mealprep-ai-project.iam.gserviceaccount.com" \
    --role="roles/aiplatform.user"

# Create and download key
gcloud iam service-accounts keys create ~/mealprep-ai-key.json \
    --iam-account=mealprep-ai-service@mealprep-ai-project.iam.gserviceaccount.com
```

### Authentication Configuration
```csharp
// appsettings.json
{
  "GoogleCloud": {
    "ProjectId": "mealprep-ai-project",
    "ServiceAccountKeyPath": "/path/to/mealprep-ai-key.json",
    "Location": "us-central1",
    "ModelConfig": {
      "ModelName": "gemini-pro",
      "Temperature": 0.7,
      "TopP": 0.8,
      "TopK": 40,
      "MaxOutputTokens": 2048
    }
  }
}
```

## Core AI Service Implementation

### GeminiAiService Class
```csharp
using Google.Cloud.AIPlatform.V1;
using Google.Protobuf.WellKnownTypes;
using Microsoft.Extensions.Options;

public class GeminiAiService : IGeminiAiService
{
    private readonly PredictionServiceClient _predictionClient;
    private readonly GoogleCloudConfig _config;
    private readonly ILogger<GeminiAiService> _logger;
    private readonly IMemoryCache _cache;

    public GeminiAiService(
        IOptions<GoogleCloudConfig> config,
        ILogger<GeminiAiService> logger,
        IMemoryCache cache)
    {
        _config = config.Value;
        _logger = logger;
        _cache = cache;
        
        var clientBuilder = new PredictionServiceClientBuilder
        {
            CredentialsPath = _config.ServiceAccountKeyPath
        };
        _predictionClient = clientBuilder.Build();
    }

    public async Task<List<MealSuggestion>> GenerateMealSuggestionsAsync(
        MealSuggestionRequest request)
    {
        try
        {
            // Build family persona
            var familyPersona = await BuildFamilyPersonaAsync(request.FamilyMembers);
            
            // Generate AI prompt
            var prompt = _promptBuilder.CreateMealSuggestionPrompt(familyPersona, request);
            
            // Check cache first
            var cacheKey = GenerateCacheKey(prompt);
            if (_cache.TryGetValue(cacheKey, out List<MealSuggestion> cachedResult))
            {
                _logger.LogInformation("Returning cached AI suggestions for key: {CacheKey}", cacheKey);
                return cachedResult;
            }

            // Call Gemini AI
            var response = await CallGeminiApiAsync(prompt);
            
            // Process and validate response
            var suggestions = await ProcessAiResponseAsync(response, request);
            
            // Cache the results
            _cache.Set(cacheKey, suggestions, TimeSpan.FromHours(1));
            
            return suggestions;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to generate AI meal suggestions");
            throw new AiServiceException("Failed to generate meal suggestions", ex);
        }
    }

    private async Task<string> CallGeminiApiAsync(string prompt)
    {
        var endpointName = EndpointName.FromProjectLocationEndpoint(
            _config.ProjectId, 
            _config.Location, 
            "gemini-pro");

        var instances = new[]
        {
            Value.ForStruct(new Struct
            {
                Fields =
                {
                    ["prompt"] = Value.ForString(prompt),
                    ["temperature"] = Value.ForNumber(_config.ModelConfig.Temperature),
                    ["topP"] = Value.ForNumber(_config.ModelConfig.TopP),
                    ["topK"] = Value.ForNumber(_config.ModelConfig.TopK),
                    ["maxOutputTokens"] = Value.ForNumber(_config.ModelConfig.MaxOutputTokens)
                }
            })
        };

        var request = new PredictRequest
        {
            Endpoint = endpointName.ToString(),
            Instances = { instances }
        };

        var response = await _predictionClient.PredictAsync(request);
        
        // Extract response content
        var prediction = response.Predictions.FirstOrDefault();
        if (prediction?.StructValue?.Fields?.TryGetValue("content", out var content) == true)
        {
            return content.StringValue;
        }

        throw new AiServiceException("Invalid response format from Gemini AI");
    }
}
```

### Prompt Engineering
```csharp
public class MealSuggestionPromptBuilder : IPromptBuilder
{
    public string CreateMealSuggestionPrompt(FamilyPersona persona, MealSuggestionRequest request)
    {
        var prompt = $@"
You are an expert family meal planning assistant. Generate 3 personalized meal suggestions based on the following family profile and constraints.

FAMILY PROFILE:
{BuildFamilyProfileSection(persona)}

MEAL REQUEST:
- Meal Type: {request.MealType}
- Target Date: {request.TargetDate:yyyy-MM-dd}
- Budget Limit: ${request.Budget:F2}
- Max Prep Time: {request.MaxPrepTime} minutes
- Max Cook Time: {request.MaxCookTime} minutes

CONSTRAINTS:
{BuildConstraintsSection(request)}

RECENT MEALS (avoid duplicating):
{string.Join(", ", request.RecentMeals)}

AVAILABLE INGREDIENTS:
{BuildAvailableIngredientsSection(request.AvailableIngredients)}

Please provide exactly 3 meal suggestions in the following JSON format:
{{
  ""suggestions"": [
    {{
      ""name"": ""Recipe Name"",
      ""description"": ""Brief appetizing description"",
      ""prepTime"": number,
      ""cookTime"": number,
      ""servings"": number,
      ""estimatedCost"": number,
      ""cuisine"": ""Cuisine Type"",
      ""difficulty"": ""Easy|Medium|Hard"",
      ""familyFitScore"": number (1-10),
      ""reasoning"": ""Why this meal fits the family well"",
      ""ingredients"": [
        {{
          ""name"": ""Ingredient name"",
          ""quantity"": number,
          ""unit"": ""unit"",
          ""isAvailable"": boolean,
          ""estimatedCost"": number
        }}
      ],
      ""instructions"": [
        ""Step 1 instruction"",
        ""Step 2 instruction""
      ],
      ""nutritionalInfo"": {{
        ""calories"": number,
        ""protein"": number,
        ""carbs"": number,
        ""fat"": number,
        ""fiber"": number,
        ""sodium"": number
      }},
      ""tags"": [""tag1"", ""tag2""],
      ""allergenWarnings"": [""allergen1""],
      ""personalizations"": {{
        ""{persona.Members[0].Name}"": ""Personalization note"",
        ""{persona.Members[1].Name}"": ""Personalization note""
      }}
    }}
  ]
}}

IMPORTANT GUIDELINES:
1. Ensure all suggestions fit within the budget and time constraints
2. Consider each family member's dietary restrictions and preferences
3. Avoid ingredients that family members dislike or are allergic to
4. Prioritize ingredients that are already available
5. Provide realistic cost estimates
6. Include family fit reasoning that explains why each meal works for this specific family
7. Make suggestions diverse in cuisine and cooking methods
8. Ensure nutritional information is accurate and realistic
9. Include specific personalization notes for each family member when relevant

Respond with ONLY the JSON, no additional text.
";

        return prompt.Trim();
    }

    private string BuildFamilyProfileSection(FamilyPersona persona)
    {
        var profile = new StringBuilder();
        profile.AppendLine($"Family Size: {persona.Members.Count} members");
        profile.AppendLine($"Ages: {string.Join(", ", persona.Members.Select(m => m.Age))}");
        
        foreach (var member in persona.Members)
        {
            profile.AppendLine($"\n{member.Name} ({member.Age} years old):");
            profile.AppendLine($"  - Dietary Restrictions: {string.Join(", ", member.DietaryRestrictions)}");
            if (member.Allergies.Any())
                profile.AppendLine($"  - Allergies: {string.Join(", ", member.Allergies)}");
            if (member.LikedIngredients.Any())
                profile.AppendLine($"  - Loves: {string.Join(", ", member.LikedIngredients.Take(5))}");
            if (member.DislikedIngredients.Any())
                profile.AppendLine($"  - Dislikes: {string.Join(", ", member.DislikedIngredients.Take(5))}");
            profile.AppendLine($"  - Spice Tolerance: {member.SpiceToleranceLevel}/10");
            if (member.CuisinePreferences.Any())
                profile.AppendLine($"  - Preferred Cuisines: {string.Join(", ", member.CuisinePreferences)}");
        }

        return profile.ToString();
    }
}
```

### Response Processing
```csharp
public class AiResponseProcessor : IAiResponseProcessor
{
    private readonly ILogger<AiResponseProcessor> _logger;
    private readonly IIngredientService _ingredientService;

    public async Task<List<MealSuggestion>> ProcessAiResponseAsync(
        string aiResponse, 
        MealSuggestionRequest originalRequest)
    {
        try
        {
            // Parse JSON response
            var responseData = JsonSerializer.Deserialize<AiSuggestionResponse>(aiResponse);
            
            if (responseData?.Suggestions == null || !responseData.Suggestions.Any())
            {
                throw new AiResponseException("No suggestions found in AI response");
            }

            var suggestions = new List<MealSuggestion>();

            foreach (var suggestion in responseData.Suggestions)
            {
                // Validate suggestion
                var validationResult = await ValidateSuggestionAsync(suggestion, originalRequest);
                if (!validationResult.IsValid)
                {
                    _logger.LogWarning("Invalid AI suggestion: {Errors}", 
                        string.Join(", ", validationResult.Errors));
                    continue;
                }

                // Enrich with additional data
                var enrichedSuggestion = await EnrichSuggestionAsync(suggestion);
                
                suggestions.Add(enrichedSuggestion);
            }

            if (!suggestions.Any())
            {
                throw new AiResponseException("No valid suggestions after processing");
            }

            // Sort by family fit score
            return suggestions.OrderByDescending(s => s.FamilyFitScore).ToList();
        }
        catch (JsonException ex)
        {
            _logger.LogError(ex, "Failed to parse AI response JSON: {Response}", aiResponse);
            throw new AiResponseException("Invalid JSON format in AI response", ex);
        }
    }

    private async Task<ValidationResult> ValidateSuggestionAsync(
        AiSuggestion suggestion, 
        MealSuggestionRequest request)
    {
        var errors = new List<string>();

        // Validate budget constraint
        if (suggestion.EstimatedCost > request.Budget)
        {
            errors.Add($"Cost ${suggestion.EstimatedCost:F2} exceeds budget ${request.Budget:F2}");
        }

        // Validate time constraints
        if (suggestion.PrepTime > request.MaxPrepTime)
        {
            errors.Add($"Prep time {suggestion.PrepTime}min exceeds limit {request.MaxPrepTime}min");
        }

        // Validate family fit score
        if (suggestion.FamilyFitScore < 1 || suggestion.FamilyFitScore > 10)
        {
            errors.Add($"Invalid family fit score: {suggestion.FamilyFitScore}");
        }

        // Validate ingredients exist
        foreach (var ingredient in suggestion.Ingredients)
        {
            var exists = await _ingredientService.ExistsAsync(ingredient.Name);
            if (!exists)
            {
                _logger.LogWarning("Unknown ingredient in AI suggestion: {Ingredient}", ingredient.Name);
                // Don't fail validation, but log for improvement
            }
        }

        return new ValidationResult
        {
            IsValid = !errors.Any(),
            Errors = errors
        };
    }

    private async Task<MealSuggestion> EnrichSuggestionAsync(AiSuggestion suggestion)
    {
        // Add recipe difficulty scoring
        var difficulty = CalculateDifficultyScore(suggestion);
        
        // Enhance nutritional information
        var nutrition = await EnhanceNutritionalInfoAsync(suggestion.Ingredients);
        
        // Add cooking tips based on techniques used
        var tips = GenerateCookingTips(suggestion);

        return new MealSuggestion
        {
            Id = Guid.NewGuid().ToString(),
            Name = suggestion.Name,
            Description = suggestion.Description,
            PrepTime = suggestion.PrepTime,
            CookTime = suggestion.CookTime,
            Servings = suggestion.Servings,
            EstimatedCost = suggestion.EstimatedCost,
            Cuisine = suggestion.Cuisine,
            Difficulty = suggestion.Difficulty,
            FamilyFitScore = suggestion.FamilyFitScore,
            Reasoning = suggestion.Reasoning,
            Ingredients = suggestion.Ingredients,
            Instructions = suggestion.Instructions,
            NutritionalInfo = nutrition,
            Tags = suggestion.Tags,
            AllergenWarnings = suggestion.AllergenWarnings,
            Personalizations = suggestion.Personalizations,
            CookingTips = tips,
            DifficultyScore = difficulty,
            Source = "AI Generated",
            GeneratedAt = DateTime.UtcNow,
            Confidence = CalculateConfidenceScore(suggestion)
        };
    }
}
```

## Caching Strategy

### Prompt Caching
```csharp
public class PromptCacheService : IPromptCacheService
{
    private readonly IMemoryCache _memoryCache;
    private readonly IDistributedCache _distributedCache;
    private readonly ILogger<PromptCacheService> _logger;

    public async Task<List<MealSuggestion>> GetCachedSuggestionsAsync(string promptHash)
    {
        // Try memory cache first (fastest)
        if (_memoryCache.TryGetValue($"ai_suggestions_{promptHash}", out List<MealSuggestion> memoryResult))
        {
            _logger.LogDebug("Cache hit from memory for prompt hash: {Hash}", promptHash);
            return memoryResult;
        }

        // Try distributed cache (Redis)
        var distributedResult = await _distributedCache.GetStringAsync($"ai_suggestions_{promptHash}");
        if (!string.IsNullOrEmpty(distributedResult))
        {
            _logger.LogDebug("Cache hit from distributed cache for prompt hash: {Hash}", promptHash);
            var suggestions = JsonSerializer.Deserialize<List<MealSuggestion>>(distributedResult);
            
            // Store in memory cache for faster access
            _memoryCache.Set($"ai_suggestions_{promptHash}", suggestions, TimeSpan.FromMinutes(30));
            
            return suggestions;
        }

        return null;
    }

    public async Task CacheSuggestionsAsync(string promptHash, List<MealSuggestion> suggestions)
    {
        var cacheOptions = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1),
            SlidingExpiration = TimeSpan.FromMinutes(30),
            Priority = CacheItemPriority.Normal
        };

        // Store in memory cache
        _memoryCache.Set($"ai_suggestions_{promptHash}", suggestions, cacheOptions);

        // Store in distributed cache
        var serialized = JsonSerializer.Serialize(suggestions);
        var distributedOptions = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(4)
        };
        
        await _distributedCache.SetStringAsync($"ai_suggestions_{promptHash}", serialized, distributedOptions);
        
        _logger.LogDebug("Cached suggestions for prompt hash: {Hash}", promptHash);
    }

    public string GeneratePromptHash(string prompt, Dictionary<string, object> parameters = null)
    {
        var content = prompt;
        if (parameters != null)
        {
            content += JsonSerializer.Serialize(parameters, new JsonSerializerOptions { PropertyNamingPolicy = JsonNamingPolicy.CamelCase });
        }

        using var sha256 = SHA256.Create();
        var hashBytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(content));
        return Convert.ToBase64String(hashBytes);
    }
}
```

## Error Handling and Fallbacks

### Fallback Strategy Implementation
```csharp
public class FallbackSuggestionService : IFallbackSuggestionService
{
    private readonly IRecipeRepository _recipeRepository;
    private readonly IFamilyService _familyService;
    private readonly ILogger<FallbackSuggestionService> _logger;

    public async Task<List<MealSuggestion>> GenerateFallbackSuggestionsAsync(
        MealSuggestionRequest request)
    {
        _logger.LogInformation("Generating fallback suggestions for request: {RequestId}", request.Id);

        try
        {
            // Get family preferences
            var family = await _familyService.GetFamilyAsync(request.FamilyMembers);
            
            // Build preference-based query
            var query = BuildPreferenceQuery(family, request);
            
            // Get matching recipes from database
            var matchingRecipes = await _recipeRepository.SearchAsync(query);
            
            // Score and rank recipes
            var scoredRecipes = ScoreRecipesForFamily(matchingRecipes, family, request);
            
            // Convert to meal suggestions
            var suggestions = scoredRecipes
                .Take(3)
                .Select(r => ConvertToMealSuggestion(r, family))
                .ToList();

            _logger.LogInformation("Generated {Count} fallback suggestions", suggestions.Count);
            return suggestions;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to generate fallback suggestions");
            
            // Last resort: return popular recipes
            return await GetPopularRecipesAsync(request);
        }
    }

    private List<ScoredRecipe> ScoreRecipesForFamily(
        IEnumerable<Recipe> recipes, 
        Family family, 
        MealSuggestionRequest request)
    {
        return recipes.Select(recipe => new ScoredRecipe
        {
            Recipe = recipe,
            Score = CalculateFamilyFitScore(recipe, family, request)
        })
        .Where(sr => sr.Score >= 5) // Minimum acceptable score
        .OrderByDescending(sr => sr.Score)
        .ToList();
    }

    private decimal CalculateFamilyFitScore(Recipe recipe, Family family, MealSuggestionRequest request)
    {
        decimal score = 5.0m; // Base score

        // Budget compatibility (20% weight)
        if (recipe.EstimatedCost <= request.Budget)
            score += 1.0m;
        else if (recipe.EstimatedCost > request.Budget * 1.2m)
            score -= 2.0m;

        // Time compatibility (15% weight)
        var totalTime = recipe.PrepTime + recipe.CookTime;
        if (totalTime <= request.MaxPrepTime + request.MaxCookTime)
            score += 0.8m;

        // Dietary restrictions compatibility (25% weight)
        foreach (var member in family.Members)
        {
            if (RecipeMatchesDietaryRestrictions(recipe, member.DietaryRestrictions))
                score += 0.5m;
            if (recipe.ContainsAllergens(member.Allergies))
                score -= 3.0m; // Heavy penalty for allergens
        }

        // Ingredient preferences (25% weight)
        var preferenceScore = CalculateIngredientPreferenceScore(recipe, family.Members);
        score += preferenceScore;

        // Cuisine preferences (15% weight)
        if (family.Members.Any(m => m.CuisinePreferences.Contains(recipe.Cuisine, StringComparer.OrdinalIgnoreCase)))
            score += 0.8m;

        return Math.Max(1.0m, Math.Min(10.0m, score));
    }
}
```

## Cost Monitoring and Optimization

### AI Usage Tracking
```csharp
public class AiUsageTracker : IAiUsageTracker
{
    private readonly IAiUsageRepository _usageRepository;
    private readonly ILogger<AiUsageTracker> _logger;

    public async Task TrackUsageAsync(AiUsageRecord record)
    {
        try
        {
            record.Id = Guid.NewGuid();
            record.Timestamp = DateTime.UtcNow;
            
            await _usageRepository.AddAsync(record);
            
            // Check for usage limits
            await CheckUsageLimitsAsync(record.UserId);
            
            _logger.LogInformation("Tracked AI usage: {Tokens} tokens, ${Cost:F4} cost for user {UserId}",
                record.TokensUsed, record.EstimatedCost, record.UserId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to track AI usage for user {UserId}", record.UserId);
        }
    }

    public async Task<AiUsageStats> GetUsageStatsAsync(string userId, DateTime startDate, DateTime endDate)
    {
        var records = await _usageRepository.GetByUserAndDateRangeAsync(userId, startDate, endDate);
        
        return new AiUsageStats
        {
            UserId = userId,
            StartDate = startDate,
            EndDate = endDate,
            TotalRequests = records.Count,
            TotalTokensUsed = records.Sum(r => r.TokensUsed),
            TotalCost = records.Sum(r => r.EstimatedCost),
            AverageResponseTime = records.Average(r => r.ResponseTimeMs),
            RequestsByType = records.GroupBy(r => r.RequestType)
                .ToDictionary(g => g.Key, g => g.Count()),
            DailyCosts = records.GroupBy(r => r.Timestamp.Date)
                .ToDictionary(g => g.Key, g => g.Sum(r => r.EstimatedCost))
        };
    }

    private async Task CheckUsageLimitsAsync(string userId)
    {
        var monthlyUsage = await GetUsageStatsAsync(userId, 
            DateTime.UtcNow.AddDays(-30), DateTime.UtcNow);

        var userTier = await GetUserTierAsync(userId);
        var limits = GetTierLimits(userTier);

        if (monthlyUsage.TotalRequests >= limits.MonthlyRequestLimit)
        {
            throw new UsageLimitException($"Monthly request limit exceeded ({limits.MonthlyRequestLimit})");
        }

        if (monthlyUsage.TotalCost >= limits.MonthlyCostLimit)
        {
            throw new UsageLimitException($"Monthly cost limit exceeded (${limits.MonthlyCostLimit:F2})");
        }
    }
}
```

## Performance Optimization

### Batch Processing
```csharp
public class BatchAiProcessor : IBatchAiProcessor
{
    private readonly IGeminiAiService _aiService;
    private readonly ILogger<BatchAiProcessor> _logger;

    public async Task<List<MealSuggestion>> ProcessBatchSuggestionsAsync(
        List<MealSuggestionRequest> requests)
    {
        var allSuggestions = new List<MealSuggestion>();
        var batchSize = 5; // Optimal batch size for Gemini API

        for (int i = 0; i < requests.Count; i += batchSize)
        {
            var batch = requests.Skip(i).Take(batchSize).ToList();
            var batchTasks = batch.Select(async request =>
            {
                try
                {
                    return await _aiService.GenerateMealSuggestionsAsync(request);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Failed to process batch item for request {RequestId}", request.Id);
                    return new List<MealSuggestion>();
                }
            });

            var batchResults = await Task.WhenAll(batchTasks);
            allSuggestions.AddRange(batchResults.SelectMany(r => r));

            // Add delay between batches to respect rate limits
            if (i + batchSize < requests.Count)
            {
                await Task.Delay(TimeSpan.FromMilliseconds(500));
            }
        }

        return allSuggestions;
    }
}
```

This comprehensive Gemini AI integration guide provides the foundation for intelligent meal planning features in the MealPrep application, with proper error handling, caching, and cost optimization strategies.

*This guide should be updated as Google Cloud AI Platform evolves and new features are added.*