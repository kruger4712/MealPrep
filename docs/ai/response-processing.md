# AI Response Processing

## Overview
Comprehensive guide for processing, validating, and enhancing AI responses from Google Gemini in the MealPrep application, ensuring high-quality, safe, and actionable meal suggestions.

## Response Processing Architecture

### Processing Pipeline
```
AI Raw Response ? JSON Parsing ? Validation ? Enhancement ? Storage ? Delivery
       ?              ?            ?            ?          ?          ?
   Text Format    Structured   Business Rules  Enrichment  Cache    User Interface
                     Data        Validation     & Scoring   Data    
```

### Core Processing Components
```csharp
public class AiResponseProcessor : IAiResponseProcessor
{
    private readonly ILogger<AiResponseProcessor> _logger;
    private readonly IResponseValidator _validator;
    private readonly IResponseEnhancer _enhancer;
    private readonly IIngredientService _ingredientService;
    private readonly IRecipeRepository _recipeRepository;
    private readonly IFamilyService _familyService;
    private readonly IResponseCache _cache;

    public async Task<ProcessedResponse<T>> ProcessResponseAsync<T>(
        string rawResponse,
        ResponseProcessingContext context) where T : class
    {
        var processingResult = new ProcessedResponse<T>
        {
            RequestId = context.RequestId,
            ProcessedAt = DateTime.UtcNow,
            ProcessingTimeMs = 0
        };

        var stopwatch = Stopwatch.StartNew();

        try
        {
            // Step 1: Parse JSON response
            var parseResult = await ParseJsonResponseAsync<T>(rawResponse);
            processingResult.ParsedData = parseResult.Data;
            processingResult.ParseErrors = parseResult.Errors;

            if (!parseResult.IsSuccess)
            {
                processingResult.Status = ProcessingStatus.ParseError;
                return processingResult;
            }

            // Step 2: Validate business rules
            var validationResult = await _validator.ValidateAsync(parseResult.Data, context);
            processingResult.ValidationErrors = validationResult.Errors;
            processingResult.ValidationWarnings = validationResult.Warnings;

            if (!validationResult.IsValid)
            {
                processingResult.Status = ProcessingStatus.ValidationError;
                return processingResult;
            }

            // Step 3: Enhance with additional data
            var enhancedData = await _enhancer.EnhanceAsync(parseResult.Data, context);
            processingResult.ProcessedData = enhancedData;

            // Step 4: Calculate quality scores
            processingResult.QualityScore = await CalculateQualityScoreAsync(enhancedData, context);
            
            processingResult.Status = ProcessingStatus.Success;
            return processingResult;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process AI response for request {RequestId}", context.RequestId);
            processingResult.Status = ProcessingStatus.ProcessingError;
            processingResult.ProcessingErrors.Add(ex.Message);
            return processingResult;
        }
        finally
        {
            stopwatch.Stop();
            processingResult.ProcessingTimeMs = stopwatch.ElapsedMilliseconds;
        }
    }
}
```

## JSON Response Parsing

