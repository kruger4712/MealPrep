# AI Prompt Engineering Guide

## Overview
Comprehensive guide for designing, optimizing, and maintaining AI prompts for the MealPrep application's intelligent meal planning features using Google Gemini AI.

## Prompt Engineering Philosophy

### Core Principles
1. **Clarity**: Prompts should be unambiguous and specific
2. **Context**: Provide sufficient background for accurate responses
3. **Constraints**: Define clear boundaries and limitations
4. **Consistency**: Maintain standardized patterns across prompts
5. **Optimization**: Balance detail with token efficiency
6. **Testability**: Design prompts that can be validated and improved

### MealPrep-Specific Considerations
- **Family-Centric**: All prompts must consider family dynamics and preferences
- **Nutritional Awareness**: Include dietary restrictions and health goals
- **Cost Consciousness**: Factor in budget constraints and ingredient costs
- **Time Sensitivity**: Consider meal preparation and cooking time limitations
- **Cultural Sensitivity**: Respect dietary preferences and cultural considerations

## Prompt Architecture

### Standard Prompt Structure
```
[ROLE DEFINITION]
You are an expert family meal planning assistant...

[CONTEXT SECTION]
FAMILY PROFILE:
[Detailed family information]

[TASK DEFINITION]
[Clear instructions for what to generate]

[CONSTRAINTS SECTION]
[Specific limitations and requirements]

[OUTPUT FORMAT]
[Exact format specification with examples]

[GUIDELINES SECTION]
[Important rules and best practices]
```

### Prompt Components

#### 1. Role Definition
```
You are an expert family meal planning assistant with deep knowledge of:
- Nutritional science and dietary requirements
- Culinary techniques and cooking methods
- Family dynamics and meal preferences
- Budget-conscious meal planning
- Cultural and dietary considerations
- Food safety and storage practices
```

#### 2. Context Building
```csharp
// Family Context Builder
public class FamilyContextBuilder
{
    public string BuildContext(FamilyPersona persona)
    {
        var context = new StringBuilder();
        
        context.AppendLine($"FAMILY PROFILE:");
        context.AppendLine($"Size: {persona.Members.Count} members");
        context.AppendLine($"Age Range: {persona.AgeRange}");
        context.AppendLine($"Location: {persona.Location}");
        
        foreach (var member in persona.Members)
        {
            context.AppendLine($"\n{member.Name} ({member.Age} years old):");
            context.AppendLine($"  - Role: {member.Relationship}");
            context.AppendLine($"  - Dietary Restrictions: {string.Join(", ", member.DietaryRestrictions)}");
            
            if (member.Allergies.Any())
                context.AppendLine($"  - CRITICAL ALLERGIES: {string.Join(", ", member.Allergies)}");
                
            context.AppendLine($"  - Preferred Foods: {string.Join(", ", member.LikedIngredients.Take(5))}");
            context.AppendLine($"  - Dislikes: {string.Join(", ", member.DislikedIngredients.Take(3))}");
            context.AppendLine($"  - Spice Tolerance: {member.SpiceToleranceLevel}/10");
            context.AppendLine($"  - Cooking Skill: {member.CookingSkillLevel}");
            
            if (member.HealthGoals.Any())
                context.AppendLine($"  - Health Goals: {string.Join(", ", member.HealthGoals)}");
        }
        
        return context.ToString();
    }
}
```

#### 3. Constraint Definition
```
MANDATORY CONSTRAINTS:
- NEVER suggest ingredients that contain family allergens
- Stay within budget: ${budget:F2} maximum
- Respect preparation time: {maxPrepTime} minutes maximum
- Include only ingredients from approved dietary restrictions
- Consider cooking skill level of primary cook

PREFERENCE CONSTRAINTS:
- Prioritize ingredients family members love
- Minimize ingredients family members dislike
- Match spice tolerance levels
- Align with stated health goals
- Consider seasonal ingredient availability
```

## Prompt Templates

