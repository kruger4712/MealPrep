# Coding Standards

## Overview
Code style and conventions for the MealPrep project to ensure consistency and maintainability.

## C# Backend Standards

### Naming Conventions
- **Classes**: PascalCase (`MealSuggestionService`, `RecipeController`)
- **Methods**: PascalCase (`GetMealSuggestions`, `CreateRecipe`)
- **Properties**: PascalCase (`FamilyMembers`, `RecipeId`)
- **Fields**: camelCase with underscore prefix (`_aiService`, `_logger`)
- **Parameters**: camelCase (`mealType`, `userId`)
- **Constants**: UPPER_SNAKE_CASE (`MAX_SUGGESTIONS_COUNT`, `DEFAULT_TIMEOUT`)
- **Interfaces**: PascalCase with 'I' prefix (`IAiService`, `IRecipeRepository`)

### File Organization
```csharp
// File header with namespace
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using MealPrep.API.Models;
using MealPrep.API.Services;

namespace MealPrep.API.Controllers
{
    /// <summary>
    /// Controller for managing AI-powered meal suggestions
    /// </summary>
    [ApiController]
    [Route("api/[controller]")]
    public class AiSuggestionsController : ControllerBase
    {
        // Fields
        private readonly IAiService _aiService;
        private readonly ILogger<AiSuggestionsController> _logger;

        // Constructor
        public AiSuggestionsController(IAiService aiService, ILogger<AiSuggestionsController> logger)
        {
            _aiService = aiService ?? throw new ArgumentNullException(nameof(aiService));
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        }

        // Public methods
        [HttpPost("meal-suggestions")]
        public async Task<ActionResult<List<MealSuggestion>>> GenerateMealSuggestions(
            [FromBody] MealSuggestionRequest request)
        {
            // Implementation
        }

        // Private methods
        private bool ValidateRequest(MealSuggestionRequest request)
        {
            // Implementation
        }
    }
}
```

### Code Style Guidelines

#### Variable Declaration
```csharp
// Use var when type is obvious
var suggestions = await _aiService.GetMealSuggestions(request);
var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

// Use explicit types for complex objects or when clarity is needed
List<FamilyMember> familyMembers = await _familyService.GetFamilyMembersAsync(userId);
MealSuggestionRequest request = new MealSuggestionRequest();
```

#### Async/Await Patterns
```csharp
// Always use async/await for I/O operations
public async Task<List<Recipe>> GetRecipesAsync(int userId)
{
    var recipes = await _recipeRepository.GetByUserIdAsync(userId);
    return recipes.ToList();
}

// Use ConfigureAwait(false) in library code
public async Task<Recipe> CreateRecipeAsync(Recipe recipe)
{
    await _repository.AddAsync(recipe).ConfigureAwait(false);
    await _unitOfWork.SaveChangesAsync().ConfigureAwait(false);
    return recipe;
}
```

#### Error Handling
```csharp
// Use specific exception types
public async Task<Recipe> GetRecipeAsync(int recipeId)
{
    if (recipeId <= 0)
        throw new ArgumentException("Recipe ID must be positive", nameof(recipeId));

    var recipe = await _repository.GetByIdAsync(recipeId);
    if (recipe == null)
        throw new RecipeNotFoundException($"Recipe with ID {recipeId} not found");

    return recipe;
}

// Use try-catch for external service calls
public async Task<List<MealSuggestion>> GetAiSuggestionsAsync(MealSuggestionRequest request)
{
    try
    {
        var suggestions = await _aiService.GenerateSuggestionsAsync(request);
        return suggestions;
    }
    catch (AiServiceException ex)
    {
        _logger.LogError(ex, "Failed to generate AI suggestions for user {UserId}", request.UserId);
        return await _fallbackService.GetSuggestionsAsync(request);
    }
}
```

