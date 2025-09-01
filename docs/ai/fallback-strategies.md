# AI Fallback Strategies

## Overview
Comprehensive fallback strategies for the MealPrep application to ensure continuous service when AI systems are unavailable, degraded, or produce unsatisfactory results.

## Fallback Architecture

### Multi-Tier Fallback System
```
Primary AI (Gemini) ? Secondary AI ? Rule-Based ? Cached Responses ? Default Responses
        ?                  ?             ?              ?                ?
   Full AI Power    Alternative AI   Smart Rules    Previous Data    Basic Functionality
   (Best Quality)   (Good Quality)  (Fair Quality) (Cached Quality) (Minimal Quality)
```

### Fallback Decision Engine
```csharp
public class FallbackDecisionEngine : IFallbackDecisionEngine
{
    private readonly ILogger<FallbackDecisionEngine> _logger;
    private readonly IAiHealthMonitor _healthMonitor;
    private readonly IFallbackStrategyFactory _strategyFactory;
    private readonly IMetricsCollector _metrics;

    public async Task<FallbackDecision> DetermineFallbackStrategyAsync(
        FallbackContext context)
    {
        var decision = new FallbackDecision
        {
            RequestId = context.RequestId,
            DecisionTime = DateTime.UtcNow
        };

        // Assess AI service health
        var healthStatus = await _healthMonitor.GetHealthStatusAsync();
        
        // Determine appropriate fallback level
        var fallbackLevel = DetermineFallbackLevel(context, healthStatus);
        
        // Select strategy based on fallback level
        decision.SelectedStrategy = await _strategyFactory.GetStrategyAsync(fallbackLevel, context);
        decision.FallbackLevel = fallbackLevel;
        decision.Reasoning = GenerateDecisionReasoning(context, healthStatus, fallbackLevel);
        
        // Track decision metrics
        await _metrics.TrackFallbackDecisionAsync(decision);
        
        return decision;
    }

    private FallbackLevel DetermineFallbackLevel(FallbackContext context, AiHealthStatus healthStatus)
    {
        // Critical failures require immediate fallback
        if (healthStatus.IsDown || healthStatus.ErrorRate > 0.5)
        {
            return context.RequiresHighQuality ? FallbackLevel.RuleBased : FallbackLevel.Cached;
        }

        // Performance degradation
        if (healthStatus.AverageResponseTime > TimeSpan.FromSeconds(30))
        {
            return FallbackLevel.SecondaryAi;
        }

        // Quality issues
        if (healthStatus.QualityScore < 0.6)
        {
            return context.CanAcceptLowerQuality ? FallbackLevel.RuleBased : FallbackLevel.SecondaryAi;
        }

        // Timeout or specific request failures
        if (context.PrimaryFailureReason == FailureReason.Timeout)
        {
            return FallbackLevel.SecondaryAi;
        }

        if (context.PrimaryFailureReason == FailureReason.InvalidResponse)
        {
            return FallbackLevel.RuleBased;
        }

        return FallbackLevel.PrimaryAi; // No fallback needed
    }
}
```

## Fallback Strategies

### 1. Secondary AI Provider
```csharp
public class SecondaryAiFallbackStrategy : IFallbackStrategy
{
    private readonly IOpenAiService _openAiService;
    private readonly IAnthropicService _anthropicService;
    private readonly IPromptTranslator _promptTranslator;

    public async Task<FallbackResult<T>> ExecuteAsync<T>(FallbackContext context) where T : class
    {
        try
        {
            // Try OpenAI as secondary
            var openAiResult = await TryOpenAiAsync<T>(context);
            if (openAiResult.IsSuccessful)
            {
                return openAiResult;
            }

            // Try Anthropic as tertiary
            var anthropicResult = await TryAnthropicAsync<T>(context);
            if (anthropicResult.IsSuccessful)
            {
                return anthropicResult;
            }

            return FallbackResult<T>.Failed("All secondary AI providers failed");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Secondary AI fallback strategy failed");
            return FallbackResult<T>.Failed($"Secondary AI strategy error: {ex.Message}");
        }
    }

    private async Task<FallbackResult<T>> TryOpenAiAsync<T>(FallbackContext context) where T : class
    {
        try
        {
            // Translate Gemini prompt to OpenAI format
            var openAiPrompt = await _promptTranslator.TranslateToOpenAiAsync(context.OriginalPrompt);
            
            var response = await _openAiService.GenerateResponseAsync(openAiPrompt, new OpenAiConfig
            {
                Model = "gpt-4",
                Temperature = 0.7,
                MaxTokens = 2048,
                Timeout = TimeSpan.FromSeconds(30)
            });

            var processedResponse = await ProcessSecondaryResponse<T>(response, context);
            
            return FallbackResult<T>.Success(processedResponse, "OpenAI fallback successful");
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "OpenAI fallback failed for request {RequestId}", context.RequestId);
            return FallbackResult<T>.Failed($"OpenAI fallback failed: {ex.Message}");
        }
    }

    private async Task<FallbackResult<T>> TryAnthropicAsync<T>(FallbackContext context) where T : class
    {
        try
        {
            var anthropicPrompt = await _promptTranslator.TranslateToAnthropicAsync(context.OriginalPrompt);
            
            var response = await _anthropicService.GenerateResponseAsync(anthropicPrompt, new AnthropicConfig
            {
                Model = "claude-3-sonnet-20240229",
                MaxTokens = 2048,
                Temperature = 0.7
            });

            var processedResponse = await ProcessSecondaryResponse<T>(response, context);
            
            return FallbackResult<T>.Success(processedResponse, "Anthropic fallback successful");
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Anthropic fallback failed for request {RequestId}", context.RequestId);
            return FallbackResult<T>.Failed($"Anthropic fallback failed: {ex.Message}");
        }
    }
}
```

