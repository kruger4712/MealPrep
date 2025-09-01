# MealMenu AI Integration Guide

## Overview

This guide covers the integration of Google Gemini AI into the MealMenu family meal preparation application. The AI system provides intelligent meal suggestions, recipe recommendations, and personalized cooking guidance based on family personas and preferences.

## Google Gemini AI Setup

### Prerequisites
- Google Cloud Platform account
- Google Cloud AI Platform API enabled
- Service account with appropriate permissions
- Google.Cloud.AiPlatform NuGet package

### Configuration

#### appsettings.json
```json
{
  "GoogleCloud": {
    "ProjectId": "mealmenu-project-id",
    "Location": "us-central1",
    "Model": "gemini-pro",
    "ServiceAccountKeyPath": "path/to/service-account-key.json"
  },
  "AI": {
    "MaxTokens": 4096,
    "Temperature": 0.7,
    "TopP": 0.8,
    "TopK": 40,
    "SafetySettings": {
      "HateSpeech": "BLOCK_MEDIUM_AND_ABOVE",
      "DangerousContent": "BLOCK_MEDIUM_AND_ABOVE",
      "Harassment": "BLOCK_MEDIUM_AND_ABOVE",
      "SexuallyExplicit": "BLOCK_MEDIUM_AND_ABOVE"
    }
  }
}
```

#### Dependency Injection Setup
```csharp
// Program.cs
builder.Services.Configure<GoogleCloudOptions>(
    builder.Configuration.GetSection("GoogleCloud"));
builder.Services.Configure<AIOptions>(
    builder.Configuration.GetSection("AI"));

builder.Services.AddScoped<IGeminiAiService, GeminiAiService>();
builder.Services.AddScoped<IPromptEngineering, PromptEngineeringService>();
builder.Services.AddScoped<IPersonaModelingService, PersonaModelingService>();
```

## Family Persona Modeling

### Core Data Structures

#### FamilyPersona Model
```csharp
public class FamilyPersona
{
    public string Name { get; set; }
    public int Age { get; set; }
    public List<string> LikedIngredients { get; set; } = new();
    public List<string> DislikedIngredients { get; set; } = new();
    public List<string> Allergies { get; set; } = new();
    public List<DietaryRestriction> DietaryRestrictions { get; set; } = new();
    public List<string> CuisinePreferences { get; set; } = new();
    public int SpiceToleranceLevel { get; set; } // 1-10 scale
    public CookingSkillLevel CookingSkill { get; set; }
    public int PreferredCookingTime { get; set; } // Minutes
    public List<string> AvailableKitchenAppliances { get; set; } = new();
    public List<string> HealthGoals { get; set; } = new();
    public decimal WeeklyFoodBudget { get; set; }
}

public enum DietaryRestriction
{
    None,
    Vegetarian,
    Vegan,
    Keto,
    Paleo,
    Mediterranean,
    LowCarb,
    LowFat,
    GlutenFree,
    DairyFree,
    NutFree,
    Halal,
    Kosher
}

public enum CookingSkillLevel
{
    Beginner,
    Intermediate,
    Advanced,
    Expert
}
```

#### Meal Suggestion Request
```csharp
public class MealSuggestionRequest
{
    public List<FamilyPersona> FamilyMembers { get; set; } = new();
    public MealType MealType { get; set; }
    public DateTime TargetDate { get; set; }
    public List<AvailableIngredient> AvailableIngredients { get; set; } = new();
    public decimal Budget { get; set; }
    public OccasionType Occasion { get; set; }
    public int MaxPrepTime { get; set; }
    public int MaxCookTime { get; set; }
    public List<string> RecentMeals { get; set; } = new(); // To avoid repetition
    public SeasonalPreference SeasonalPreference { get; set; }
    public int DesiredServings { get; set; }
    public NutritionalGoals NutritionalGoals { get; set; }
}

public class AvailableIngredient
{
    public string Name { get; set; }
    public decimal Quantity { get; set; }
    public string Unit { get; set; }
    public DateTime? ExpirationDate { get; set; }
}

public class NutritionalGoals
{
    public int? MaxCalories { get; set; }
    public int? MinProtein { get; set; }
    public int? MaxCarbs { get; set; }
    public int? MaxFat { get; set; }
    public int? MinFiber { get; set; }
    public int? MaxSodium { get; set; }
}
```