### Robust JSON Parser
```csharp
public class AiJsonParser : IAiJsonParser
{
    private readonly ILogger<AiJsonParser> _logger;

    public async Task<ParseResult<T>> ParseJsonResponseAsync<T>(string rawResponse) where T : class
    {
        var result = new ParseResult<T>();

        try
        {
            // Clean the response (remove markdown code blocks, extra whitespace)
            var cleanedResponse = CleanJsonResponse(rawResponse);
            
            // Attempt primary parsing
            try
            {
                result.Data = JsonSerializer.Deserialize<T>(cleanedResponse, GetJsonOptions());
                result.IsSuccess = true;
                return result;
            }
            catch (JsonException primaryEx)
            {
                _logger.LogWarning("Primary JSON parse failed: {Error}", primaryEx.Message);
                
                // Attempt recovery strategies
                var recoveredData = await AttemptJsonRecoveryAsync<T>(cleanedResponse);
                if (recoveredData != null)
                {
                    result.Data = recoveredData;
                    result.IsSuccess = true;
                    result.Warnings.Add("Response recovered using fallback parsing");
                    return result;
                }
                
                result.Errors.Add($"JSON parsing failed: {primaryEx.Message}");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error during JSON parsing");
            result.Errors.Add($"Parsing error: {ex.Message}");
        }

        result.IsSuccess = false;
        return result;
    }

    private string CleanJsonResponse(string rawResponse)
    {
        // Remove markdown code blocks
        var cleaned = Regex.Replace(rawResponse, @"```json\s*", "", RegexOptions.IgnoreCase);
        cleaned = Regex.Replace(cleaned, @"\s*```", "");
        
        // Remove extra whitespace and newlines
        cleaned = cleaned.Trim();
        
        // Find JSON object bounds
        var startIndex = cleaned.IndexOf('{');
        var lastIndex = cleaned.LastIndexOf('}');
        
        if (startIndex >= 0 && lastIndex > startIndex)
        {
            cleaned = cleaned.Substring(startIndex, lastIndex - startIndex + 1);
        }
        
        return cleaned;
    }

    private async Task<T> AttemptJsonRecoveryAsync<T>(string malformedJson) where T : class
    {
        var recoveryStrategies = new List<Func<string, Task<T>>>
        {
            TryFixCommonJsonErrors,
            TryPartialParsing,
            TryManualExtraction
        };

        foreach (var strategy in recoveryStrategies)
        {
            try
            {
                var result = await strategy(malformedJson);
                if (result != null)
                {
                    _logger.LogInformation("JSON recovery successful using strategy: {Strategy}", strategy.Method.Name);
                    return result;
                }
            }
            catch (Exception ex)
            {
                _logger.LogDebug("Recovery strategy {Strategy} failed: {Error}", strategy.Method.Name, ex.Message);
            }
        }

        return null;
    }

    private async Task<T> TryFixCommonJsonErrors<T>(string json) where T : class
    {
        // Fix trailing commas
        json = Regex.Replace(json, @",\s*}", "}");
        json = Regex.Replace(json, @",\s*]", "]");
        
        // Fix unescaped quotes
        json = Regex.Replace(json, @"(?<!\\)""(?=\w)", "\\\"");
        
        // Fix missing quotes around property names
        json = Regex.Replace(json, @"(\w+):", "\"$1\":");
        
        return JsonSerializer.Deserialize<T>(json, GetJsonOptions());
    }
}
```

### Meal Suggestion Response Parser
```csharp
public class MealSuggestionResponseParser : ISpecializedParser<MealSuggestionResponse>
{
    public async Task<MealSuggestionResponse> ParseAsync(string rawResponse)
    {
        var baseParser = new AiJsonParser();
        var parseResult = await baseParser.ParseJsonResponseAsync<MealSuggestionResponseRaw>(rawResponse);
        
        if (!parseResult.IsSuccess)
        {
            throw new ResponseParsingException("Failed to parse meal suggestion response", parseResult.Errors);
        }

        // Convert raw response to domain objects
        var response = new MealSuggestionResponse
        {
            Suggestions = new List<MealSuggestion>()
        };

        foreach (var rawSuggestion in parseResult.Data.Suggestions)
        {
            var suggestion = await ConvertToMealSuggestion(rawSuggestion);
            response.Suggestions.Add(suggestion);
        }

        return response;
    }

    private async Task<MealSuggestion> ConvertToMealSuggestion(RawMealSuggestion raw)
    {
        return new MealSuggestion
        {
            Id = Guid.NewGuid().ToString(),
            Name = SanitizeText(raw.Name),
            Description = SanitizeText(raw.Description),
            PrepTime = ValidateTimeValue(raw.PrepTime, "PrepTime"),
            CookTime = ValidateTimeValue(raw.CookTime, "CookTime"),
            Servings = ValidateServingsValue(raw.Servings),
            EstimatedCost = ValidateCostValue(raw.EstimatedCost),
            Cuisine = SanitizeText(raw.Cuisine),
            Difficulty = ValidateDifficultyValue(raw.Difficulty),
            FamilyFitScore = ValidateFamilyFitScore(raw.FamilyFitScore),
            Reasoning = SanitizeText(raw.Reasoning),
            Ingredients = await ConvertIngredients(raw.Ingredients),
            Instructions = SanitizeInstructions(raw.Instructions),
            NutritionalInfo = ConvertNutritionalInfo(raw.NutritionalInfo),
            Tags = SanitizeTags(raw.Tags),
            AllergenWarnings = ValidateAllergens(raw.AllergenWarnings),
            Source = "AI Generated",
            GeneratedAt = DateTime.UtcNow,
            Confidence = CalculateConfidenceScore(raw)
        };
    }