### 1. Meal Suggestion Prompt
```csharp
public class MealSuggestionPrompt : IPromptTemplate
{
    public string Generate(MealSuggestionRequest request, FamilyPersona persona)
    {
        return $@"
You are an expert family meal planning assistant specializing in personalized, budget-conscious meal suggestions.

{BuildFamilyContext(persona)}

MEAL REQUEST:
- Meal Type: {request.MealType}
- Target Date: {request.TargetDate:yyyy-MM-dd}
- Budget Limit: ${request.Budget:F2}
- Max Prep Time: {request.MaxPrepTime} minutes
- Max Cook Time: {request.MaxCookTime} minutes
- Servings Needed: {request.Servings}

AVAILABLE INGREDIENTS:
{BuildAvailableIngredientsSection(request.AvailableIngredients)}

RECENT MEALS (avoid repetition):
{string.Join("", "", request.RecentMeals)}

SPECIAL CONSIDERATIONS:
{BuildSpecialConsiderations(request)}

Generate exactly 3 meal suggestions that perfectly match this family's profile. Each suggestion must:
1. Respect ALL dietary restrictions and allergies
2. Stay within the specified budget
3. Match the time constraints
4. Consider family preferences and dislikes
5. Include realistic ingredient costs and preparation times

Respond in this EXACT JSON format:
{{
  ""suggestions"": [
    {{
      ""name"": ""Recipe Name"",
      ""description"": ""Appetizing 2-sentence description"",
      ""familyFitReasoning"": ""Specific explanation of why this works for this family"",
      ""prepTime"": number,
      ""cookTime"": number,
      ""servings"": {request.Servings},
      ""estimatedCost"": number,
      ""difficulty"": ""Easy|Medium|Hard"",
      ""cuisine"": ""Cuisine Type"",
      ""familyFitScore"": number (1-10),
      ""ingredients"": [
        {{
          ""name"": ""Ingredient name"",
          ""quantity"": number,
          ""unit"": ""measurement unit"",
          ""estimatedCost"": number,
          ""isAvailable"": boolean,
          ""substitutions"": [""alternative1"", ""alternative2""]
        }}
      ],
      ""instructions"": [
        ""Step 1: Clear instruction"",
        ""Step 2: Clear instruction""
      ],
      ""nutritionalInfo"": {{
        ""calories"": number,
        ""protein"": number,
        ""carbs"": number,
        ""fat"": number,
        ""fiber"": number,
        ""sodium"": number
      }},
      ""tags"": [""family-friendly"", ""quick-prep"", ""budget-friendly""],
      ""allergenWarnings"": [""Contains: allergen""],
      ""personalizations"": {{
        ""{persona.Members[0].Name}"": ""How this meal is perfect for them"",
        ""{persona.Members[1].Name}"": ""How this meal accommodates their needs""
      }},
      ""cookingTips"": [
        ""Helpful tip for better results"",
        ""Time-saving suggestion""
      ],
      ""makeAheadOptions"": ""How to prep in advance"",
      ""leftoversIdeas"": ""Creative ways to use leftovers""
    }}
  ]
}}

CRITICAL GUIDELINES:
- NEVER include allergens: {string.Join("", "", persona.GetAllAllergens())}
- Prioritize ingredients the family loves: {string.Join("", "", persona.GetFavoriteIngredients())}
- Minimize disliked ingredients: {string.Join("", "", persona.GetDislikedIngredients())}
- Match spice tolerance: Average family tolerance is {persona.AverageSpiceToleranceLevel}/10
- Consider cooking skill: Primary cook skill level is {persona.PrimaryCookSkillLevel}
- Include cost breakdown that adds up correctly
- Ensure nutritional info is realistic for ingredients used
- Make personalization notes specific to each family member's preferences

Respond with ONLY the JSON, no additional text.
";
    }
}
```