## Prompt Engineering Strategy

### Core Prompt Templates

#### Meal Suggestion Prompt
```csharp
public class PromptTemplates
{
    public static string MealSuggestionPrompt = @"
You are an expert family meal planning assistant with knowledge of nutrition, cooking techniques, and cultural cuisines. 

FAMILY CONTEXT:
{familyProfiles}

MEAL REQUIREMENTS:
- Meal Type: {mealType}
- Target Date: {targetDate}
- Maximum Prep Time: {maxPrepTime} minutes
- Maximum Cook Time: {maxCookTime} minutes
- Budget: ${budget}
- Servings Needed: {servings}
- Occasion: {occasion}

AVAILABLE INGREDIENTS:
{availableIngredients}

RECENT MEALS (avoid repetition):
{recentMeals}

NUTRITIONAL GOALS:
{nutritionalGoals}

INSTRUCTIONS:
1. Consider ALL family members' preferences, restrictions, and allergies
2. Prioritize ingredients already available to minimize waste
3. Ensure meals fit within time and budget constraints
4. Provide 3 diverse meal suggestions with different cooking styles
5. Include ingredient substitutions for dietary restrictions
6. Rate each meal's family compatibility (1-10 scale)
7. Explain reasoning for each suggestion

RESPONSE FORMAT (JSON):
{{
  ""suggestions"": [
    {{
      ""name"": ""Meal Name"",
      ""description"": ""Brief appealing description"",
      ""familyFitScore"": 8,
      ""reasoning"": ""Why this meal works for the family"",
      ""prepTime"": 15,
      ""cookTime"": 25,
      ""difficulty"": ""Easy"",
      ""estimatedCost"": 12.50,
      ""cuisine"": ""Italian"",
      ""ingredients"": [
        {{
          ""name"": ""Chicken breast"",
          ""amount"": 1.5,
          ""unit"": ""lbs"",
          ""substitutions"": [""Tofu for vegetarians""]
        }}
      ],
      ""instructions"": [
        {{
          ""step"": 1,
          ""instruction"": ""Detailed step description"",
          ""timeEstimate"": 5
        }}
      ],
      ""nutritionalInfo"": {{
        ""calories"": 350,
        ""protein"": 25,
        ""carbs"": 20,
        ""fat"": 15,
        ""fiber"": 5,
        ""sodium"": 650
      }},
      ""tags"": [""quick"", ""healthy"", ""family-friendly""],
      ""allergenWarnings"": [""Contains dairy""],
      ""dietaryCompatibility"": [""Keto-friendly with modifications""]
    }}
  ]
}}
";

    public static string RecipePersonalizationPrompt = @"
You are tasked with personalizing a recipe for a specific family.

ORIGINAL RECIPE:
{originalRecipe}

FAMILY PROFILES:
{familyProfiles}

PERSONALIZATION REQUESTS:
- Adjust for dietary restrictions: {dietaryRestrictions}
- Modify spice level for: {spiceAdjustments}
- Substitute ingredients for allergies: {allergySubstitutions}
- Adjust serving size from {originalServings} to {targetServings}

Provide the personalized recipe with:
1. Modified ingredient list with quantities
2. Updated cooking instructions
3. Notes about substitutions made
4. Nutritional impact of changes
5. Tips for family members with specific needs
";

    public static string WeeklyMenuPrompt = @"
Create a complete weekly meal plan for a family.

FAMILY PROFILES:
{familyProfiles}

WEEK PARAMETERS:
- Start Date: {startDate}
- Budget: ${weeklyBudget}
- Meal Types: {mealTypes}
- Special Occasions: {specialOccasions}

CONSTRAINTS:
- Maximum prep time per meal: {maxPrepTime} minutes
- Grocery shopping frequency: {shoppingFrequency}
- Leftover utilization: {leftoverStrategy}

Provide:
1. 7-day meal plan with breakfast, lunch, dinner
2. Consolidated shopping list organized by store section
3. Meal prep recommendations
4. Cost breakdown
5. Nutritional balance analysis
6. Recipe scaling for family size
";
}
```

