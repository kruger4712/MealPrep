# Family Persona Modeling

## Overview
Comprehensive guide for modeling family personas in the MealPrep application to enable highly personalized AI-driven meal planning and recipe suggestions.

## Persona Architecture

### Core Persona Components
```csharp
public class FamilyPersona
{
    public string Id { get; set; }
    public string FamilyId { get; set; }
    public List<PersonaMember> Members { get; set; } = new();
    public DateTime CreatedAt { get; set; }
    public DateTime LastUpdated { get; set; }
    public PersonaMetadata Metadata { get; set; }
    
    // Computed Properties
    public string AgeRange => CalculateAgeRange();
    public List<string> CommonDietaryRestrictions => GetCommonDietaryRestrictions();
    public List<string> CriticalAllergens => GetCriticalAllergens();
    public decimal AverageSpiceToleranceLevel => CalculateAverageSpiceTolerance();
    public string PrimaryCookSkillLevel => GetPrimaryCookSkillLevel();
    public List<string> FavoriteIngredients => GetFavoriteIngredients();
    public List<string> DislikedIngredients => GetDislikedIngredients();
    public List<string> PreferredCuisines => GetPreferredCuisines();
}

public class PersonaMember
{
    // Basic Demographics
    public int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
    public string Relationship { get; set; }
    public string Gender { get; set; }
    public string ActivityLevel { get; set; }
    
    // Dietary Profile
    public List<DietaryRestriction> DietaryRestrictions { get; set; } = new();
    public List<string> Allergies { get; set; } = new();
    public List<string> LikedIngredients { get; set; } = new();
    public List<string> DislikedIngredients { get; set; } = new();
    public List<string> CuisinePreferences { get; set; } = new();
    public int SpiceToleranceLevel { get; set; } = 5; // 1-10 scale
    
    // Health and Nutrition
    public List<string> HealthGoals { get; set; } = new();
    public NutritionalNeeds NutritionalNeeds { get; set; }
    public List<string> MedicalConditions { get; set; } = new();
    
    // Lifestyle and Preferences
    public string CookingSkillLevel { get; set; }
    public int PreferredCookingTime { get; set; }
    public List<string> KitchenEquipment { get; set; } = new();
    public MealTimingPreferences MealTiming { get; set; }
    
    // Behavioral Data
    public PersonaBehaviorProfile BehaviorProfile { get; set; }
}
```

### Behavioral Profiling
```csharp
public class PersonaBehaviorProfile
{
    // Meal Rating History
    public List<MealRating> MealRatings { get; set; } = new();
    public Dictionary<string, decimal> IngredientPreferenceScores { get; set; } = new();
    public Dictionary<string, decimal> CuisinePreferenceScores { get; set; } = new();
    
    // Temporal Patterns
    public Dictionary<DayOfWeek, MealTypePreferences> WeeklyPatterns { get; set; } = new();
    public Dictionary<string, List<string>> SeasonalPreferences { get; set; } = new();
    
    // Social Patterns
    public decimal FamilyInfluenceScore { get; set; }
    public List<string> MealCompromises { get; set; } = new();
    
    // Learning Metadata
    public int TotalMealsEvaluated { get; set; }
    public DateTime LastLearningUpdate { get; set; }
    public decimal PersonaConfidenceScore { get; set; }
}

public class MealRating
{
    public string RecipeId { get; set; }
    public string RecipeName { get; set; }
    public int Rating { get; set; } // 1-5 scale
    public DateTime RatedAt { get; set; }
    public string MealType { get; set; }
    public List<string> PositiveFeedback { get; set; } = new();
    public List<string> NegativeFeedback { get; set; } = new();
    public string Context { get; set; } // "family dinner", "quick lunch", etc.
}
```

## Persona Generation Strategies