#### XML Documentation
```csharp
/// <summary>
/// Generates personalized meal suggestions based on family preferences and constraints
/// </summary>
/// <param name="request">The meal suggestion request containing family data and constraints</param>
/// <returns>A list of AI-generated meal suggestions ranked by family compatibility</returns>
/// <exception cref="ArgumentNullException">Thrown when request is null</exception>
/// <exception cref="AiServiceException">Thrown when AI service fails to generate suggestions</exception>
/// <example>
/// <code>
/// var request = new MealSuggestionRequest
/// {
///     FamilyMembers = familyMembers,
///     MealType = MealType.Dinner,
///     Budget = 25.00m
/// };
/// var suggestions = await aiService.GetMealSuggestions(request);
/// </code>
/// </example>
public async Task<List<MealSuggestion>> GetMealSuggestions(MealSuggestionRequest request)
{
    // Implementation
}
```

## TypeScript Frontend Standards

### Naming Conventions
- **Components**: PascalCase (`MealPlannerComponent`, `AiSuggestionsPanel`)
- **Variables/Functions**: camelCase (`getUserPreferences`, `generateSuggestions`)
- **Types/Interfaces**: PascalCase (`MealSuggestion`, `FamilyMember`)
- **Constants**: UPPER_SNAKE_CASE (`API_BASE_URL`, `MAX_UPLOAD_SIZE`)
- **Files**: kebab-case (`meal-planner.tsx`, `ai-suggestions.service.ts`)

### Component Structure
```typescript
import React, { useState, useEffect, useCallback } from 'react';
import { Card, Button, Typography, CircularProgress } from '@mui/material';
import { useMutation, useQuery } from '@tanstack/react-query';
import { MealSuggestion, FamilyMember, MealSuggestionRequest } from '../types';
import { aiService } from '../services/aiService';

interface MealPlannerProps {
  familyMembers: FamilyMember[];
  onSuggestionSelect: (suggestion: MealSuggestion) => void;
  className?: string;
}

export const MealPlanner: React.FC<MealPlannerProps> = ({
  familyMembers,
  onSuggestionSelect,
  className = ''
}) => {
  // State declarations
  const [selectedMealType, setSelectedMealType] = useState<'breakfast' | 'lunch' | 'dinner'>('dinner');
  const [budget, setBudget] = useState<number>(25);

  // Queries and mutations
  const {
    data: suggestions,
    isLoading,
    error,
    refetch
  } = useQuery({
    queryKey: ['meal-suggestions', familyMembers, selectedMealType, budget],
    queryFn: () => aiService.getMealSuggestions({
      familyMembers,
      mealType: selectedMealType,
      budget
    }),
    enabled: familyMembers.length > 0
  });

  // Event handlers
  const handleMealTypeChange = useCallback((mealType: 'breakfast' | 'lunch' | 'dinner') => {
    setSelectedMealType(mealType);
  }, []);

  const handleSuggestionSelect = useCallback((suggestion: MealSuggestion) => {
    onSuggestionSelect(suggestion);
  }, [onSuggestionSelect]);

  // Effects
  useEffect(() => {
    if (familyMembers.length === 0) {
      console.warn('No family members provided to MealPlanner');
    }
  }, [familyMembers]);

  // Render
  return (
    <Card className={`meal-planner ${className}`}>
      <Typography variant="h5" component="h2" gutterBottom>
        AI Meal Suggestions
      </Typography>
      
      {/* Component JSX */}
      {isLoading && <CircularProgress />}
      {error && <Typography color="error">Failed to load suggestions</Typography>}
      {suggestions?.map((suggestion) => (
        <SuggestionCard
          key={suggestion.id}
          suggestion={suggestion}
          onSelect={handleSuggestionSelect}
        />
      ))}
    </Card>
  );
};

export default MealPlanner;
```

### Type Definitions
```typescript
// Use interfaces for object shapes
interface MealSuggestion {
  id: string;
  name: string;
  description: string;
  familyFitScore: number;
  prepTime: number;
  cookTime: number;
  estimatedCost: number;
  ingredients: Ingredient[];
  instructions: CookingInstruction[];
  nutritionalInfo: NutritionalInfo;
}

// Use union types for constrained values
type MealType = 'breakfast' | 'lunch' | 'dinner' | 'snack';
type DietaryRestriction = 'vegetarian' | 'vegan' | 'gluten-free' | 'dairy-free' | 'nut-free';

// Use enums for related constants
enum CookingSkillLevel {
  Beginner = 'beginner',
  Intermediate = 'intermediate',
  Advanced = 'advanced',
  Expert = 'expert'
}
```