### Prompt Engineering Service

```csharp
public interface IPromptEngineering
{
    Task<string> BuildMealSuggestionPrompt(MealSuggestionRequest request);
    Task<string> BuildPersonalizationPrompt(Recipe recipe, List<FamilyPersona> family, PersonalizationRequest personalization);
    Task<string> BuildWeeklyMenuPrompt(WeeklyMenuRequest request);
}

public class PromptEngineeringService : IPromptEngineering
{
    public async Task<string> BuildMealSuggestionPrompt(MealSuggestionRequest request)
    {
        var familyProfiles = FormatFamilyProfiles(request.FamilyMembers);
        var availableIngredients = FormatAvailableIngredients(request.AvailableIngredients);
        var recentMeals = string.Join(", ", request.RecentMeals);
        var nutritionalGoals = FormatNutritionalGoals(request.NutritionalGoals);

        return PromptTemplates.MealSuggestionPrompt
            .Replace("{familyProfiles}", familyProfiles)
            .Replace("{mealType}", request.MealType.ToString())
            .Replace("{targetDate}", request.TargetDate.ToString("yyyy-MM-dd"))
            .Replace("{maxPrepTime}", request.MaxPrepTime.ToString())
            .Replace("{maxCookTime}", request.MaxCookTime.ToString())
            .Replace("{budget}", request.Budget.ToString("F2"))
            .Replace("{servings}", request.DesiredServings.ToString())
            .Replace("{occasion}", request.Occasion.ToString())
            .Replace("{availableIngredients}", availableIngredients)
            .Replace("{recentMeals}", recentMeals)
            .Replace("{nutritionalGoals}", nutritionalGoals);
    }

    private string FormatFamilyProfiles(List<FamilyPersona> familyMembers)
    {
        var profiles = familyMembers.Select(member => $@"
{member.Name} (Age: {member.Age}):
- Dietary Restrictions: {string.Join(", ", member.DietaryRestrictions)}
- Allergies: {string.Join(", ", member.Allergies)}
- Likes: {string.Join(", ", member.LikedIngredients)}
- Dislikes: {string.Join(", ", member.DislikedIngredients)}
- Cuisine Preferences: {string.Join(", ", member.CuisinePreferences)}
- Spice Tolerance: {member.SpiceToleranceLevel}/10
- Cooking Skill: {member.CookingSkill}
- Health Goals: {string.Join(", ", member.HealthGoals)}
");
        return string.Join("\n", profiles);
    }

    private string FormatAvailableIngredients(List<AvailableIngredient> ingredients)
    {
        if (!ingredients.Any()) return "None specified";
        
        var formatted = ingredients.Select(ing => 
            $"- {ing.Name}: {ing.Quantity} {ing.Unit}" + 
            (ing.ExpirationDate.HasValue ? $" (expires {ing.ExpirationDate:yyyy-MM-dd})" : ""));
        
        return string.Join("\n", formatted);
    }

    private string FormatNutritionalGoals(NutritionalGoals goals)
    {
        if (goals == null) return "No specific nutritional goals";

        var goalsList = new List<string>();
        if (goals.MaxCalories.HasValue) goalsList.Add($"Max Calories: {goals.MaxCalories}");
        if (goals.MinProtein.HasValue) goalsList.Add($"Min Protein: {goals.MinProtein}g");
        if (goals.MaxCarbs.HasValue) goalsList.Add($"Max Carbs: {goals.MaxCarbs}g");
        if (goals.MaxFat.HasValue) goalsList.Add($"Max Fat: {goals.MaxFat}g");
        if (goals.MinFiber.HasValue) goalsList.Add($"Min Fiber: {goals.MinFiber}g");
        if (goals.MaxSodium.HasValue) goalsList.Add($"Max Sodium: {goals.MaxSodium}mg");

        return string.Join(", ", goalsList);
    }
}
```

## Gemini AI Service Implementation