    private decimal CalculateConfidenceScore(RawMealSuggestion raw)
    {
        var score = 1.0m;
        
        // Deduct for missing or suspicious data
        if (string.IsNullOrWhiteSpace(raw.Name)) score -= 0.3m;
        if (string.IsNullOrWhiteSpace(raw.Description)) score -= 0.1m;
        if (raw.PrepTime <= 0 || raw.PrepTime > 300) score -= 0.2m;
        if (raw.EstimatedCost <= 0 || raw.EstimatedCost > 200) score -= 0.2m;
        if (raw.FamilyFitScore < 1 || raw.FamilyFitScore > 10) score -= 0.3m;
        if (raw.Ingredients == null || !raw.Ingredients.Any()) score -= 0.4m;
        if (raw.Instructions == null || !raw.Instructions.Any()) score -= 0.3m;

        return Math.Max(0.1m, Math.Min(1.0m, score));
    }
}
```

## Response Validation

### Business Rule Validator
```csharp
public class MealSuggestionValidator : IResponseValidator<MealSuggestionResponse>
{
    private readonly IFamilyService _familyService;
    private readonly IIngredientService _ingredientService;

    public async Task<ValidationResult> ValidateAsync(
        MealSuggestionResponse response, 
        ResponseProcessingContext context)
    {
        var result = new ValidationResult();
        var family = await _familyService.GetFamilyAsync(context.FamilyId);

        foreach (var suggestion in response.Suggestions)
        {
            // Critical validations (failures)
            await ValidateCriticalConstraints(suggestion, family, context, result);
            
            // Important validations (warnings)
            await ValidateImportantConstraints(suggestion, family, context, result);
            
            // Quality validations (suggestions)
            await ValidateQualityConstraints(suggestion, family, context, result);
        }

        return result;
    }

    private async Task ValidateCriticalConstraints(
        MealSuggestion suggestion,
        Family family,
        ResponseProcessingContext context,
        ValidationResult result)
    {
        // Allergen safety check
        var familyAllergens = family.GetAllAllergens();
        foreach (var ingredient in suggestion.Ingredients)
        {
            var ingredientAllergens = await _ingredientService.GetAllergensAsync(ingredient.Name);
            var conflicts = ingredientAllergens.Intersect(familyAllergens, StringComparer.OrdinalIgnoreCase);
            
            if (conflicts.Any())
            {
                result.AddError($"CRITICAL: Recipe '{suggestion.Name}' contains allergen '{string.Join(", ", conflicts)}' in ingredient '{ingredient.Name}'");
            }
        }

        // Budget constraint check
        if (context.BudgetLimit.HasValue && suggestion.EstimatedCost > context.BudgetLimit.Value)
        {
            result.AddError($"Recipe '{suggestion.Name}' cost ${suggestion.EstimatedCost:F2} exceeds budget ${context.BudgetLimit:F2}");
        }

        // Time constraint check
        if (context.MaxPrepTime.HasValue && suggestion.PrepTime > context.MaxPrepTime.Value)
        {
            result.AddError($"Recipe '{suggestion.Name}' prep time {suggestion.PrepTime}min exceeds limit {context.MaxPrepTime}min");
        }

        // Dietary restriction compliance
        foreach (var member in family.Members)
        {
            var violations = await CheckDietaryRestrictionViolations(suggestion, member);
            if (violations.Any())
            {
                result.AddError($"Recipe '{suggestion.Name}' violates {member.Name}'s dietary restrictions: {string.Join(", ", violations)}");
            }
        }
    }