### React Hooks Best Practices
```typescript
// Custom hooks for reusable logic
export const useAiSuggestions = (familyMembers: FamilyMember[]) => {
  const [isGenerating, setIsGenerating] = useState(false);
  
  const generateSuggestions = useCallback(async (request: MealSuggestionRequest) => {
    setIsGenerating(true);
    try {
      const suggestions = await aiService.getMealSuggestions(request);
      return suggestions;
    } finally {
      setIsGenerating(false);
    }
  }, []);

  return { generateSuggestions, isGenerating };
};

// Memoization for expensive operations
const MealCard = React.memo<MealCardProps>(({ meal, onSelect }) => {
  const formattedNutrition = useMemo(() => {
    return formatNutritionalInfo(meal.nutritionalInfo);
  }, [meal.nutritionalInfo]);

  return (
    <Card onClick={() => onSelect(meal)}>
      {/* Card content */}
    </Card>
  );
});
```

## Database Standards

### Entity Framework Conventions
```csharp
// Entity naming
public class Recipe
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public int PrepTimeMinutes { get; set; }
    public decimal EstimatedCost { get; set; }
    public DateTime CreatedDate { get; set; }
    public DateTime UpdatedDate { get; set; }

    // Navigation properties
    public virtual List<RecipeIngredient> Ingredients { get; set; } = new();
    public virtual List<RecipeTag> Tags { get; set; } = new();
    public virtual User CreatedBy { get; set; } = null!;
}

// Configuration class
public class RecipeConfiguration : IEntityTypeConfiguration<Recipe>
{
    public void Configure(EntityTypeBuilder<Recipe> builder)
    {
        builder.ToTable("Recipes");
        
        builder.HasKey(r => r.Id);
        
        builder.Property(r => r.Name)
            .IsRequired()
            .HasMaxLength(200);
            
        builder.Property(r => r.EstimatedCost)
            .HasColumnType("decimal(10,2)");
            
        builder.HasIndex(r => r.Name)
            .HasDatabaseName("IX_Recipe_Name");
    }
}
```

### Migration Naming
```csharp
// Use descriptive migration names
// Good: AddFamilyPersonaTable, UpdateRecipeNutritionColumns
// Bad: Migration1, Update1

// Example migration
public partial class AddFamilyPersonaTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "FamilyPersonas",
            columns: table => new
            {
                Id = table.Column<int>(type: "int", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(type: "nvarchar(100)", maxLength: 100, nullable: false),
                Age = table.Column<int>(type: "int", nullable: false)
            });
    }
}
```

## Testing Standards

### Unit Test Structure
```csharp
[TestClass]
public class MealSuggestionServiceTests
{
    private Mock<IAiService> _mockAiService;
    private Mock<ILogger<MealSuggestionService>> _mockLogger;
    private MealSuggestionService _service;

    [TestInitialize]
    public void Setup()
    {
        _mockAiService = new Mock<IAiService>();
        _mockLogger = new Mock<ILogger<MealSuggestionService>>();
        _service = new MealSuggestionService(_mockAiService.Object, _mockLogger.Object);
    }

    [TestMethod]
    public async Task GetMealSuggestions_ValidRequest_ReturnsAppropriateRecipes()
    {
        // Arrange
        var request = new MealSuggestionRequest
        {
            FamilyMembers = CreateTestFamilyMembers(),
            MealType = MealType.Dinner,
            Budget = 25.00m
        };

        var expectedSuggestions = CreateTestSuggestions();
        _mockAiService.Setup(x => x.GenerateContentAsync(It.IsAny<string>()))
                     .ReturnsAsync(JsonSerializer.Serialize(expectedSuggestions));

        // Act
        var result = await _service.GetMealSuggestions(request);

        // Assert
        Assert.AreEqual(3, result.Count);
        Assert.IsTrue(result.All(s => s.EstimatedCost <= request.Budget));
        Assert.IsTrue(result.All(s => s.FamilyFitScore > 0));
    }

    private List<FamilyMember> CreateTestFamilyMembers()
    {
        return new List<FamilyMember>
        {
            new FamilyMember { Name = "John", Age = 35, DietaryRestrictions = new[] { "None" } },
            new FamilyMember { Name = "Jane", Age = 32, DietaryRestrictions = new[] { "Vegetarian" } }
        };
    }
}
```