### 2. Recipe Personalization Prompt
```csharp
public class RecipePersonalizationPrompt : IPromptTemplate
{
    public string Generate(Recipe originalRecipe, FamilyPersona persona, PersonalizationRequest request)
    {
        return $@"
You are an expert chef and family nutrition specialist. Transform the following recipe to perfectly match this family's needs.

{BuildFamilyContext(persona)}

ORIGINAL RECIPE:
Name: {originalRecipe.Name}
Servings: {originalRecipe.Servings}
Prep Time: {originalRecipe.PrepTime} minutes
Cook Time: {originalRecipe.CookTime} minutes

Ingredients:
{BuildIngredientsList(originalRecipe.Ingredients)}

Instructions:
{BuildInstructionsList(originalRecipe.Instructions)}

PERSONALIZATION REQUIREMENTS:
- Adjust servings from {originalRecipe.Servings} to {request.TargetServings}
- Dietary modifications: {string.Join("", "", request.DietaryModifications)}
- Substitute ingredients: {BuildSubstitutionRequests(request.IngredientSubstitutions)}
- Spice level: {request.SpiceLevel}
- Health focus: {request.HealthFocus}
- Time constraints: {request.MaxPrepTime} minutes prep, {request.MaxCookTime} minutes cook

PERSONALIZATION GOALS:
{BuildPersonalizationGoals(persona, request)}

Create a personalized version that:
1. Maintains the essence and flavor profile of the original
2. Accommodates ALL family dietary restrictions and allergies
3. Adjusts quantities proportionally for new serving size
4. Replaces ingredients according to substitution requests
5. Modifies spice levels to match family tolerance
6. Optimizes nutrition for family health goals
7. Simplifies preparation if requested

Respond in this JSON format:
{{
  ""personalizedRecipe"": {{
    ""name"": ""Personalized Recipe Name"",
    ""description"": ""How this version is tailored for the family"",
    ""servings"": {request.TargetServings},
    ""prepTime"": number,
    ""cookTime"": number,
    ""difficulty"": ""Easy|Medium|Hard"",
    ""modifications"": [
      ""List of key changes made"",
      ""Substitutions performed"",
      ""Adjustments for family preferences""
    ],
    ""ingredients"": [
      {{
        ""name"": ""Ingredient name"",
        ""quantity"": number,
        ""unit"": ""unit"",
        ""originalIngredient"": ""Original ingredient if substituted"",
        ""substitutionReason"": ""Why this substitution was made""
      }}
    ],
    ""instructions"": [
      ""Modified step 1"",
      ""Modified step 2""
    ],
    ""nutritionalChanges"": {{
      ""calories"": {{""original"": number, ""modified"": number, ""change"": number}},
      ""protein"": {{""original"": number, ""modified"": number, ""change"": number}},
      ""sodium"": {{""original"": number, ""modified"": number, ""change"": number}}
    }},
    ""familyBenefits"": {{
      ""{persona.Members[0].Name}"": ""How this version benefits them"",
      ""{persona.Members[1].Name}"": ""How this version accommodates their needs""
    }},
    ""cookingTips"": [
      ""Tips specific to the modifications made"",
      ""How to handle substituted ingredients""
    ],
    ""confidenceScore"": number (1-10)
  }}
}}

MODIFICATION GUIDELINES:
- Maintain flavor balance when substituting ingredients
- Scale cooking times appropriately for serving size changes
- Consider how substitutions affect cooking methods
- Preserve cultural authenticity when possible
- Ensure nutritional adequacy for the family
- Simplify without compromising food safety

Respond with ONLY the JSON, no additional text.
";
    }
}
```