```csharp
public interface IGeminiAiService
{
    Task<List<MealSuggestion>> GetMealSuggestions(MealSuggestionRequest request);
    Task<PersonalizedRecipe> PersonalizeRecipe(Recipe recipe, List<FamilyPersona> family, PersonalizationRequest personalization);
    Task<WeeklyMenuPlan> GenerateWeeklyMenu(WeeklyMenuRequest request);
    Task<List<IngredientSubstitution>> GetIngredientSubstitutions(string ingredient, List<DietaryRestriction> restrictions);
}

public class GeminiAiService : IGeminiAiService
{
    private readonly PredictionServiceClient _predictionService;
    private readonly IPromptEngineering _promptEngineering;
    private readonly GoogleCloudOptions _options;
    private readonly AIOptions _aiOptions;
    private readonly ILogger<GeminiAiService> _logger;

    public GeminiAiService(
        IOptions<GoogleCloudOptions> options,
        IOptions<AIOptions> aiOptions,
        IPromptEngineering promptEngineering,
        ILogger<GeminiAiService> logger)
    {
        _options = options.Value;
        _aiOptions = aiOptions.Value;
        _promptEngineering = promptEngineering;
        _logger = logger;
        
        var credential = GoogleCredential.FromFile(_options.ServiceAccountKeyPath);
        _predictionService = new PredictionServiceClientBuilder
        {
            Credential = credential
        }.Build();
    }

    public async Task<List<MealSuggestion>> GetMealSuggestions(MealSuggestionRequest request)
    {
        try
        {
            var prompt = await _promptEngineering.BuildMealSuggestionPrompt(request);
            var response = await GenerateContent(prompt);
            
            var suggestions = JsonSerializer.Deserialize<MealSuggestionResponse>(response);
            return suggestions.Suggestions;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error generating meal suggestions for family");
            throw new AiServiceException("Failed to generate meal suggestions", ex);
        }
    }

    private async Task<string> GenerateContent(string prompt)
    {
        var projectLocation = ProjectLocationName.FromProjectLocation(_options.ProjectId, _options.Location);
        
        var generateContentRequest = new GenerateContentRequest
        {
            Model = $"{projectLocation}/publishers/google/models/{_options.Model}",
            Contents =
            {
                new Content
                {
                    Parts =
                    {
                        new Part { Text = prompt }
                    }
                }
            },
            GenerationConfig = new GenerationConfig
            {
                MaxOutputTokens = _aiOptions.MaxTokens,
                Temperature = (float)_aiOptions.Temperature,
                TopP = (float)_aiOptions.TopP,
                TopK = _aiOptions.TopK
            },
            SafetySettings =
            {
                new SafetySetting
                {
                    Category = HarmCategory.HateSpeech,
                    Threshold = ParseSafetyThreshold(_aiOptions.SafetySettings.HateSpeech)
                },
                new SafetySetting
                {
                    Category = HarmCategory.DangerousContent,
                    Threshold = ParseSafetyThreshold(_aiOptions.SafetySettings.DangerousContent)
                }
            }
        };

        var response = await _predictionService.GenerateContentAsync(generateContentRequest);
        
        if (response.Candidates?.Any() != true)
        {
            throw new AiServiceException("No content generated by AI service");
        }

        var content = response.Candidates.First().Content;
        return content.Parts.First().Text;
    }

    private SafetySetting.Types.HarmBlockThreshold ParseSafetyThreshold(string threshold)
    {
        return threshold.ToUpper() switch
        {
            "BLOCK_NONE" => SafetySetting.Types.HarmBlockThreshold.BlockNone,
            "BLOCK_LOW_AND_ABOVE" => SafetySetting.Types.HarmBlockThreshold.BlockLowAndAbove,
            "BLOCK_MEDIUM_AND_ABOVE" => SafetySetting.Types.HarmBlockThreshold.BlockMediumAndAbove,
            "BLOCK_ONLY_HIGH" => SafetySetting.Types.HarmBlockThreshold.BlockOnlyHigh,
            _ => SafetySetting.Types.HarmBlockThreshold.BlockMediumAndAbove
        };
    }
}
```

## Response Processing