### 2. Rule-Based Fallback Strategy
```csharp
public class RuleBasedFallbackStrategy : IFallbackStrategy
{
    private readonly IRecipeRepository _recipeRepository;
    private readonly IFamilyService _familyService;
    private readonly IIngredientService _ingredientService;
    private readonly IRuleEngine _ruleEngine;

    public async Task<FallbackResult<T>> ExecuteAsync<T>(FallbackContext context) where T : class
    {
        if (context.RequestType == RequestType.MealSuggestion)
        {
            return await GenerateRuleBasedMealSuggestions<T>(context);
        }
        else if (context.RequestType == RequestType.WeeklyMenu)
        {
            return await GenerateRuleBasedWeeklyMenu<T>(context);
        }
        else if (context.RequestType == RequestType.RecipePersonalization)
        {
            return await GenerateRuleBasedPersonalization<T>(context);
        }

        return FallbackResult<T>.Failed("Unsupported request type for rule-based fallback");
    }

    private async Task<FallbackResult<T>> GenerateRuleBasedMealSuggestions<T>(FallbackContext context)
    {
        var request = context.OriginalRequest as MealSuggestionRequest;
        var family = await _familyService.GetFamilyAsync(request.FamilyMembers);
        
        // Build search criteria based on family preferences
        var searchCriteria = await BuildSearchCriteria(family, request);
        
        // Get candidate recipes from database
        var candidateRecipes = await _recipeRepository.SearchAsync(searchCriteria);
        
        // Score and rank recipes using rule engine
        var scoredRecipes = await ScoreRecipesAsync(candidateRecipes, family, request);
        
        // Select top suggestions
        var suggestions = scoredRecipes
            .Take(3)
            .Select(sr => ConvertToMealSuggestion(sr.Recipe, sr.Score, family))
            .ToList();

        var response = new MealSuggestionResponse { Suggestions = suggestions };
        
        return FallbackResult<T>.Success((T)(object)response, "Rule-based suggestions generated");
    }

    private async Task<RecipeSearchCriteria> BuildSearchCriteria(Family family, MealSuggestionRequest request)
    {
        var criteria = new RecipeSearchCriteria
        {
            MealType = request.MealType,
            MaxCost = request.Budget,
            MaxPrepTime = request.MaxPrepTime,
            MaxCookTime = request.MaxCookTime,
            ServingRange = new Range(family.Members.Count - 1, family.Members.Count + 2)
        };

        // Exclude allergens
        criteria.ExcludeAllergens = family.GetAllAllergens();
        
        // Include dietary restrictions
        criteria.DietaryRestrictions = family.GetCommonDietaryRestrictions();
        
        // Prefer liked ingredients
        var likedIngredients = family.Members
            .SelectMany(m => m.LikedIngredients)
            .GroupBy(i => i)
            .OrderByDescending(g => g.Count())
            .Take(10)
            .Select(g => g.Key)
            .ToList();
        criteria.PreferredIngredients = likedIngredients;
        
        // Avoid disliked ingredients when possible
        var dislikedIngredients = family.Members
            .SelectMany(m => m.DislikedIngredients)
            .GroupBy(i => i)
            .Where(g => g.Count() >= family.Members.Count / 2) // Majority dislikes
            .Select(g => g.Key)
            .ToList();
        criteria.AvoidIngredients = dislikedIngredients;
        
        // Cuisine preferences
        criteria.PreferredCuisines = family.GetPreferredCuisines();
        
        return criteria;
    }

    private async Task<List<ScoredRecipe>> ScoreRecipesAsync(
        List<Recipe> recipes, 
        Family family, 
        MealSuggestionRequest request)
    {
        var scoredRecipes = new List<ScoredRecipe>();

        foreach (var recipe in recipes)
        {
            var score = await _ruleEngine.CalculateFamilyFitScoreAsync(recipe, family, request);
            if (score >= 5.0m) // Minimum acceptable score
            {
                scoredRecipes.Add(new ScoredRecipe { Recipe = recipe, Score = score });
            }
        }

        return scoredRecipes.OrderByDescending(sr => sr.Score).ToList();
    }

    private MealSuggestion ConvertToMealSuggestion(Recipe recipe, decimal score, Family family)
    {
        return new MealSuggestion
        {
            Id = recipe.Id,
            Name = recipe.Name,
            Description = recipe.Description,
            PrepTime = recipe.PrepTime,
            CookTime = recipe.CookTime,
            Servings = recipe.Servings,
            EstimatedCost = recipe.EstimatedCost ?? 0,
            Cuisine = recipe.Cuisine,
            Difficulty = recipe.Difficulty,
            FamilyFitScore = score,
            Reasoning = GenerateRuleBasedReasoning(recipe, family, score),
            Ingredients = recipe.Ingredients,
            Instructions = recipe.Instructions,
            NutritionalInfo = recipe.NutritionalInfo,
            Tags = recipe.Tags,
            AllergenWarnings = recipe.AllergenWarnings,
            Source = "Rule-Based Suggestion",
            GeneratedAt = DateTime.UtcNow,
            Confidence = CalculateRuleBasedConfidence(score)
        };
    }

    private string GenerateRuleBasedReasoning(Recipe recipe, Family family, decimal score)
    {
        var reasons = new List<string>();

        // Check dietary compatibility
        if (family.GetCommonDietaryRestrictions().All(dr => recipe.Tags.Contains(dr, StringComparer.OrdinalIgnoreCase)))
        {
            reasons.Add("meets all family dietary requirements");
        }

        // Check preferred ingredients
        var familyLikes = family.Members.SelectMany(m => m.LikedIngredients).ToList();
        var recipeIngredients = recipe.Ingredients.Select(i => i.Name).ToList();
        var commonLikes = recipeIngredients.Intersect(familyLikes, StringComparer.OrdinalIgnoreCase).Count();
        
        if (commonLikes > 0)
        {
            reasons.Add($"includes {commonLikes} ingredients your family loves");
        }

        // Check cuisine preference
        var preferredCuisines = family.GetPreferredCuisines();
        if (preferredCuisines.Contains(recipe.Cuisine, StringComparer.OrdinalIgnoreCase))
        {
            reasons.Add($"matches your family's love for {recipe.Cuisine} cuisine");
        }

        // Cost efficiency
        if (recipe.EstimatedCost <= family.AverageMealBudget * 0.8m)
        {
            reasons.Add("fits well within your typical meal budget");
        }

        return $"This recipe {string.Join(", ", reasons)}.";
    }
}
```