### React Component Testing
```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { MealPlanner } from './MealPlanner';
import { mockFamilyMembers } from '../__mocks__/testData';

const createTestQueryClient = () => new QueryClient({
  defaultOptions: {
    queries: { retry: false },
    mutations: { retry: false },
  },
});

const renderWithProviders = (component: React.ReactElement) => {
  const queryClient = createTestQueryClient();
  return render(
    <QueryClientProvider client={queryClient}>
      {component}
    </QueryClientProvider>
  );
};

describe('MealPlanner Component', () => {
  test('renders meal planner with family members', () => {
    renderWithProviders(
      <MealPlanner
        familyMembers={mockFamilyMembers}
        onSuggestionSelect={jest.fn()}
      />
    );

    expect(screen.getByText('AI Meal Suggestions')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /generate suggestions/i })).toBeInTheDocument();
  });

  test('calls onSuggestionSelect when suggestion is clicked', async () => {
    const mockOnSelect = jest.fn();
    
    renderWithProviders(
      <MealPlanner
        familyMembers={mockFamilyMembers}
        onSuggestionSelect={mockOnSelect}
      />
    );

    const generateButton = screen.getByRole('button', { name: /generate suggestions/i });
    fireEvent.click(generateButton);

    await waitFor(() => {
      const suggestionCard = screen.getByTestId('suggestion-card-1');
      fireEvent.click(suggestionCard);
      expect(mockOnSelect).toHaveBeenCalledTimes(1);
    });
  });
});
```

## Code Quality Tools

### Backend Tools
- **EditorConfig** - Consistent formatting across IDEs
- **StyleCop** - C# style analysis
- **SonarQube** - Code quality and security analysis
- **Roslynator** - Additional C# analyzers

### Frontend Tools
- **ESLint** - JavaScript/TypeScript linting
- **Prettier** - Code formatting
- **Husky** - Git hooks for pre-commit checks
- **lint-staged** - Run linters on staged files

### Configuration Files

#### .editorconfig
```ini
root = true

[*]
charset = utf-8
end_of_line = crlf
insert_final_newline = true
indent_style = space
indent_size = 2

[*.{cs,csx}]
indent_size = 4

[*.{js,jsx,ts,tsx}]
indent_size = 2
```

#### .eslintrc.json
```json
{
  "extends": [
    "react-app",
    "react-app/jest",
    "@typescript-eslint/recommended"
  ],
  "rules": {
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/explicit-function-return-type": "off",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

## Git Commit Standards

### Commit Message Format
```
type(scope): description

[optional body]

[optional footer]
```

### Commit Types
- **feat**: New feature
- **fix**: Bug fix
- **docs**: Documentation changes
- **style**: Code style changes (formatting, missing semicolons, etc.)
- **refactor**: Code refactoring
- **test**: Adding or updating tests
- **chore**: Maintenance tasks

### Examples
```
feat(ai): add family persona modeling for meal suggestions

- Implement FamilyPersona model with dietary restrictions
- Add persona-based prompt engineering
- Update AI service to use family preferences

Closes #123
```

## Performance Guidelines

### Backend Performance
- Use async/await for I/O operations
- Implement proper caching strategies
- Optimize database queries with includes
- Use pagination for large datasets
- Implement request/response compression

### Frontend Performance
- Use React.memo for expensive components
- Implement code splitting with React.lazy
- Optimize bundle size with tree shaking
- Use virtual scrolling for large lists
- Implement proper image optimization

---

These coding standards should be followed consistently across the MealPrep codebase to ensure maintainability, readability, and quality. Regular code reviews should enforce these standards and help team members learn best practices.

*This document should be updated as new patterns emerge and the codebase evolves.*