### AI Response Models
```csharp
public class MealSuggestionResponse
{
    public List<MealSuggestion> Suggestions { get; set; } = new();
}

public class MealSuggestion
{
    public string Name { get; set; }
    public string Description { get; set; }
    public int FamilyFitScore { get; set; }
    public string Reasoning { get; set; }
    public int PrepTime { get; set; }
    public int CookTime { get; set; }
    public string Difficulty { get; set; }
    public decimal EstimatedCost { get; set; }
    public string Cuisine { get; set; }
    public List<RecipeIngredient> Ingredients { get; set; } = new();
    public List<CookingInstruction> Instructions { get; set; } = new();
    public NutritionalInformation NutritionalInfo { get; set; }
    public List<string> Tags { get; set; } = new();
    public List<string> AllergenWarnings { get; set; } = new();
    public List<string> DietaryCompatibility { get; set; } = new();
}

public class RecipeIngredient
{
    public string Name { get; set; }
    public decimal Amount { get; set; }
    public string Unit { get; set; }
    public List<string> Substitutions { get; set; } = new();
}

public class CookingInstruction
{
    public int Step { get; set; }
    public string Instruction { get; set; }
    public int TimeEstimate { get; set; }
}
```

## Fallback Strategies

### AI Service Resilience
```csharp
public class ResilientAiService : IGeminiAiService
{
    private readonly IGeminiAiService _primaryService;
    private readonly IFallbackRecipeService _fallbackService;
    private readonly ICircuitBreakerService _circuitBreaker;
    private readonly ILogger<ResilientAiService> _logger;

    public async Task<List<MealSuggestion>> GetMealSuggestions(MealSuggestionRequest request)
    {
        return await _circuitBreaker.ExecuteAsync(async () =>
        {
            try
            {
                return await _primaryService.GetMealSuggestions(request);
            }
            catch (AiServiceException ex) when (ex.IsRetryable)
            {
                _logger.LogWarning("AI service failed, falling back to recipe database");
                return await _fallbackService.GetSimilarRecipes(request);
            }
        });
    }
}
```

## Cost Optimization

### Token Management
```csharp
public class TokenOptimizationService
{
    public string OptimizePrompt(string prompt, int maxTokens)
    {
        // Implement prompt compression techniques
        // - Remove unnecessary whitespace
        // - Abbreviate common terms
        // - Prioritize most important context
        // - Use structured format hints
        
        return CompressPrompt(prompt, maxTokens);
    }

    public bool ShouldUseCache(MealSuggestionRequest request)
    {
        // Check if similar request was made recently
        // Return cached response for cost savings
        return HasRecentSimilarRequest(request);
    }
}
```

### Monitoring and Analytics
```csharp
public class AiUsageTracker
{
    public void TrackTokenUsage(string operation, int tokensUsed, decimal cost)
    {
        // Log token usage for cost analysis
        // Monitor request patterns
        // Identify optimization opportunities
    }

    public void TrackSuccessRate(string operation, bool success, string errorType = null)
    {
        // Monitor AI service reliability
        // Track fallback usage
        // Alert on degraded performance
    }
}
```

## Testing AI Integration

### Unit Tests
```csharp
[TestClass]
public class GeminiAiServiceTests
{
    [TestMethod]
    public async Task GetMealSuggestions_ValidRequest_ReturnsAppropriateRecipes()
    {
        // Arrange
        var request = CreateTestRequest();
        var service = CreateTestService();

        // Act
        var suggestions = await service.GetMealSuggestions(request);

        // Assert
        Assert.IsTrue(suggestions.Count <= 3);
        Assert.IsTrue(suggestions.All(s => s.FamilyFitScore > 0));
        Assert.IsTrue(suggestions.All(s => s.PrepTime <= request.MaxPrepTime));
    }

    [TestMethod]
    public async Task GetMealSuggestions_FamilyWithAllergies_ExcludesAllergens()
    {
        // Test allergy handling
        // Verify no suggested recipes contain allergens
    }
}
```

### Integration Tests
```csharp
[TestClass]
public class AiIntegrationTests
{
    [TestMethod]
    public async Task EndToEnd_MealSuggestion_ProducesValidRecipe()
    {
        // Test complete flow from request to parsed response
        // Verify AI service connectivity
        // Validate response format
    }
}
```

This comprehensive AI integration guide provides the foundation for implementing intelligent meal planning with Google Gemini AI in the MealMenu application.