### 3. Cached Response Strategy
```csharp
public class CachedResponseFallbackStrategy : IFallbackStrategy
{
    private readonly IResponseCacheService _cacheService;
    private readonly ISimilarityCalculator _similarityCalculator;

    public async Task<FallbackResult<T>> ExecuteAsync<T>(FallbackContext context) where T : class
    {
        try
        {
            // Look for exact cache hits first
            var exactMatch = await _cacheService.GetExactMatchAsync<T>(context.CacheKey);
            if (exactMatch != null)
            {
                return FallbackResult<T>.Success(exactMatch, "Exact cache hit");
            }

            // Look for similar requests
            var similarResponses = await _cacheService.GetSimilarResponsesAsync<T>(
                context.CacheKey, 
                similarityThreshold: 0.8);

            if (similarResponses.Any())
            {
                var bestMatch = similarResponses.First();
                var adaptedResponse = await AdaptCachedResponse<T>(bestMatch, context);
                
                return FallbackResult<T>.Success(adaptedResponse, "Similar cache hit adapted");
            }

            // Look for user's historical preferences
            var userHistory = await _cacheService.GetUserHistoryAsync<T>(
                context.UserId, 
                context.RequestType,
                limit: 10);

            if (userHistory.Any())
            {
                var personalizedResponse = await CreatePersonalizedFromHistory<T>(userHistory, context);
                return FallbackResult<T>.Success(personalizedResponse, "Generated from user history");
            }

            return FallbackResult<T>.Failed("No suitable cached responses found");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Cached response fallback failed");
            return FallbackResult<T>.Failed($"Cache fallback error: {ex.Message}");
        }
    }

    private async Task<T> AdaptCachedResponse<T>(CachedResponse<T> cachedResponse, FallbackContext context)
    {
        if (context.RequestType == RequestType.MealSuggestion && cachedResponse.Response is MealSuggestionResponse cached)
        {
            var adapted = cached.DeepClone();
            
            // Update constraints to match current request
            var currentRequest = context.OriginalRequest as MealSuggestionRequest;
            
            // Filter suggestions that still meet current constraints
            adapted.Suggestions = adapted.Suggestions
                .Where(s => s.EstimatedCost <= currentRequest.Budget)
                .Where(s => s.PrepTime <= currentRequest.MaxPrepTime)
                .Take(3)
                .ToList();

            // Update metadata
            foreach (var suggestion in adapted.Suggestions)
            {
                suggestion.GeneratedAt = DateTime.UtcNow;
                suggestion.Source = "Cached (Adapted)";
                suggestion.Confidence *= 0.8m; // Reduce confidence for adapted responses
            }

            return (T)(object)adapted;
        }

        return cachedResponse.Response;
    }

    private async Task<T> CreatePersonalizedFromHistory<T>(
        List<HistoricalResponse<T>> history, 
        FallbackContext context)
    {
        if (context.RequestType == RequestType.MealSuggestion)
        {
            var historicalSuggestions = history
                .SelectMany(h => h.Response as MealSuggestionResponse?.Suggestions ?? new List<MealSuggestion>())
                .Where(s => s.FamilyFitScore >= 7) // Only well-liked suggestions
                .OrderByDescending(s => s.FamilyFitScore)
                .Take(10)
                .ToList();

            // Find patterns in successful suggestions
            var preferredCuisines = historicalSuggestions
                .GroupBy(s => s.Cuisine)
                .OrderByDescending(g => g.Count())
                .Take(3)
                .Select(g => g.Key)
                .ToList();

            var preferredIngredients = historicalSuggestions
                .SelectMany(s => s.Ingredients.Select(i => i.Name))
                .GroupBy(i => i)
                .OrderByDescending(g => g.Count())
                .Take(20)
                .Select(g => g.Key)
                .ToList();

            // Generate new suggestions based on patterns
            var personalizedSuggestions = await GeneratePersonalizedSuggestions(
                preferredCuisines, 
                preferredIngredients, 
                context);

            var response = new MealSuggestionResponse { Suggestions = personalizedSuggestions };
            return (T)(object)response;
        }

        // For other request types, return the most recent successful response
        return history.OrderByDescending(h => h.Timestamp).First().Response;
    }
}
```