### 3. Weekly Menu Planning Prompt
```csharp
public class WeeklyMenuPrompt : IPromptTemplate
{
    public string Generate(WeeklyMenuRequest request, FamilyPersona persona)
    {
        return $@"
You are an expert family meal planning coordinator specializing in comprehensive weekly menu design.

{BuildFamilyContext(persona)}

WEEKLY MENU REQUEST:
- Week Starting: {request.StartDate:yyyy-MM-dd}
- Meal Types: {string.Join("", "", request.MealTypes)}
- Weekly Budget: ${request.WeeklyBudget:F2}
- Shopping Preference: {request.ShoppingFrequency}
- Meal Prep Day: {request.MealPrepDay}
- Max Daily Cooking Time: {request.MaxDailyCookingTime} minutes

SCHEDULE CONSTRAINTS:
{BuildScheduleConstraints(request.Constraints)}

VARIETY PREFERENCES:
- Cuisine Variety: {request.CuisineVariety}/10
- Cooking Method Variety: {request.CookingMethodVariety}/10
- Ingredient Overlap: {request.AllowIngredientReuse ? ""Acceptable"" : ""Minimize""}

SPECIAL OCCASIONS:
{BuildSpecialOccasions(request.SpecialOccasions)}

Create a complete 7-day meal plan that optimizes for nutrition, cost, time, and family satisfaction.

Design a weekly menu that:
1. Stays within the total budget while distributing costs evenly
2. Maximizes ingredient reuse to minimize waste
3. Balances cooking complexity throughout the week
4. Incorporates variety in cuisines and cooking methods
5. Schedules more complex meals on less busy days
6. Plans strategic leftovers and meal prep opportunities
7. Accommodates all family dietary restrictions
8. Includes seasonal ingredients when possible

Respond in this JSON format:
{{
  ""weeklyMenu"": {{
    ""weekOf"": ""{request.StartDate:yyyy-MM-dd}"",
    ""totalEstimatedCost"": number,
    ""totalPrepTime"": number,
    ""varietyScore"": number,
    ""nutritionScore"": number,
    ""dailyMenus"": [
      {{
        ""date"": ""2024-01-22"",
        ""dayOfWeek"": ""Monday"",
        ""theme"": ""Easy Monday"",
        ""meals"": {{
          ""breakfast"": {{
            ""recipeName"": ""Recipe Name"",
            ""prepTime"": number,
            ""cost"": number,
            ""familyFitScore"": number,
            ""mealPrepNotes"": ""Can be prepped Sunday night""
          }},
          ""lunch"": {{}},
          ""dinner"": {{}}
        }},
        ""dailyCost"": number,
        ""dailyPrepTime"": number,
        ""dailyNotes"": ""Strategic notes for the day""
      }}
    ],
    ""ingredientShoppingList"": {{
      ""totalCost"": number,
      ""categoryBreakdown"": {{
        ""proteins"": number,
        ""vegetables"": number,
        ""pantryItems"": number
      }},
      ""ingredientReuse"": [
        {{
          ""ingredient"": ""Chicken breast"",
          ""usedIn"": [""Monday dinner"", ""Wednesday lunch""],
          ""totalQuantity"": ""2 lbs"",
          ""costSavings"": number
        }}
      ]
    }},
    ""mealPrepPlan"": {{
      ""prepDay"": ""{request.MealPrepDay}"",
      ""totalPrepTime"": number,
      ""tasks"": [
        {{
          ""task"": ""Wash and chop vegetables for week"",
          ""timeEstimate"": number,
          ""benefitsMeals"": [""Monday dinner"", ""Tuesday lunch""]
        }}
      ]
    }},
    ""nutritionalSummary"": {{
      ""dailyAverageCalories"": number,
      ""weeklyNutritionGoals"": {{
        ""proteinTarget"": ""Achieved"",
        ""vegetableServings"": ""Above recommended"",
        ""wholegrains"": ""Met""
      }}
    }},
    ""familyPersonalization"": {{
      ""{persona.Members[0].Name}"": [""Monday lunch perfect for lunch meeting""],
      ""{persona.Members[1].Name}"": [""Vegetarian options provided Tuesday and Thursday""]
    }}
  }},
  ""alternatives"": [
    {{
      ""day"": ""Wednesday"",
      ""meal"": ""dinner"",
      ""alternative"": ""Slow cooker option for busy day"",
      ""reason"": ""Backup plan for schedule conflicts""
    }}
  ],
  ""weeklyTips"": [
    ""Prep vegetables on Sunday to save 30 minutes during the week"",
    ""Cook extra rice Monday for Wednesday fried rice""
  ]
}}

WEEKLY PLANNING RULES:
- Distribute cooking complexity: Save elaborate meals for weekends
- Strategic leftovers: Plan intentional leftovers for busy days
- Ingredient efficiency: Use versatile ingredients across multiple meals
- Prep optimization: Group similar prep tasks on meal prep day
- Budget pacing: Front-load expensive proteins, use cheaper options later
- Nutrition balance: Ensure each day meets basic nutritional needs
- Family preferences: Rotate favorite meals throughout the week
- Seasonal considerations: Incorporate fresh, seasonal ingredients

Respond with ONLY the JSON, no additional text.
";
    }
}
```