### 1. Initial Persona Creation
```csharp
public class PersonaGenerator : IPersonaGenerator
{
    public async Task<FamilyPersona> CreateInitialPersonaAsync(
        List<FamilyMember> familyMembers,
        InitialPreferenceSurvey survey)
    {
        var persona = new FamilyPersona
        {
            Id = Guid.NewGuid().ToString(),
            FamilyId = familyMembers.First().FamilyId,
            CreatedAt = DateTime.UtcNow,
            LastUpdated = DateTime.UtcNow
        };

        foreach (var member in familyMembers)
        {
            var personaMember = await CreatePersonaMemberAsync(member, survey);
            persona.Members.Add(personaMember);
        }

        // Calculate family-level preferences
        await CalculateFamilyDynamicsAsync(persona);
        
        // Initialize behavioral profiles
        await InitializeBehaviorProfilesAsync(persona);
        
        return persona;
    }

    private async Task<PersonaMember> CreatePersonaMemberAsync(
        FamilyMember member, 
        InitialPreferenceSurvey survey)
    {
        var personaMember = new PersonaMember
        {
            Id = member.Id,
            Name = member.Name,
            Age = member.Age,
            Relationship = member.Relationship,
            DietaryRestrictions = member.DietaryRestrictions,
            Allergies = member.Allergies,
            SpiceToleranceLevel = member.SpiceToleranceLevel,
            CookingSkillLevel = member.CookingSkillLevel ?? DetermineCookingSkillFromAge(member.Age)
        };

        // Enhance with survey data
        if (survey.MemberPreferences.ContainsKey(member.Id))
        {
            var preferences = survey.MemberPreferences[member.Id];
            personaMember.LikedIngredients = preferences.LikedIngredients;
            personaMember.DislikedIngredients = preferences.DislikedIngredients;
            personaMember.CuisinePreferences = preferences.CuisinePreferences;
            personaMember.HealthGoals = preferences.HealthGoals;
        }

        // Calculate nutritional needs based on demographics
        personaMember.NutritionalNeeds = await CalculateNutritionalNeedsAsync(personaMember);
        
        // Initialize behavior profile
        personaMember.BehaviorProfile = new PersonaBehaviorProfile
        {
            PersonaConfidenceScore = 0.3m, // Low confidence initially
            LastLearningUpdate = DateTime.UtcNow
        };

        return personaMember;
    }
}
```

### 2. Adaptive Learning System
```csharp
public class PersonaLearningEngine : IPersonaLearningEngine
{
    public async Task UpdatePersonaFromFeedbackAsync(
        FamilyPersona persona,
        MealFeedback feedback)
    {
        // Update individual member preferences
        await UpdateMemberPreferencesAsync(persona, feedback);
        
        // Update family dynamics
        await UpdateFamilyDynamicsAsync(persona, feedback);
        
        // Recalculate confidence scores
        await RecalculateConfidenceScoresAsync(persona);
        
        // Update timestamp
        persona.LastUpdated = DateTime.UtcNow;
        
        await _personaRepository.UpdateAsync(persona);
    }

    private async Task UpdateMemberPreferencesAsync(
        FamilyPersona persona,
        MealFeedback feedback)
    {
        foreach (var memberFeedback in feedback.MemberFeedbacks)
        {
            var member = persona.Members.FirstOrDefault(m => m.Id == memberFeedback.MemberId);
            if (member == null) continue;

            var rating = new MealRating
            {
                RecipeId = feedback.RecipeId,
                RecipeName = feedback.RecipeName,
                Rating = memberFeedback.Rating,
                RatedAt = DateTime.UtcNow,
                MealType = feedback.MealType,
                PositiveFeedback = memberFeedback.PositiveComments,
                NegativeFeedback = memberFeedback.NegativeComments,
                Context = feedback.Context
            };

            member.BehaviorProfile.MealRatings.Add(rating);
            member.BehaviorProfile.TotalMealsEvaluated++;

            // Update ingredient preferences based on rating
            await UpdateIngredientPreferencesAsync(member, feedback.Recipe, memberFeedback.Rating);
            
            // Update cuisine preferences
            await UpdateCuisinePreferencesAsync(member, feedback.Recipe.Cuisine, memberFeedback.Rating);
        }
    }

    private async Task UpdateIngredientPreferencesAsync(
        PersonaMember member,
        Recipe recipe,
        int rating)
    {
        foreach (var ingredient in recipe.Ingredients)
        {
            var currentScore = member.BehaviorProfile.IngredientPreferenceScores
                .GetValueOrDefault(ingredient.Name, 5.0m); // Neutral starting point

            // Apply learning rate based on confidence
            var learningRate = CalculateLearningRate(member.BehaviorProfile.PersonaConfidenceScore);
            
            // Update score using weighted average
            var newScore = CalculateUpdatedPreferenceScore(currentScore, rating, learningRate);
            
            member.BehaviorProfile.IngredientPreferenceScores[ingredient.Name] = newScore;

            // Update explicit like/dislike lists if score is extreme
            await UpdateExplicitPreferencesAsync(member, ingredient.Name, newScore);
        }
    }

    private decimal CalculateUpdatedPreferenceScore(decimal currentScore, int rating, decimal learningRate)
    {
        // Convert 1-5 rating to 1-10 preference score
        var ratingAsScore = rating * 2m;
        
        // Weighted average with learning rate
        var newScore = (currentScore * (1 - learningRate)) + (ratingAsScore * learningRate);
        
        // Bound between 1-10
        return Math.Max(1m, Math.Min(10m, newScore));
    }

    private decimal CalculateLearningRate(decimal confidenceScore)
    {
        // Higher confidence = lower learning rate (more stable)
        // Lower confidence = higher learning rate (more adaptive)
        return Math.Max(0.05m, Math.Min(0.3m, 1m - confidenceScore));
    }
}
```