### 4. Default Response Strategy
```csharp
public class DefaultResponseFallbackStrategy : IFallbackStrategy
{
    private readonly IDefaultContentRepository _defaultContentRepository;
    private readonly IPopularRecipeService _popularRecipeService;

    public async Task<FallbackResult<T>> ExecuteAsync<T>(FallbackContext context) where T : class
    {
        try
        {
            if (context.RequestType == RequestType.MealSuggestion)
            {
                return await GenerateDefaultMealSuggestions<T>(context);
            }
            else if (context.RequestType == RequestType.WeeklyMenu)
            {
                return await GenerateDefaultWeeklyMenu<T>(context);
            }

            return FallbackResult<T>.Failed("No default response available for request type");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Default response fallback failed");
            return FallbackResult<T>.Failed($"Default fallback error: {ex.Message}");
        }
    }

    private async Task<FallbackResult<T>> GenerateDefaultMealSuggestions<T>(FallbackContext context)
    {
        var request = context.OriginalRequest as MealSuggestionRequest;
        
        // Get popular recipes that meet basic constraints
        var popularRecipes = await _popularRecipeService.GetPopularRecipesAsync(new PopularRecipeFilter
        {
            MealType = request.MealType,
            MaxCost = request.Budget,
            MaxTotalTime = request.MaxPrepTime + request.MaxCookTime,
            IsAllergenFree = true, // Safe defaults
            Limit = 5
        });

        var suggestions = popularRecipes.Take(3).Select(recipe => new MealSuggestion
        {
            Id = recipe.Id,
            Name = recipe.Name,
            Description = recipe.Description + " (Popular choice)",
            PrepTime = recipe.PrepTime,
            CookTime = recipe.CookTime,
            Servings = recipe.Servings,
            EstimatedCost = recipe.EstimatedCost ?? 0,
            Cuisine = recipe.Cuisine,
            Difficulty = recipe.Difficulty,
            FamilyFitScore = 6.0m, // Neutral score
            Reasoning = "This is a popular recipe that many families enjoy.",
            Ingredients = recipe.Ingredients,
            Instructions = recipe.Instructions,
            NutritionalInfo = recipe.NutritionalInfo,
            Tags = recipe.Tags,
            Source = "Default Popular Recipe",
            GeneratedAt = DateTime.UtcNow,
            Confidence = 0.5m // Lower confidence for default responses
        }).ToList();

        var response = new MealSuggestionResponse { Suggestions = suggestions };
        
        return FallbackResult<T>.Success((T)(object)response, "Default popular recipes provided");
    }
}
```