    private async Task ValidateImportantConstraints(
        MealSuggestion suggestion,
        Family family,
        ResponseProcessingContext context,
        ValidationResult result)
    {
        // Serving size appropriateness
        var familySize = family.Members.Count;
        if (suggestion.Servings < familySize * 0.5 || suggestion.Servings > familySize * 2)
        {
            result.AddWarning($"Recipe '{suggestion.Name}' serves {suggestion.Servings} but family has {familySize} members");
        }

        // Ingredient availability
        var unavailableIngredients = await GetUnavailableIngredients(suggestion.Ingredients, context.AvailableIngredients);
        if (unavailableIngredients.Count > suggestion.Ingredients.Count * 0.5)
        {
            result.AddWarning($"Recipe '{suggestion.Name}' requires many unavailable ingredients: {string.Join(", ", unavailableIngredients.Take(3))}");
        }

        // Family preference alignment
        var familyFitScore = await CalculateFamilyFitScore(suggestion, family);
        if (familyFitScore < 5)
        {
            result.AddWarning($"Recipe '{suggestion.Name}' has low family fit score: {familyFitScore}/10");
        }
    }

    private async Task ValidateQualityConstraints(
        MealSuggestion suggestion,
        Family family,
        ResponseProcessingContext context,
        ValidationResult result)
    {
        // Nutritional balance
        var nutritionalScore = CalculateNutritionalScore(suggestion.NutritionalInfo);
        if (nutritionalScore < 6)
        {
            result.AddSuggestion($"Recipe '{suggestion.Name}' could be more nutritionally balanced");
        }

        // Recipe complexity vs family cooking skill
        var complexityMismatch = await CheckComplexityMismatch(suggestion, family);
        if (complexityMismatch)
        {
            result.AddSuggestion($"Recipe '{suggestion.Name}' may be too complex for family's cooking skill level");
        }

        // Seasonal appropriateness
        var isSeasonallyAppropriate = await CheckSeasonalAppropriateness(suggestion, DateTime.Now);
        if (!isSeasonallyAppropriate)
        {
            result.AddSuggestion($"Recipe '{suggestion.Name}' includes out-of-season ingredients");
        }
    }
}
```

## Response Enhancement

### Data Enrichment Engine
```csharp
public class MealSuggestionEnhancer : IResponseEnhancer<MealSuggestionResponse>
{
    private readonly IIngredientService _ingredientService;
    private readonly IRecipeRepository _recipeRepository;
    private readonly INutritionCalculator _nutritionCalculator;
    private readonly IImageService _imageService;

    public async Task<MealSuggestionResponse> EnhanceAsync(
        MealSuggestionResponse response,
        ResponseProcessingContext context)
    {
        var enhancedResponse = response.DeepClone();

        foreach (var suggestion in enhancedResponse.Suggestions)
        {
            await EnhanceSuggestion(suggestion, context);
        }

        return enhancedResponse;
    }

    private async Task EnhanceSuggestion(MealSuggestion suggestion, ResponseProcessingContext context)
    {
        // Enhance ingredients with detailed information
        await EnhanceIngredients(suggestion);
        
        // Improve nutritional information accuracy
        await EnhanceNutritionalInfo(suggestion);
        
        // Add cooking tips and techniques
        await AddCookingTips(suggestion);
        
        // Generate or find recipe image
        await AddRecipeImage(suggestion);
        
        // Add meal prep suggestions
        await AddMealPrepSuggestions(suggestion);
        
        // Calculate enhanced family fit score
        await UpdateFamilyFitScore(suggestion, context);
        
        // Add similar recipe suggestions
        await AddSimilarRecipes(suggestion);
        
        // Enhance cost breakdown
        await EnhanceCostBreakdown(suggestion);
    }