### 3. Family Dynamics Modeling
```csharp
public class FamilyDynamicsAnalyzer : IFamilyDynamicsAnalyzer
{
    public async Task<FamilyDynamics> AnalyzeFamilyDynamicsAsync(FamilyPersona persona)
    {
        var dynamics = new FamilyDynamics
        {
            FamilyId = persona.FamilyId,
            AnalyzedAt = DateTime.UtcNow
        };

        // Analyze decision-making patterns
        dynamics.DecisionMaker = await IdentifyPrimaryDecisionMakerAsync(persona);
        dynamics.InfluenceHierarchy = await CalculateInfluenceHierarchyAsync(persona);
        
        // Analyze compromise patterns
        dynamics.CommonCompromises = await IdentifyCommonCompromisesAsync(persona);
        dynamics.ConflictAreas = await IdentifyConflictAreasAsync(persona);
        
        // Analyze meal planning patterns
        dynamics.MealPlanningStyle = await DetermineMealPlanningStyleAsync(persona);
        dynamics.CookingResponsibilities = await AnalyzeCookingResponsibilitiesAsync(persona);
        
        return dynamics;
    }

    private async Task<string> IdentifyPrimaryDecisionMakerAsync(FamilyPersona persona)
    {
        // Analyze historical meal selections and feedback patterns
        var memberInfluenceScores = new Dictionary<int, decimal>();

        foreach (var member in persona.Members)
        {
            var influenceScore = 0m;
            
            // Factor 1: Cooking skill and involvement
            influenceScore += GetCookingInfluenceScore(member);
            
            // Factor 2: Age and relationship role
            influenceScore += GetRoleInfluenceScore(member);
            
            // Factor 3: Historical meal selection success
            influenceScore += await GetMealSelectionSuccessScore(member);
            
            memberInfluenceScores[member.Id] = influenceScore;
        }

        var primaryDecisionMaker = memberInfluenceScores
            .OrderByDescending(kvp => kvp.Value)
            .First();

        return persona.Members.First(m => m.Id == primaryDecisionMaker.Key).Name;
    }

    private async Task<List<string>> IdentifyCommonCompromisesAsync(FamilyPersona persona)
    {
        var compromises = new List<string>();

        // Analyze conflicting preferences
        var allergenConflicts = AnalyzeAllergenConflicts(persona);
        var spiceToleranceConflicts = AnalyzeSpiceToleranceConflicts(persona);
        var cuisinePreferenceConflicts = AnalyzeCuisineConflicts(persona);
        var dietaryRestrictionConflicts = AnalyzeDietaryRestrictionConflicts(persona);

        if (allergenConflicts.Any())
            compromises.Add($"Avoid allergens: {string.Join(", ", allergenConflicts)}");

        if (spiceToleranceConflicts.SpiceRange > 5)
            compromises.Add($"Use mild spices to accommodate sensitive members");

        if (cuisinePreferenceConflicts.Any())
            compromises.Add($"Rotate between preferred cuisines: {string.Join(", ", cuisinePreferenceConflicts)}");

        if (dietaryRestrictionConflicts.Any())
            compromises.Add($"Accommodate dietary restrictions: {string.Join(", ", dietaryRestrictionConflicts)}");

        return compromises;
    }
}
```