## Fallback Orchestration

### Fallback Coordinator
```csharp
public class FallbackCoordinator : IFallbackCoordinator
{
    private readonly IFallbackDecisionEngine _decisionEngine;
    private readonly IFallbackStrategyFactory _strategyFactory;
    private readonly IFallbackMonitor _monitor;

    public async Task<FallbackResult<T>> ExecuteFallbackAsync<T>(FallbackContext context) where T : class
    {
        var fallbackChain = new List<FallbackResult<T>>();
        var currentLevel = FallbackLevel.SecondaryAi;

        while (currentLevel <= FallbackLevel.Default)
        {
            try
            {
                var strategy = await _strategyFactory.GetStrategyAsync(currentLevel, context);
                var result = await strategy.ExecuteAsync<T>(context);
                
                fallbackChain.Add(result);
                
                if (result.IsSuccessful && await ValidateFallbackResult(result, context))
                {
                    await _monitor.TrackSuccessfulFallbackAsync(currentLevel, context);
                    return result;
                }
                
                // Move to next fallback level
                currentLevel = GetNextFallbackLevel(currentLevel);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Fallback strategy {Level} failed", currentLevel);
                await _monitor.TrackFailedFallbackAsync(currentLevel, context, ex);
                currentLevel = GetNextFallbackLevel(currentLevel);
            }
        }

        // All fallback strategies failed
        var finalResult = FallbackResult<T>.Failed("All fallback strategies exhausted");
        await _monitor.TrackCompleteFallbackFailureAsync(context, fallbackChain);
        
        return finalResult;
    }

    private async Task<bool> ValidateFallbackResult<T>(FallbackResult<T> result, FallbackContext context)
    {
        // Basic validation
        if (result.Data == null) return false;
        
        // Type-specific validation
        if (result.Data is MealSuggestionResponse mealResponse)
        {
            return ValidateMealSuggestionFallback(mealResponse, context);
        }
        
        return true; // Default to valid for unknown types
    }

    private bool ValidateMealSuggestionFallback(MealSuggestionResponse response, FallbackContext context)
    {
        // Must have at least one suggestion
        if (!response.Suggestions.Any()) return false;
        
        var request = context.OriginalRequest as MealSuggestionRequest;
        
        // All suggestions must meet basic constraints
        return response.Suggestions.All(s => 
            s.EstimatedCost <= request.Budget * 1.1m && // Allow 10% budget variance
            s.PrepTime <= request.MaxPrepTime * 1.2m &&  // Allow 20% time variance
            !string.IsNullOrWhiteSpace(s.Name) &&
            s.Ingredients.Any());
    }
}
```

## Fallback Quality Management