    private async Task EnhanceIngredients(MealSuggestion suggestion)
    {
        for (int i = 0; i < suggestion.Ingredients.Count; i++)
        {
            var ingredient = suggestion.Ingredients[i];
            
            // Get detailed ingredient information
            var detailedIngredient = await _ingredientService.GetDetailedAsync(ingredient.Name);
            if (detailedIngredient != null)
            {
                // Update with more accurate cost
                ingredient.EstimatedCost = detailedIngredient.EstimatedCost * ingredient.Quantity;
                
                // Add nutritional contribution
                ingredient.NutritionalContribution = CalculateNutritionalContribution(
                    detailedIngredient.NutritionalInfo, 
                    ingredient.Quantity, 
                    ingredient.Unit);
                
                // Add storage and prep tips
                ingredient.StorageTips = detailedIngredient.StorageInstructions;
                ingredient.PrepTips = GetPrepTips(detailedIngredient, ingredient.Quantity);
                
                // Suggest substitutions
                ingredient.Substitutions = await GetIngredientSubstitutions(detailedIngredient, suggestion);
            }
        }
    }

    private async Task EnhanceNutritionalInfo(MealSuggestion suggestion)
    {
        // Recalculate nutritional information based on ingredients
        var calculatedNutrition = await _nutritionCalculator.CalculateAsync(
            suggestion.Ingredients, 
            suggestion.Servings);

        // Blend AI-provided nutrition with calculated values
        suggestion.NutritionalInfo = BlendNutritionalData(
            suggestion.NutritionalInfo, 
            calculatedNutrition);

        // Add nutritional highlights
        suggestion.NutritionalHighlights = IdentifyNutritionalHighlights(suggestion.NutritionalInfo);
        
        // Calculate health score
        suggestion.HealthScore = CalculateHealthScore(suggestion.NutritionalInfo);
    }

    private async Task AddCookingTips(MealSuggestion suggestion)
    {
        var tips = new List<string>();
        
        // Technique-based tips
        var techniques = ExtractCookingTechniques(suggestion.Instructions);
        foreach (var technique in techniques)
        {
            var techniqueTips = await GetTechniqueSpecificTips(technique);
            tips.AddRange(techniqueTips);
        }
        
        // Ingredient-based tips
        var specialIngredients = GetSpecialIngredients(suggestion.Ingredients);
        foreach (var ingredient in specialIngredients)
        {
            var ingredientTips = await GetIngredientSpecificTips(ingredient);
            tips.AddRange(ingredientTips);
        }
        
        // Time-saving tips
        if (suggestion.PrepTime > 30)
        {
            tips.AddRange(GetTimeSavingTips(suggestion));
        }
        
        // Difficulty-specific tips
        if (suggestion.Difficulty == "Hard")
        {
            tips.AddRange(GetComplexRecipeTips(suggestion));
        }

        suggestion.CookingTips = tips.Distinct().Take(5).ToList();
    }

    private async Task AddMealPrepSuggestions(MealSuggestion suggestion)
    {
        var mealPrepSuggestions = new MealPrepSuggestions();
        
        // Components that can be prepped ahead
        mealPrepSuggestions.PrepAheadComponents = IdentifyPrepAheadComponents(suggestion);
        
        // Storage instructions for prepped components
        mealPrepSuggestions.StorageInstructions = GetStorageInstructions(suggestion);
        
        // Assembly instructions for final cooking
        mealPrepSuggestions.AssemblyInstructions = GenerateAssemblyInstructions(suggestion);
        
        // Freezer-friendly modifications
        mealPrepSuggestions.FreezerInstructions = GetFreezerInstructions(suggestion);
        
        suggestion.MealPrepSuggestions = mealPrepSuggestions;
    }
}
```

## Quality Scoring

### Response Quality Calculator
```csharp
public class ResponseQualityCalculator : IQualityCalculator
{
    public async Task<QualityScore> CalculateQualityScoreAsync<T>(T response, ResponseProcessingContext context)
    {
        var score = new QualityScore
        {
            ResponseId = context.RequestId,
            CalculatedAt = DateTime.UtcNow
        };

        if (response is MealSuggestionResponse mealResponse)
        {
            score = await CalculateMealSuggestionQuality(mealResponse, context);
        }

        return score;
    }