## AI Prompt Integration

### Persona-Optimized Prompt Generation
```csharp
public class PersonaPromptBuilder : IPersonaPromptBuilder
{
    public string BuildPersonaContext(FamilyPersona persona)
    {
        var context = new StringBuilder();
        
        context.AppendLine("FAMILY PERSONA PROFILE:");
        context.AppendLine($"Family ID: {persona.FamilyId}");
        context.AppendLine($"Family Size: {persona.Members.Count} members");
        context.AppendLine($"Age Range: {persona.AgeRange}");
        context.AppendLine($"Persona Confidence: {GetOverallConfidenceScore(persona):P0}");
        context.AppendLine($"Last Updated: {persona.LastUpdated:yyyy-MM-dd}");
        
        context.AppendLine("\nFAMILY DYNAMICS:");
        context.AppendLine($"Primary Decision Maker: {await GetPrimaryDecisionMaker(persona)}");
        context.AppendLine($"Cooking Skill Range: {GetCookingSkillRange(persona)}");
        context.AppendLine($"Average Spice Tolerance: {persona.AverageSpiceToleranceLevel:F1}/10");
        
        context.AppendLine("\nCRITICAL CONSTRAINTS:");
        var criticalAllergens = persona.CriticalAllergens;
        if (criticalAllergens.Any())
            context.AppendLine($"SEVERE ALLERGIES (NEVER USE): {string.Join(", ", criticalAllergens)}");
        
        var commonRestrictions = persona.CommonDietaryRestrictions;
        if (commonRestrictions.Any())
            context.AppendLine($"Dietary Restrictions: {string.Join(", ", commonRestrictions)}");

        context.AppendLine("\nFAMILY PREFERENCES:");
        var topIngredients = GetTopLikedIngredients(persona, 10);
        if (topIngredients.Any())
            context.AppendLine($"Loved Ingredients: {string.Join(", ", topIngredients)}");
        
        var dislikedIngredients = GetTopDislikedIngredients(persona, 5);
        if (dislikedIngredients.Any())
            context.AppendLine($"Avoid When Possible: {string.Join(", ", dislikedIngredients)}");
        
        var topCuisines = GetTopCuisines(persona, 5);
        if (topCuisines.Any())
            context.AppendLine($"Preferred Cuisines: {string.Join(", ", topCuisines)}");

        context.AppendLine("\nINDIVIDUAL MEMBER PROFILES:");
        foreach (var member in persona.Members)
        {
            context.AppendLine(BuildMemberProfile(member));
        }

        return context.ToString();
    }

    private string BuildMemberProfile(PersonaMember member)
    {
        var profile = new StringBuilder();
        
        profile.AppendLine($"\n{member.Name} ({member.Age}y, {member.Relationship}):");
        
        // Critical information first
        if (member.Allergies.Any())
            profile.AppendLine($"  ?? ALLERGIES: {string.Join(", ", member.Allergies)}");
        
        if (member.DietaryRestrictions.Any())
            profile.AppendLine($"  ?? Diet: {string.Join(", ", member.DietaryRestrictions.Select(d => d.ToString()))}");
        
        // Preferences with confidence indicators
        var topLiked = GetTopRatedIngredients(member, true, 5);
        if (topLiked.Any())
            profile.AppendLine($"  ?? Loves: {string.Join(", ", topLiked)}");
        
        var topDisliked = GetTopRatedIngredients(member, false, 3);
        if (topDisliked.Any())
            profile.AppendLine($"  ? Dislikes: {string.Join(", ", topDisliked)}");
        
        profile.AppendLine($"  ??? Spice Tolerance: {member.SpiceToleranceLevel}/10");
        profile.AppendLine($"  ????? Cooking Skill: {member.CookingSkillLevel}");
        
        if (member.HealthGoals.Any())
            profile.AppendLine($"  ?? Health Goals: {string.Join(", ", member.HealthGoals)}");
        
        // Confidence indicator
        var confidence = member.BehaviorProfile.PersonaConfidenceScore;
        var confidenceEmoji = confidence > 0.8m ? "??" : confidence > 0.5m ? "??" : "??";
        profile.AppendLine($"  {confidenceEmoji} Profile Confidence: {confidence:P0} ({member.BehaviorProfile.TotalMealsEvaluated} meals evaluated)");
        
        return profile.ToString();
    }
}
```