### Quality Degradation Handling
```csharp
public class FallbackQualityManager : IFallbackQualityManager
{
    public async Task<QualityManagedResult<T>> ManageQualityDegradationAsync<T>(
        FallbackResult<T> fallbackResult,
        FallbackContext context) where T : class
    {
        var qualityLevel = AssessFallbackQuality(fallbackResult);
        var managedResult = new QualityManagedResult<T>
        {
            Data = fallbackResult.Data,
            OriginalQuality = qualityLevel,
            IsQualityAcceptable = qualityLevel >= QualityLevel.Acceptable
        };

        // Apply quality improvements
        if (qualityLevel < QualityLevel.Good)
        {
            managedResult.Data = await ApplyQualityImprovements(fallbackResult.Data, context);
            managedResult.ImprovementsApplied = true;
        }

        // Add quality disclaimers
        managedResult.QualityDisclaimer = GenerateQualityDisclaimer(qualityLevel, fallbackResult.Source);
        
        // Recommend alternatives if quality is poor
        if (qualityLevel < QualityLevel.Acceptable)
        {
            managedResult.AlternativeRecommendations = await GenerateAlternativeRecommendations(context);
        }

        return managedResult;
    }

    private async Task<T> ApplyQualityImprovements<T>(T data, FallbackContext context)
    {
        if (data is MealSuggestionResponse mealResponse)
        {
            var improved = mealResponse.DeepClone();
            
            // Enhance suggestions with additional context
            foreach (var suggestion in improved.Suggestions)
            {
                // Add cooking tips for simpler suggestions
                if (suggestion.CookingTips?.Count < 2)
                {
                    suggestion.CookingTips = await GetBasicCookingTips(suggestion);
                }
                
                // Improve reasoning if generic
                if (suggestion.Reasoning?.Length < 50)
                {
                    suggestion.Reasoning = await GenerateImprovedReasoning(suggestion, context);
                }
                
                // Add substitution suggestions
                if (suggestion.Ingredients.Any(i => i.Substitutions?.Count == 0))
                {
                    await AddIngredientSubstitutions(suggestion);
                }
            }
            
            return (T)(object)improved;
        }

        return data;
    }

    private string GenerateQualityDisclaimer(QualityLevel quality, string source)
    {
        return quality switch
        {
            QualityLevel.Poor => "These suggestions are basic recommendations. For personalized options, please try again later when our AI service is available.",
            QualityLevel.Acceptable => "These suggestions are generated using simplified algorithms. Quality may be lower than our usual AI-powered recommendations.",
            QualityLevel.Good => $"These suggestions are provided by {source} and may differ slightly from our primary AI recommendations.",
            _ => null
        };
    }
}
```

## Monitoring and Analytics

### Fallback Performance Tracking
```csharp
public class FallbackAnalytics : IFallbackAnalytics
{
    public async Task TrackFallbackPatterns()
    {
        var recentFallbacks = await _repository.GetRecentFallbacksAsync(TimeSpan.FromDays(7));
        
        var analytics = new FallbackAnalytics
        {
            TotalFallbacks = recentFallbacks.Count,
            FallbacksByStrategy = recentFallbacks.GroupBy(f => f.Strategy).ToDictionary(g => g.Key, g => g.Count()),
            AverageQualityScore = recentFallbacks.Average(f => f.QualityScore),
            UserSatisfactionRate = await CalculateUserSatisfactionRate(recentFallbacks),
            MostCommonFailureReasons = GetMostCommonFailureReasons(recentFallbacks),
            RecoveryTimeStatistics = CalculateRecoveryTimeStats(recentFallbacks)
        };

        await _dashboardService.UpdateFallbackAnalyticsAsync(analytics);
        
        // Generate alerts for concerning patterns
        await CheckForAlarming Patterns(analytics);
    }

    private async Task CheckForAlarmingPatterns(FallbackAnalytics analytics)
    {
        // High fallback rate
        if (analytics.FallbackRate > 0.15) // More than 15% of requests
        {
            await _alertService.SendFallbackRateAlertAsync(analytics.FallbackRate);
        }

        // Low quality scores
        if (analytics.AverageQualityScore < 0.6)
        {
            await _alertService.SendQualityDegradationAlertAsync(analytics.AverageQualityScore);
        }

        // Primary AI health issues
        var primaryAiFailureRate = analytics.FallbacksByStrategy.GetValueOrDefault("SecondaryAi", 0) / 
                                  (double)analytics.TotalFallbacks;
        
        if (primaryAiFailureRate > 0.5)
        {
            await _alertService.SendPrimaryAiHealthAlertAsync(primaryAiFailureRate);
        }
    }
}
```

This comprehensive fallback strategy system ensures that the MealPrep application maintains functionality and user satisfaction even when primary AI services are unavailable or degraded, with multiple layers of intelligent fallback options and quality management.

*This guide should be updated as fallback strategies evolve and new AI providers or techniques become available.*