    private async Task<QualityScore> CalculateMealSuggestionQuality(
        MealSuggestionResponse response,
        ResponseProcessingContext context)
    {
        var qualityFactors = new Dictionary<string, decimal>();
        
        // Completeness score (0-1)
        qualityFactors["completeness"] = CalculateCompletenessScore(response);
        
        // Accuracy score (0-1)
        qualityFactors["accuracy"] = await CalculateAccuracyScore(response, context);
        
        // Relevance score (0-1)
        qualityFactors["relevance"] = await CalculateRelevanceScore(response, context);
        
        // Safety score (0-1)
        qualityFactors["safety"] = await CalculateSafetyScore(response, context);
        
        // Diversity score (0-1)
        qualityFactors["diversity"] = CalculateDiversityScore(response);
        
        // Feasibility score (0-1)
        qualityFactors["feasibility"] = await CalculateFeasibilityScore(response, context);

        // Calculate weighted overall score
        var weights = new Dictionary<string, decimal>
        {
            ["completeness"] = 0.15m,
            ["accuracy"] = 0.20m,
            ["relevance"] = 0.25m,
            ["safety"] = 0.20m,
            ["diversity"] = 0.10m,
            ["feasibility"] = 0.10m
        };

        var overallScore = qualityFactors.Sum(kv => kv.Value * weights[kv.Key]);

        return new QualityScore
        {
            OverallScore = overallScore,
            ComponentScores = qualityFactors,
            ScoreBreakdown = GenerateScoreBreakdown(qualityFactors, weights),
            QualityLevel = DetermineQualityLevel(overallScore),
            Recommendations = GenerateQualityRecommendations(qualityFactors)
        };
    }

    private decimal CalculateCompletenessScore(MealSuggestionResponse response)
    {
        if (!response.Suggestions.Any()) return 0m;

        var totalFields = 0;
        var completedFields = 0;

        foreach (var suggestion in response.Suggestions)
        {
            var fieldChecks = new Dictionary<string, bool>
            {
                ["name"] = !string.IsNullOrWhiteSpace(suggestion.Name),
                ["description"] = !string.IsNullOrWhiteSpace(suggestion.Description),
                ["prepTime"] = suggestion.PrepTime > 0,
                ["cookTime"] = suggestion.CookTime >= 0,
                ["servings"] = suggestion.Servings > 0,
                ["cost"] = suggestion.EstimatedCost > 0,
                ["ingredients"] = suggestion.Ingredients.Any(),
                ["instructions"] = suggestion.Instructions.Any(),
                ["nutrition"] = suggestion.NutritionalInfo != null,
                ["familyFit"] = suggestion.FamilyFitScore >= 1 && suggestion.FamilyFitScore <= 10
            };

            totalFields += fieldChecks.Count;
            completedFields += fieldChecks.Count(kv => kv.Value);
        }

        return totalFields > 0 ? (decimal)completedFields / totalFields : 0m;
    }

    private async Task<decimal> CalculateAccuracyScore(
        MealSuggestionResponse response,
        ResponseProcessingContext context)
    {
        var accuracyChecks = new List<decimal>();

        foreach (var suggestion in response.Suggestions)
        {
            var checks = new List<bool>();
            
            // Cost accuracy (within reasonable range)
            var estimatedCost = await RecalculateCost(suggestion.Ingredients);
            var costAccuracy = Math.Abs(suggestion.EstimatedCost - estimatedCost) / estimatedCost;
            checks.Add(costAccuracy <= 0.3m); // Within 30%
            
            // Time accuracy (reasonable for complexity)
            var timeAccuracy = ValidateTimeEstimates(suggestion);
            checks.Add(timeAccuracy);
            
            // Nutritional accuracy
            var nutritionalAccuracy = await ValidateNutritionalInfo(suggestion);
            checks.Add(nutritionalAccuracy);
            
            // Serving size accuracy
            var servingAccuracy = ValidateServingSize(suggestion);
            checks.Add(servingAccuracy);

            accuracyChecks.Add((decimal)checks.Count(c => c) / checks.Count);
        }

        return accuracyChecks.Average();
    }