### Dynamic Persona Adaptation
```csharp
public class PersonaAdaptationEngine : IPersonaAdaptationEngine
{
    public async Task<PersonaUpdateResult> AdaptPersonaAsync(
        FamilyPersona persona,
        PersonaAdaptationContext context)
    {
        var updateResult = new PersonaUpdateResult
        {
            PersonaId = persona.Id,
            UpdatedAt = DateTime.UtcNow,
            Changes = new List<PersonaChange>()
        };

        // Seasonal adaptations
        if (context.SeasonalUpdate)
        {
            await ApplySeasonalAdaptationsAsync(persona, context.CurrentSeason, updateResult);
        }

        // Life stage adaptations
        if (context.CheckLifeStageChanges)
        {
            await ApplyLifeStageAdaptationsAsync(persona, updateResult);
        }

        // Learning-based adaptations
        if (context.ApplyLearning)
        {
            await ApplyLearningAdaptationsAsync(persona, updateResult);
        }

        // Social trend adaptations
        if (context.SocialTrendAdaptation)
        {
            await ApplySocialTrendAdaptationsAsync(persona, context.TrendData, updateResult);
        }

        return updateResult;
    }

    private async Task ApplySeasonalAdaptationsAsync(
        FamilyPersona persona,
        Season currentSeason,
        PersonaUpdateResult updateResult)
    {
        foreach (var member in persona.Members)
        {
            var seasonalPrefs = member.BehaviorProfile.SeasonalPreferences
                .GetValueOrDefault(currentSeason.ToString(), new List<string>());

            if (seasonalPrefs.Any())
            {
                // Boost seasonal ingredient preferences
                foreach (var ingredient in seasonalPrefs)
                {
                    var currentScore = member.BehaviorProfile.IngredientPreferenceScores
                        .GetValueOrDefault(ingredient, 5.0m);
                    
                    var seasonalBoost = 1.2m; // 20% boost for seasonal ingredients
                    var newScore = Math.Min(10m, currentScore * seasonalBoost);
                    
                    member.BehaviorProfile.IngredientPreferenceScores[ingredient] = newScore;
                    
                    updateResult.Changes.Add(new PersonaChange
                    {
                        MemberId = member.Id,
                        ChangeType = "SeasonalBoost",
                        Description = $"Boosted preference for {ingredient} (seasonal)"
                    });
                }
            }
        }
    }

    private async Task ApplyLifeStageAdaptationsAsync(
        FamilyPersona persona,
        PersonaUpdateResult updateResult)
    {
        foreach (var member in persona.Members)
        {
            var ageCategory = CategorizeAge(member.Age);
            var previousAgeCategory = CategorizeAge(member.Age - 1); // Assume yearly updates
            
            if (ageCategory != previousAgeCategory)
            {
                await ApplyAgeTransitionAsync(member, ageCategory, updateResult);
            }
        }
    }

    private async Task ApplyLearningAdaptationsAsync(
        FamilyPersona persona,
        PersonaUpdateResult updateResult)
    {
        foreach (var member in persona.Members)
        {
            // Analyze recent rating patterns
            var recentRatings = member.BehaviorProfile.MealRatings
                .Where(r => r.RatedAt > DateTime.UtcNow.AddDays(-30))
                .ToList();

            if (recentRatings.Count >= 10) // Sufficient data for learning
            {
                await UpdatePreferenceConfidenceAsync(member, recentRatings, updateResult);
                await IdentifyNewPatternsAsync(member, recentRatings, updateResult);
            }
        }
    }
}
```