## Prompt Optimization Strategies

### 1. Token Efficiency
```csharp
public class PromptOptimizer
{
    public string OptimizePrompt(string originalPrompt, int maxTokens = 4000)
    {
        // Remove redundant phrases
        var optimized = RemoveRedundancy(originalPrompt);
        
        // Compress verbose instructions
        optimized = CompressInstructions(optimized);
        
        // Use abbreviations for common terms
        optimized = ApplyAbbreviations(optimized);
        
        // Validate token count
        if (EstimateTokenCount(optimized) > maxTokens)
        {
            optimized = TruncateIntelligently(optimized, maxTokens);
        }
        
        return optimized;
    }

    private string ApplyAbbreviations(string prompt)
    {
        var abbreviations = new Dictionary<string, string>
        {
            ["dietary restrictions"] = "diet restrict",
            ["ingredients"] = "ingred",
            ["preparation time"] = "prep time",
            ["estimated cost"] = "est cost",
            ["family members"] = "family"
        };

        foreach (var abbrev in abbreviations)
        {
            prompt = prompt.Replace(abbrev.Key, abbrev.Value);
        }

        return prompt;
    }
}
```

### 2. Response Quality Validation
```csharp
public class ResponseValidator
{
    public ValidationResult ValidateResponse(string aiResponse, PromptType promptType)
    {
        var result = new ValidationResult();
        
        try
        {
            // Parse JSON response
            var parsed = JsonSerializer.Deserialize<dynamic>(aiResponse);
            
            // Validate structure
            result.AddValidation("JSON Format", ValidateJsonStructure(parsed, promptType));
            
            // Validate content requirements
            result.AddValidation("Required Fields", ValidateRequiredFields(parsed, promptType));
            
            // Validate data types
            result.AddValidation("Data Types", ValidateDataTypes(parsed, promptType));
            
            // Validate business rules
            result.AddValidation("Business Rules", ValidateBusinessRules(parsed, promptType));
            
        }
        catch (JsonException)
        {
            result.AddError("Invalid JSON format in AI response");
        }
        
        return result;
    }

    private bool ValidateBusinessRules(dynamic response, PromptType promptType)
    {
        switch (promptType)
        {
            case PromptType.MealSuggestion:
                return ValidateMealSuggestionRules(response);
            case PromptType.WeeklyMenu:
                return ValidateWeeklyMenuRules(response);
            default:
                return true;
        }
    }

    private bool ValidateMealSuggestionRules(dynamic response)
    {
        // Validate family fit scores are 1-10
        // Validate costs are positive numbers
        // Validate time estimates are reasonable
        // Validate ingredient quantities are positive
        return true; // Simplified for example
    }
}
```

### 3. A/B Testing Framework
```csharp
public class PromptABTester
{
    public async Task<ABTestResult> RunPromptTest(
        string promptA, 
        string promptB, 
        List<TestCase> testCases)
    {
        var resultsA = new List<TestResult>();
        var resultsB = new List<TestResult>();

        foreach (var testCase in testCases)
        {
            // Test Prompt A
            var responseA = await _aiService.GenerateResponseAsync(promptA, testCase.Input);
            resultsA.Add(EvaluateResponse(responseA, testCase.ExpectedOutput));

            // Test Prompt B
            var responseB = await _aiService.GenerateResponseAsync(promptB, testCase.Input);
            resultsB.Add(EvaluateResponse(responseB, testCase.ExpectedOutput));
        }

        return new ABTestResult
        {
            PromptAResults = AnalyzeResults(resultsA),
            PromptBResults = AnalyzeResults(resultsB),
            Winner = DetermineWinner(resultsA, resultsB),
            ConfidenceLevel = CalculateConfidence(resultsA, resultsB)
        };
    }
}
```