    private string DetermineQualityLevel(decimal overallScore)
    {
        return overallScore switch
        {
            >= 0.9m => "Excellent",
            >= 0.8m => "Good",
            >= 0.7m => "Acceptable",
            >= 0.6m => "Fair",
            _ => "Poor"
        };
    }
}
```

## Error Handling and Recovery

### Response Recovery Strategies
```csharp
public class ResponseRecoveryService : IResponseRecoveryService
{
    public async Task<RecoveryResult<T>> AttemptRecoveryAsync<T>(
        FailedResponse failedResponse,
        ResponseProcessingContext context) where T : class
    {
        var recoveryStrategies = new List<IRecoveryStrategy<T>>
        {
            new PartialDataRecoveryStrategy<T>(),
            new FallbackDataStrategy<T>(),
            new SimilarRequestCacheStrategy<T>(),
            new DefaultResponseStrategy<T>()
        };

        foreach (var strategy in recoveryStrategies)
        {
            try
            {
                var result = await strategy.AttemptRecoveryAsync(failedResponse, context);
                if (result.IsSuccessful)
                {
                    _logger.LogInformation("Response recovery successful using strategy: {Strategy}", 
                        strategy.GetType().Name);
                    return result;
                }
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Recovery strategy {Strategy} failed", strategy.GetType().Name);
            }
        }

        return RecoveryResult<T>.Failed("All recovery strategies exhausted");
    }
}

public class PartialDataRecoveryStrategy<T> : IRecoveryStrategy<T> where T : class
{
    public async Task<RecoveryResult<T>> AttemptRecoveryAsync(
        FailedResponse failedResponse,
        ResponseProcessingContext context)
    {
        // Attempt to extract partial data from malformed response
        if (failedResponse.PartialData != null && failedResponse.PartialData.Any())
        {
            var partialResult = await BuildPartialResponse<T>(failedResponse.PartialData, context);
            if (partialResult != null)
            {
                return RecoveryResult<T>.Success(partialResult, "Partial data recovery");
            }
        }

        return RecoveryResult<T>.Failed("No recoverable partial data");
    }
}
```

## Performance Monitoring

### Response Processing Metrics
```csharp
public class ResponseProcessingMetrics : IResponseProcessingMetrics
{
    public async Task TrackProcessingMetricsAsync(ProcessingMetricsData data)
    {
        var metrics = new
        {
            RequestId = data.RequestId,
            ProcessingTimeMs = data.ProcessingTimeMs,
            ResponseSizeBytes = data.ResponseSizeBytes,
            ValidationErrors = data.ValidationErrors.Count,
            ValidationWarnings = data.ValidationWarnings.Count,
            QualityScore = data.QualityScore,
            EnhancementTimeMs = data.EnhancementTimeMs,
            ProcessingStatus = data.Status.ToString(),
            Timestamp = DateTime.UtcNow
        };

        await _metricsRepository.RecordAsync(metrics);
        
        // Check for performance degradation
        await CheckPerformanceThresholds(data);
    }

    private async Task CheckPerformanceThresholds(ProcessingMetricsData data)
    {
        var recentMetrics = await _metricsRepository.GetRecentMetricsAsync(TimeSpan.FromHours(1));
        
        var avgProcessingTime = recentMetrics.Average(m => m.ProcessingTimeMs);
        var avgQualityScore = recentMetrics.Average(m => m.QualityScore);
        var errorRate = recentMetrics.Count(m => m.ValidationErrors > 0) / (double)recentMetrics.Count;

        if (avgProcessingTime > 5000) // > 5 seconds
        {
            await _alertService.SendPerformanceAlertAsync("High processing time", avgProcessingTime);
        }

        if (avgQualityScore < 0.7) // Quality degradation
        {
            await _alertService.SendQualityAlertAsync("Quality score degradation", avgQualityScore);
        }

        if (errorRate > 0.1) // > 10% error rate
        {
            await _alertService.SendErrorRateAlertAsync("High validation error rate", errorRate);
        }
    }
}
```

This comprehensive AI response processing system ensures that raw AI responses are transformed into high-quality, safe, and actionable meal suggestions for the MealPrep application with robust validation, enhancement, and quality control mechanisms.

*This guide should be updated as AI response patterns evolve and new processing techniques are developed.*