## Quality Metrics and Validation

### Persona Quality Assessment
```csharp
public class PersonaQualityAssessor : IPersonaQualityAssessor
{
    public PersonaQualityScore AssessPersonaQuality(FamilyPersona persona)
    {
        var score = new PersonaQualityScore
        {
            PersonaId = persona.Id,
            AssessedAt = DateTime.UtcNow
        };

        // Data completeness score (0-1)
        score.CompletenessScore = CalculateCompletenessScore(persona);
        
        // Consistency score (0-1)
        score.ConsistencyScore = CalculateConsistencyScore(persona);
        
        // Confidence score (0-1)
        score.ConfidenceScore = CalculateConfidenceScore(persona);
        
        // Freshness score (0-1)
        score.FreshnessScore = CalculateFreshnessScore(persona);
        
        // Overall quality score
        score.OverallScore = CalculateOverallScore(score);
        
        // Quality recommendations
        score.Recommendations = GenerateQualityRecommendations(persona, score);
        
        return score;
    }

    private decimal CalculateCompletenessScore(FamilyPersona persona)
    {
        decimal totalFields = 0;
        decimal completedFields = 0;

        foreach (var member in persona.Members)
        {
            // Essential fields
            totalFields += 10; // Basic demographics, allergies, restrictions, etc.
            
            if (!string.IsNullOrEmpty(member.Name)) completedFields++;
            if (member.Age > 0) completedFields++;
            if (!string.IsNullOrEmpty(member.Relationship)) completedFields++;
            if (member.DietaryRestrictions.Any()) completedFields++;
            if (member.SpiceToleranceLevel > 0) completedFields++;
            if (member.LikedIngredients.Any()) completedFields++;
            if (member.CuisinePreferences.Any()) completedFields++;
            if (!string.IsNullOrEmpty(member.CookingSkillLevel)) completedFields++;
            if (member.HealthGoals.Any()) completedFields++;
            if (member.BehaviorProfile.TotalMealsEvaluated > 0) completedFields++;
        }

        return totalFields > 0 ? completedFields / totalFields : 0;
    }

    private List<string> GenerateQualityRecommendations(
        FamilyPersona persona, 
        PersonaQualityScore score)
    {
        var recommendations = new List<string>();

        if (score.CompletenessScore < 0.7m)
        {
            recommendations.Add("Complete missing member profile information");
        }

        if (score.ConsistencyScore < 0.8m)
        {
            recommendations.Add("Review and resolve conflicting preference data");
        }

        if (score.ConfidenceScore < 0.6m)
        {
            recommendations.Add("Collect more meal feedback to improve accuracy");
        }

        if (score.FreshnessScore < 0.5m)
        {
            recommendations.Add("Update persona with recent behavior patterns");
        }

        var lowConfidenceMembers = persona.Members
            .Where(m => m.BehaviorProfile.PersonaConfidenceScore < 0.5m)
            .ToList();

        if (lowConfidenceMembers.Any())
        {
            recommendations.Add($"Focus learning on: {string.Join(", ", lowConfidenceMembers.Select(m => m.Name))}");
        }

        return recommendations;
    }
}
```

This comprehensive family persona modeling system enables the MealPrep application to deliver highly personalized AI-driven meal suggestions that evolve and improve over time based on family feedback and behavior patterns.

*This guide should be updated as persona modeling techniques evolve and new behavioral patterns are identified.*