## Prompt Versioning and Management

### Version Control Strategy
```csharp
public class PromptVersionManager
{
    public class PromptVersion
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public string Content { get; set; }
        public string Version { get; set; }
        public DateTime CreatedAt { get; set; }
        public string CreatedBy { get; set; }
        public PromptMetrics Metrics { get; set; }
        public bool IsActive { get; set; }
    }

    public async Task<string> GetActivePrompt(PromptType type, string variant = "default")
    {
        var activePrompt = await _repository.GetActivePromptAsync(type, variant);
        
        if (activePrompt == null)
        {
            throw new PromptNotFoundException($"No active prompt found for {type}:{variant}");
        }

        return activePrompt.Content;
    }

    public async Task<PromptVersion> CreateNewVersion(
        PromptType type, 
        string content, 
        string description)
    {
        var newVersion = new PromptVersion
        {
            Id = Guid.NewGuid().ToString(),
            Name = $"{type}_v{GetNextVersionNumber(type)}",
            Content = content,
            Version = GetNextVersionNumber(type),
            CreatedAt = DateTime.UtcNow,
            CreatedBy = GetCurrentUser(),
            IsActive = false
        };

        await _repository.SavePromptVersionAsync(newVersion);
        return newVersion;
    }
}
```

### Performance Monitoring
```csharp
public class PromptPerformanceMonitor
{
    public async Task TrackPromptPerformance(PromptExecutionContext context)
    {
        var metrics = new PromptMetrics
        {
            PromptId = context.PromptId,
            ExecutionTime = context.ExecutionTime,
            TokensUsed = context.TokensUsed,
            Cost = context.Cost,
            ResponseQuality = await EvaluateResponseQuality(context.Response),
            UserSatisfaction = context.UserFeedback?.Satisfaction,
            Timestamp = DateTime.UtcNow
        };

        await _metricsRepository.SaveMetricsAsync(metrics);
        
        // Check if performance has degraded
        await CheckPerformanceThresholds(context.PromptId, metrics);
    }

    private async Task CheckPerformanceThresholds(string promptId, PromptMetrics metrics)
    {
        var recentMetrics = await _metricsRepository.GetRecentMetricsAsync(promptId, TimeSpan.FromDays(7));
        
        var avgQuality = recentMetrics.Average(m => m.ResponseQuality);
        var avgExecutionTime = recentMetrics.Average(m => m.ExecutionTime.TotalMilliseconds);
        
        if (avgQuality < 0.8 || avgExecutionTime > 5000)
        {
            await _alertService.SendPromptPerformanceAlertAsync(promptId, metrics);
        }
    }
}
```

## Best Practices and Guidelines

### 1. Prompt Design Principles
- **Be Specific**: Vague prompts lead to inconsistent results
- **Use Examples**: Include examples of desired output format
- **Set Clear Boundaries**: Define what the AI should not do
- **Iterate and Test**: Continuously improve based on results
- **Monitor Performance**: Track quality metrics over time

### 2. Common Pitfalls to Avoid
- **Over-prompting**: Too much detail can confuse the AI
- **Under-constraining**: Too few constraints lead to irrelevant responses
- **Inconsistent Format**: Varying output formats complicate parsing
- **Ignoring Context**: Not providing sufficient family context
- **Static Prompts**: Not adapting prompts based on performance data

### 3. Testing and Validation
- **Unit Test Prompts**: Test individual prompt components
- **Integration Testing**: Test prompts with real family data
- **Performance Testing**: Monitor response times and quality
- **User Acceptance Testing**: Validate with actual families
- **A/B Testing**: Compare prompt variations systematically

This comprehensive prompt engineering guide ensures consistent, high-quality AI responses for the MealPrep application's intelligent meal planning features.

*This guide should be updated as prompt engineering techniques evolve and new patterns emerge.*