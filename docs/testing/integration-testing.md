# Integration Testing Guide

## Overview
Comprehensive guide for writing and executing integration tests in the MealPrep application, covering API testing, database integration, external service testing, and end-to-end workflow validation.

## Integration Testing Philosophy

### Definition
Integration testing verifies that different components or services work correctly together. Unlike unit tests that test components in isolation, integration tests validate the interactions between:
- Controllers and Services
- Services and Repositories
- Application and Database
- Application and External APIs (Google Gemini AI)
- Frontend and Backend Integration

### Testing Pyramid Integration Layer
```
         E2E Tests (5%)
      Integration Tests (25%)
    Unit Tests (70%)
```

Integration tests form the crucial middle layer, providing confidence that components work together while remaining faster than full E2E tests.

---

## Backend Integration Testing (.NET/C#)

### Testing Framework Setup

#### Core Testing Dependencies
```xml
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
<PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="8.0.0" />
<PackageReference Include="Testcontainers" Version="3.6.0" />
<PackageReference Include="Testcontainers.MsSql" Version="3.6.0" />
<PackageReference Include="WebApplicationFactory" Version="8.0.0" />
<PackageReference Include="Respawn" Version="6.1.0" />
<PackageReference Include="WireMock.Net" Version="1.5.42" />
```

#### Test Project Structure
```
MealPrep.IntegrationTests/
??? Controllers/
?   ??? RecipeControllerIntegrationTests.cs
?   ??? FamilyControllerIntegrationTests.cs
?   ??? AiSuggestionControllerIntegrationTests.cs
??? Services/
?   ??? RecipeServiceIntegrationTests.cs
?   ??? AiSuggestionServiceIntegrationTests.cs
??? Database/
?   ??? RecipeRepositoryIntegrationTests.cs
?   ??? DatabaseMigrationTests.cs
??? External/
?   ??? GeminiAiServiceIntegrationTests.cs
?   ??? EmailServiceIntegrationTests.cs
??? Workflows/
?   ??? MealPlanningWorkflowTests.cs
?   ??? UserRegistrationWorkflowTests.cs
??? Fixtures/
?   ??? WebApplicationTestFixture.cs
?   ??? DatabaseFixture.cs
?   ??? MockExternalServicesFixture.cs
??? Helpers/
    ??? TestDataSeeder.cs
    ??? HttpClientExtensions.cs
```

### WebApplicationFactory Setup

#### Custom Test Web Application Factory
```csharp
public class MealPrepWebApplicationFactory : WebApplicationFactory<Program>
{
    public string DatabaseName { get; } = $"MealPrepTestDb_{Guid.NewGuid()}";

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureAppConfiguration((context, config) =>
        {
            // Override configuration for testing
            config.AddInMemoryCollection(new Dictionary<string, string>
            {
                ["ConnectionStrings:DefaultConnection"] = GetTestConnectionString(),
                ["GoogleCloud:ProjectId"] = "test-project",
                ["Authentication:JwtSettings:SecretKey"] = "test-secret-key-for-integration-tests-minimum-32-chars",
                ["Email:EnableEmailSending"] = "false"
            });
        });

        builder.ConfigureServices(services =>
        {
            // Remove the existing DbContext registration
            var descriptor = services.SingleOrDefault(d => 
                d.ServiceType == typeof(DbContextOptions<MealPrepDbContext>));
            if (descriptor != null)
                services.Remove(descriptor);

            // Add in-memory database for testing
            services.AddDbContext<MealPrepDbContext>(options =>
            {
                options.UseInMemoryDatabase(DatabaseName);
                options.EnableSensitiveDataLogging();
            });

            // Replace external services with mocks
            services.Replace(ServiceDescriptor.Singleton<IGeminiAiService, MockGeminiAiService>());
            services.Replace(ServiceDescriptor.Singleton<IEmailService, MockEmailService>());
        });
    }

    private string GetTestConnectionString()
    {
        return $"Server=(localdb)\\mssqllocaldb;Database={DatabaseName};Trusted_Connection=true;MultipleActiveResultSets=true";
    }
}
```

#### Base Integration Test Class
```csharp
public abstract class IntegrationTestBase : IClassFixture<MealPrepWebApplicationFactory>
{
    protected readonly MealPrepWebApplicationFactory Factory;
    protected readonly HttpClient Client;
    protected readonly MealPrepDbContext DbContext;
    protected readonly IServiceScope Scope;

    protected IntegrationTestBase(MealPrepWebApplicationFactory factory)
    {
        Factory = factory;
        Client = factory.CreateClient();
        Scope = factory.Services.CreateScope();
        DbContext = Scope.ServiceProvider.GetRequiredService<MealPrepDbContext>();
    }

    protected async Task<string> GetAuthTokenAsync(string email = "test@example.com")
    {
        // Create test user and return JWT token
        var user = new User
        {
            Email = email,
            FirstName = "Test",
            LastName = "User",
            PasswordHash = BCrypt.Net.BCrypt.HashPassword("TestPassword123!"),
            EmailVerified = true,
            IsActive = true
        };

        DbContext.Users.Add(user);
        await DbContext.SaveChangesAsync();

        var loginRequest = new LoginRequest
        {
            Email = email,
            Password = "TestPassword123!"
        };

        var response = await Client.PostAsJsonAsync("/api/auth/login", loginRequest);
        var loginResponse = await response.Content.ReadFromJsonAsync<LoginResponse>();
        
        return loginResponse.Token;
    }

    protected void SetAuthorizationHeader(string token)
    {
        Client.DefaultRequestHeaders.Authorization = 
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
    }

    public virtual void Dispose()
    {
        Scope?.Dispose();
        Client?.Dispose();
    }
}
```

### Controller Integration Tests

#### Recipe Controller Integration Tests
```csharp
[Collection("IntegrationTest")]
public class RecipeControllerIntegrationTests : IntegrationTestBase
{
    public RecipeControllerIntegrationTests(MealPrepWebApplicationFactory factory) 
        : base(factory) { }

    [Fact]
    public async Task GetRecipes_WithValidUser_ReturnsUserRecipes()
    {
        // Arrange
        var token = await GetAuthTokenAsync();
        SetAuthorizationHeader(token);

        var user = await DbContext.Users.FirstAsync(u => u.Email == "test@example.com");
        var recipe = new Recipe
        {
            Name = "Test Recipe",
            Description = "A test recipe",
            Instructions = "1. Test\n2. More testing",
            PrepTimeMinutes = 15,
            CookTimeMinutes = 30,
            ServingSize = 4,
            DifficultyLevel = "Easy",
            CreatedByUserId = user.Id,
            IsActive = true
        };

        DbContext.Recipes.Add(recipe);
        await DbContext.SaveChangesAsync();

        // Act
        var response = await Client.GetAsync("/api/recipes");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var recipes = await response.Content.ReadFromJsonAsync<List<RecipeDto>>();
        recipes.Should().NotBeNull();
        recipes.Should().HaveCount(1);
        recipes[0].Name.Should().Be("Test Recipe");
    }

    [Fact]
    public async Task CreateRecipe_WithValidData_CreatesRecipeSuccessfully()
    {
        // Arrange
        var token = await GetAuthTokenAsync();
        SetAuthorizationHeader(token);

        var createRequest = new CreateRecipeDto
        {
            Name = "New Recipe",
            Description = "A new test recipe",
            Instructions = "1. Create\n2. Test\n3. Verify",
            PrepTimeMinutes = 20,
            CookTimeMinutes = 25,
            ServingSize = 2,
            DifficultyLevel = "Medium",
            Cuisine = "Italian",
            Ingredients = new List<CreateRecipeIngredientDto>
            {
                new CreateRecipeIngredientDto
                {
                    IngredientName = "Pasta",
                    Quantity = 200,
                    Unit = "grams"
                },
                new CreateRecipeIngredientDto
                {
                    IngredientName = "Tomato Sauce",
                    Quantity = 1,
                    Unit = "cup"
                }
            }
        };

        // Act
        var response = await Client.PostAsJsonAsync("/api/recipes", createRequest);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        
        var createdRecipe = await response.Content.ReadFromJsonAsync<RecipeDto>();
        createdRecipe.Should().NotBeNull();
        createdRecipe.Name.Should().Be("New Recipe");

        // Verify in database
        var dbRecipe = await DbContext.Recipes
            .Include(r => r.Ingredients)
            .FirstAsync(r => r.Id == createdRecipe.Id);
        
        dbRecipe.Ingredients.Should().HaveCount(2);
    }

    [Fact]
    public async Task SearchRecipes_WithFilters_ReturnsFilteredResults()
    {
        // Arrange
        var token = await GetAuthTokenAsync();
        SetAuthorizationHeader(token);

        await SeedTestRecipes();

        var searchQuery = "?searchTerm=pasta&cuisine=Italian&maxPrepTime=30";

        // Act
        var response = await Client.GetAsync($"/api/recipes/search{searchQuery}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var recipes = await response.Content.ReadFromJsonAsync<List<RecipeDto>>();
        recipes.Should().NotBeNull();
        recipes.Should().OnlyContain(r => r.Cuisine == "Italian" && r.PrepTimeMinutes <= 30);
    }

    private async Task SeedTestRecipes()
    {
        var user = await DbContext.Users.FirstAsync(u => u.Email == "test@example.com");
        
        var recipes = new[]
        {
            new Recipe
            {
                Name = "Pasta Carbonara",
                Description = "Classic Italian pasta",
                Instructions = "Traditional carbonara recipe",
                PrepTimeMinutes = 15,
                CookTimeMinutes = 20,
                ServingSize = 4,
                DifficultyLevel = "Medium",
                Cuisine = "Italian",
                CreatedByUserId = user.Id,
                IsActive = true
            },
            new Recipe
            {
                Name = "Chicken Curry",
                Description = "Spicy Indian curry",
                Instructions = "Authentic curry recipe",
                PrepTimeMinutes = 45,
                CookTimeMinutes = 60,
                ServingSize = 6,
                DifficultyLevel = "Hard",
                Cuisine = "Indian",
                CreatedByUserId = user.Id,
                IsActive = true
            }
        };

        DbContext.Recipes.AddRange(recipes);
        await DbContext.SaveChangesAsync();
    }
}
```

### Service Layer Integration Tests

#### AI Suggestion Service Integration Tests
```csharp
[Collection("IntegrationTest")]
public class AiSuggestionServiceIntegrationTests : IntegrationTestBase
{
    private readonly IAiSuggestionService _aiSuggestionService;

    public AiSuggestionServiceIntegrationTests(MealPrepWebApplicationFactory factory) 
        : base(factory)
    {
        _aiSuggestionService = Scope.ServiceProvider.GetRequiredService<IAiSuggestionService>();
    }

    [Fact]
    public async Task GenerateMealSuggestions_WithFamilyData_ReturnsPersonalizedSuggestions()
    {
        // Arrange
        var user = await CreateTestUserWithFamily();
        
        var request = new AiSuggestionRequest
        {
            UserId = user.Id,
            MealType = "Dinner",
            MaxPrepTime = 45,
            MaxCookTime = 60,
            ServingSize = 4,
            Occasion = "Weeknight"
        };

        // Act
        var suggestions = await _aiSuggestionService.GenerateMealSuggestionsAsync(request);

        // Assert
        suggestions.Should().NotBeNull();
        suggestions.Should().HaveCountGreaterThan(0);
        suggestions.Should().OnlyContain(s => s.FamilyFitScore > 0);
        
        // Verify suggestions respect family preferences
        var hasAllergenFreeOptions = suggestions.Any(s => 
            !s.Recipe.Ingredients.Any(i => i.IngredientName.Contains("nuts")));
        hasAllergenFreeOptions.Should().BeTrue();
    }

    [Fact]
    public async Task GenerateWeeklyMenu_WithConstraints_CreatesBalancedMenu()
    {
        // Arrange
        var user = await CreateTestUserWithFamily();
        
        var request = new WeeklyMenuRequest
        {
            UserId = user.Id,
            StartDate = DateTime.Today.AddDays(-(int)DateTime.Today.DayOfWeek + 1), // Monday
            MaxBudget = 150.00m,
            PreferredCuisines = new[] { "Italian", "Mexican", "American" },
            AvoidRepeats = true
        };

        // Act
        var weeklyMenu = await _aiSuggestionService.GenerateWeeklyMenuAsync(request);

        // Assert
        weeklyMenu.Should().NotBeNull();
        weeklyMenu.Meals.Should().HaveCount(21); // 3 meals × 7 days
        weeklyMenu.EstimatedCost.Should().BeLessOrEqualTo(150.00m);
        
        // Verify variety in cuisines
        var cuisines = weeklyMenu.Meals.Select(m => m.Recipe.Cuisine).Distinct();
        cuisines.Should().HaveCountGreaterThan(1);
    }

    private async Task<User> CreateTestUserWithFamily()
    {
        var user = new User
        {
            Email = "family@example.com",
            FirstName = "John",
            LastName = "Family",
            PasswordHash = BCrypt.Net.BCrypt.HashPassword("TestPassword123!"),
            EmailVerified = true,
            IsActive = true
        };

        DbContext.Users.Add(user);
        await DbContext.SaveChangesAsync();

        var familyMembers = new[]
        {
            new FamilyMember
            {
                UserId = user.Id,
                Name = "John",
                Age = 35,
                SpiceToleranceLevel = 7,
                CookingSkillLevel = "Intermediate",
                PreferredCookingTime = 60,
                IsActive = true,
                Preferences = new List<FamilyMemberPreference>
                {
                    new FamilyMemberPreference
                    {
                        PreferenceType = "Like",
                        PreferenceValue = "Italian cuisine",
                        Severity = "Medium"
                    }
                }
            },
            new FamilyMember
            {
                UserId = user.Id,
                Name = "Jane",
                Age = 32,
                SpiceToleranceLevel = 4,
                CookingSkillLevel = "Beginner",
                PreferredCookingTime = 30,
                IsActive = true,
                Preferences = new List<FamilyMemberPreference>
                {
                    new FamilyMemberPreference
                    {
                        PreferenceType = "Allergy",
                        PreferenceValue = "nuts",
                        Severity = "Critical"
                    }
                }
            }
        };

        DbContext.FamilyMembers.AddRange(familyMembers);
        await DbContext.SaveChangesAsync();

        return user;
    }
}
```

### Database Integration Tests

#### Repository Integration Tests
```csharp
[Collection("IntegrationTest")]
public class RecipeRepositoryIntegrationTests : IntegrationTestBase
{
    private readonly IRecipeRepository _recipeRepository;

    public RecipeRepositoryIntegrationTests(MealPrepWebApplicationFactory factory) 
        : base(factory)
    {
        _recipeRepository = Scope.ServiceProvider.GetRequiredService<IRecipeRepository>();
    }

    [Fact]
    public async Task SearchRecipesAsync_WithComplexFilters_ReturnsCorrectResults()
    {
        // Arrange
        await SeedComplexRecipeData();

        var criteria = new RecipeSearchCriteria
        {
            SearchTerm = "chicken",
            Cuisine = "Italian",
            MaxPrepTime = 30,
            MinRating = 4.0m,
            DifficultyLevel = "Easy",
            MaxIngredientCount = 10
        };

        // Act
        var results = await _recipeRepository.SearchAsync(criteria);

        // Assert
        results.Should().NotBeNull();
        results.Should().OnlyContain(r => 
            r.Name.Contains("chicken", StringComparison.OrdinalIgnoreCase) ||
            r.Description.Contains("chicken", StringComparison.OrdinalIgnoreCase));
        results.Should().OnlyContain(r => r.Cuisine == "Italian");
        results.Should().OnlyContain(r => r.PrepTimeMinutes <= 30);
        results.Should().OnlyContain(r => r.AverageRating >= 4.0m);
    }

    [Fact]
    public async Task GetRecipeWithIngredientsAsync_ExistingRecipe_ReturnsCompleteRecipe()
    {
        // Arrange
        var recipe = await CreateRecipeWithIngredients();

        // Act
        var result = await _recipeRepository.GetWithIngredientsAsync(recipe.Id);

        // Assert
        result.Should().NotBeNull();
        result.Id.Should().Be(recipe.Id);
        result.Ingredients.Should().HaveCount(3);
        result.Ingredients.Should().OnlyContain(i => !string.IsNullOrEmpty(i.IngredientName));
    }

    private async Task<Recipe> CreateRecipeWithIngredients()
    {
        var recipe = new Recipe
        {
            Name = "Chicken Parmesan",
            Description = "Breaded chicken with marinara",
            Instructions = "1. Bread chicken\n2. Fry\n3. Add sauce",
            PrepTimeMinutes = 20,
            CookTimeMinutes = 25,
            ServingSize = 4,
            DifficultyLevel = "Easy",
            Cuisine = "Italian",
            IsActive = true,
            Ingredients = new List<RecipeIngredient>
            {
                new RecipeIngredient
                {
                    IngredientName = "Chicken Breast",
                    Quantity = 4,
                    Unit = "pieces"
                },
                new RecipeIngredient
                {
                    IngredientName = "Breadcrumbs",
                    Quantity = 1,
                    Unit = "cup"
                },
                new RecipeIngredient
                {
                    IngredientName = "Marinara Sauce",
                    Quantity = 2,
                    Unit = "cups"
                }
            }
        };

        DbContext.Recipes.Add(recipe);
        await DbContext.SaveChangesAsync();

        return recipe;
    }
}
```

### External Service Integration Tests

#### Mock External Services
```csharp
public class MockGeminiAiService : IGeminiAiService
{
    public async Task<AiSuggestionResponse> GenerateMealSuggestionsAsync(AiSuggestionRequest request)
    {
        // Simulate AI processing delay
        await Task.Delay(100);

        return new AiSuggestionResponse
        {
            Suggestions = new List<MealSuggestion>
            {
                new MealSuggestion
                {
                    Recipe = new Recipe
                    {
                        Name = "AI Generated Recipe",
                        Description = "Mock AI generated recipe",
                        Instructions = "Mock instructions",
                        PrepTimeMinutes = 20,
                        CookTimeMinutes = 30,
                        ServingSize = request.ServingSize,
                        DifficultyLevel = "Easy",
                        IsAiGenerated = true
                    },
                    FamilyFitScore = 8,
                    FamilyFitReasoning = "Mock reasoning for family fit"
                }
            },
            RequestId = Guid.NewGuid().ToString(),
            ProcessingTimeMs = 1500,
            TokensUsed = 150
        };
    }
}

public class MockEmailService : IEmailService
{
    public List<EmailSent> SentEmails { get; } = new List<EmailSent>();

    public async Task<bool> SendEmailAsync(EmailMessage message)
    {
        SentEmails.Add(new EmailSent
        {
            To = message.To,
            Subject = message.Subject,
            Body = message.Body,
            SentAt = DateTime.UtcNow
        });

        return true;
    }
}
```

---

## Frontend Integration Testing (React/TypeScript)

### Testing Framework Setup

#### Frontend Integration Testing Dependencies
```json
{
  "devDependencies": {
    "@testing-library/react": "^13.4.0",
    "@testing-library/jest-dom": "^5.16.5",
    "@testing-library/user-event": "^14.4.3",
    "msw": "^1.3.2",
    "jest-environment-jsdom": "^27.5.1",
    "@types/jest": "^27.5.2"
  }
}
```

#### Mock Service Worker Setup
```typescript
// src/mocks/handlers.ts
import { rest } from 'msw';
import { Recipe, CreateRecipeDto } from '../types/Recipe';

export const handlers = [
  // Recipe endpoints
  rest.get('/api/recipes', (req, res, ctx) => {
    const mockRecipes: Recipe[] = [
      {
        id: 1,
        name: 'Test Recipe',
        description: 'A test recipe',
        prepTimeMinutes: 15,
        cookTimeMinutes: 30,
        servingSize: 4,
        difficulty: 'Easy',
        cuisine: 'Italian'
      }
    ];

    return res(ctx.json(mockRecipes));
  }),

  rest.post('/api/recipes', (req, res, ctx) => {
    const newRecipe = req.body as CreateRecipeDto;
    
    return res(
      ctx.status(201),
      ctx.json({
        id: 999,
        ...newRecipe,
        createdDate: new Date().toISOString()
      })
    );
  }),

  // AI suggestion endpoints
  rest.post('/api/ai/suggestions', (req, res, ctx) => {
    return res(
      ctx.json({
        suggestions: [
          {
            recipe: {
              id: 1001,
              name: 'AI Suggested Recipe',
              description: 'Generated by AI',
              prepTimeMinutes: 25,
              cookTimeMinutes: 35,
              servingSize: 4,
              difficulty: 'Medium',
              isAiGenerated: true
            },
            familyFitScore: 9,
            reasoning: 'Perfect match for your family preferences'
          }
        ]
      })
    );
  })
];

// src/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

### Component Integration Tests

#### Recipe Management Integration Test
```typescript
// src/components/RecipeManagement/RecipeManagement.integration.test.tsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';
import { RecipeManagement } from './RecipeManagement';
import { AuthProvider } from '../../contexts/AuthContext';
import { server } from '../../mocks/server';

// Test wrapper with all providers
const TestWrapper: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  });

  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <AuthProvider>
          {children}
        </AuthProvider>
      </BrowserRouter>
    </QueryClientProvider>
  );
};

describe('RecipeManagement Integration', () => {
  beforeAll(() => server.listen());
  afterEach(() => server.resetHandlers());
  afterAll(() => server.close());

  it('loads and displays recipes from API', async () => {
    render(
      <TestWrapper>
        <RecipeManagement />
      </TestWrapper>
    );

    // Wait for recipes to load
    await waitFor(() => {
      expect(screen.getByText('Test Recipe')).toBeInTheDocument();
    });

    expect(screen.getByText('A test recipe')).toBeInTheDocument();
    expect(screen.getByText('15 min prep')).toBeInTheDocument();
    expect(screen.getByText('30 min cook')).toBeInTheDocument();
  });

  it('creates new recipe through full workflow', async () => {
    const user = userEvent.setup();
    
    render(
      <TestWrapper>
        <RecipeManagement />
      </TestWrapper>
    );

    // Open create recipe form
    await user.click(screen.getByText('Add Recipe'));

    // Fill out form
    await user.type(screen.getByLabelText('Recipe Name'), 'New Test Recipe');
    await user.type(screen.getByLabelText('Description'), 'A new test recipe');
    await user.type(screen.getByLabelText('Prep Time'), '20');
    await user.type(screen.getByLabelText('Cook Time'), '25');
    await user.selectOptions(screen.getByLabelText('Difficulty'), 'Medium');

    // Add ingredients
    await user.click(screen.getByText('Add Ingredient'));
    await user.type(screen.getByLabelText('Ingredient Name'), 'Chicken');
    await user.type(screen.getByLabelText('Quantity'), '2');
    await user.type(screen.getByLabelText('Unit'), 'lbs');

    // Submit form
    await user.click(screen.getByText('Create Recipe'));

    // Verify success message and new recipe appears
    await waitFor(() => {
      expect(screen.getByText('Recipe created successfully!')).toBeInTheDocument();
    });

    // The new recipe should appear in the list (mocked response)
    expect(screen.getByText('New Test Recipe')).toBeInTheDocument();
  });

  it('handles search and filtering', async () => {
    const user = userEvent.setup();
    
    render(
      <TestWrapper>
        <RecipeManagement />
      </TestWrapper>
    );

    // Wait for initial load
    await waitFor(() => {
      expect(screen.getByText('Test Recipe')).toBeInTheDocument();
    });

    // Perform search
    await user.type(screen.getByPlaceholderText('Search recipes...'), 'pasta');
    await user.click(screen.getByText('Search'));

    // Verify search request was made (handled by MSW)
    await waitFor(() => {
      // The search should filter results
      expect(screen.queryByText('Test Recipe')).not.toBeInTheDocument();
    });
  });
});
```

### Workflow Integration Tests

#### Meal Planning Workflow Test
```typescript
// src/workflows/MealPlanning.integration.test.tsx
import React from 'react';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MealPlanningWorkflow } from './MealPlanningWorkflow';
import { TestWrapper } from '../test-utils/TestWrapper';

describe('Meal Planning Workflow Integration', () => {
  it('completes full meal planning workflow', async () => {
    const user = userEvent.setup();
    
    render(
      <TestWrapper>
        <MealPlanningWorkflow />
      </TestWrapper>
    );

    // Step 1: Family Setup
    expect(screen.getByText('Set up your family')).toBeInTheDocument();
    
    await user.click(screen.getByText('Add Family Member'));
    await user.type(screen.getByLabelText('Name'), 'John');
    await user.type(screen.getByLabelText('Age'), '35');
    await user.selectOptions(screen.getByLabelText('Cooking Skill'), 'Intermediate');
    await user.click(screen.getByText('Save Member'));

    await user.click(screen.getByText('Next: Preferences'));

    // Step 2: Preferences
    expect(screen.getByText('Set dietary preferences')).toBeInTheDocument();
    
    await user.click(screen.getByText('Add Preference'));
    await user.selectOptions(screen.getByLabelText('Type'), 'Allergy');
    await user.type(screen.getByLabelText('Value'), 'nuts');
    await user.selectOptions(screen.getByLabelText('Severity'), 'Critical');
    await user.click(screen.getByText('Add'));

    await user.click(screen.getByText('Next: AI Suggestions'));

    // Step 3: AI Suggestions
    expect(screen.getByText('Get AI meal suggestions')).toBeInTheDocument();
    
    await user.click(screen.getByText('Generate Suggestions'));

    // Wait for AI suggestions to load
    await waitFor(() => {
      expect(screen.getByText('AI Suggested Recipe')).toBeInTheDocument();
    });

    expect(screen.getByText('Family Fit: 9/10')).toBeInTheDocument();
    
    // Accept suggestion
    await user.click(screen.getByText('Add to Menu'));

    // Step 4: Menu Review
    await user.click(screen.getByText('Review Menu'));
    
    expect(screen.getByText('Weekly Menu')).toBeInTheDocument();
    expect(screen.getByText('AI Suggested Recipe')).toBeInTheDocument();

    // Generate shopping list
    await user.click(screen.getByText('Generate Shopping List'));

    await waitFor(() => {
      expect(screen.getByText('Shopping List')).toBeInTheDocument();
    });

    // Verify workflow completion
    expect(screen.getByText('Menu planning completed!')).toBeInTheDocument();
  });
});
```

---

## Database Integration Testing

### Test Database Setup

#### Using Testcontainers for Real Database Testing
```csharp
public class DatabaseIntegrationTestBase : IAsyncLifetime
{
    private readonly MsSqlContainer _msSqlContainer;
    protected readonly string ConnectionString;

    public DatabaseIntegrationTestBase()
    {
        _msSqlContainer = new MsSqlBuilder()
            .WithPassword("StrongPassword123!")
            .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
            .Build();

        ConnectionString = _msSqlContainer.GetConnectionString();
    }

    public async Task InitializeAsync()
    {
        await _msSqlContainer.StartAsync();
        
        // Run migrations
        using var connection = new SqlConnection(ConnectionString);
        await connection.OpenAsync();
        
        // Apply database schema
        var options = new DbContextOptionsBuilder<MealPrepDbContext>()
            .UseSqlServer(ConnectionString)
            .Options;

        using var context = new MealPrepDbContext(options);
        await context.Database.MigrateAsync();
        
        // Seed test data
        await SeedTestDataAsync(context);
    }

    public async Task DisposeAsync()
    {
        await _msSqlContainer.DisposeAsync();
    }

    private async Task SeedTestDataAsync(MealPrepDbContext context)
    {
        // Add common test data
        var ingredients = new[]
        {
            new Ingredient { Name = "Chicken Breast", Category = "Protein" },
            new Ingredient { Name = "Rice", Category = "Grain" },
            new Ingredient { Name = "Broccoli", Category = "Vegetable" }
        };

        context.Ingredients.AddRange(ingredients);
        await context.SaveChangesAsync();
    }
}
```

### Migration Testing
```csharp
[Collection("DatabaseIntegration")]
public class DatabaseMigrationTests : DatabaseIntegrationTestBase
{
    [Fact]
    public async Task DatabaseMigrations_ApplySuccessfully()
    {
        // Arrange & Act - Migrations applied in InitializeAsync

        // Assert
        using var connection = new SqlConnection(ConnectionString);
        await connection.OpenAsync();

        // Verify tables exist
        var tableNames = new[] { "Users", "Recipes", "FamilyMembers", "Ingredients" };
        
        foreach (var tableName in tableNames)
        {
            var command = new SqlCommand(
                $"SELECT COUNT(*) FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = '{tableName}'", 
                connection);
            
            var count = (int)await command.ExecuteScalarAsync();
            count.Should().Be(1, $"Table {tableName} should exist");
        }
    }

    [Fact]
    public async Task DatabaseSchema_HasCorrectConstraints()
    {
        // Arrange
        var options = new DbContextOptionsBuilder<MealPrepDbContext>()
            .UseSqlServer(ConnectionString)
            .Options;

        using var context = new MealPrepDbContext(options);

        // Act & Assert - Try to violate constraints
        var user = new User
        {
            Email = "test@example.com",
            FirstName = "Test",
            LastName = "User",
            PasswordHash = "hash"
        };

        context.Users.Add(user);
        await context.SaveChangesAsync();

        // Try to add duplicate email
        var duplicateUser = new User
        {
            Email = "test@example.com", // Same email
            FirstName = "Another",
            LastName = "User",
            PasswordHash = "hash"
        };

        context.Users.Add(duplicateUser);

        // Should throw due to unique constraint
        await FluentActions.Invoking(() => context.SaveChangesAsync())
            .Should().ThrowAsync<DbUpdateException>();
    }
}
```

---

## Performance Integration Testing

### Load Testing with NBomber
```csharp
[Collection("PerformanceIntegration")]
public class ApiPerformanceIntegrationTests : IntegrationTestBase
{
    public ApiPerformanceIntegrationTests(MealPrepWebApplicationFactory factory) 
        : base(factory) { }

    [Fact]
    public async Task RecipeSearch_UnderLoad_MeetsPerformanceTargets()
    {
        // Arrange
        var token = await GetAuthTokenAsync();
        await SeedLargeRecipeDataset();

        var scenario = Scenario.Create("recipe_search", async context =>
        {
            var client = new HttpClient();
            client.DefaultRequestHeaders.Authorization = 
                new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);

            var response = await client.GetAsync($"{Factory.Server.BaseAddress}api/recipes/search?searchTerm=chicken");
            
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.InjectPerSec(rate: 10, during: TimeSpan.FromMinutes(1))
        );

        // Act
        var stats = NBomberRunner
            .RegisterScenarios(scenario)
            .Run();

        // Assert
        var scnStats = stats.AllScenarioStats.First();
        scnStats.Ok.Response.Mean.Should().BeLessOrEqualTo(TimeSpan.FromMilliseconds(500));
        scnStats.Ok.Response.Percentile95.Should().BeLessOrEqualTo(TimeSpan.FromMilliseconds(1000));
        scnStats.AllFailCount.Should().Be(0);
    }

    private async Task SeedLargeRecipeDataset()
    {
        var user = await DbContext.Users.FirstAsync(u => u.Email == "test@example.com");
        
        var recipes = Enumerable.Range(1, 1000).Select(i => new Recipe
        {
            Name = $"Recipe {i}",
            Description = $"Description for recipe {i}",
            Instructions = "Sample instructions",
            PrepTimeMinutes = Random.Shared.Next(5, 120),
            CookTimeMinutes = Random.Shared.Next(10, 180),
            ServingSize = Random.Shared.Next(1, 8),
            DifficultyLevel = new[] { "Easy", "Medium", "Hard" }[Random.Shared.Next(3)],
            Cuisine = new[] { "Italian", "Mexican", "Chinese", "American" }[Random.Shared.Next(4)],
            CreatedByUserId = user.Id,
            IsActive = true
        });

        DbContext.Recipes.AddRange(recipes);
        await DbContext.SaveChangesAsync();
    }
}
```

---

## Testing Best Practices

### Test Data Management
1. **Use Builders**: Create test data builders for complex objects
2. **Clean State**: Ensure each test starts with clean state
3. **Realistic Data**: Use realistic test data that represents actual usage
4. **Shared Fixtures**: Use shared fixtures for expensive setup operations

### Test Organization
1. **Arrange-Act-Assert**: Structure tests clearly with AAA pattern
2. **Descriptive Names**: Use descriptive test names that explain scenarios
3. **Single Responsibility**: Each test should verify one behavior
4. **Test Categories**: Group related tests using categories or collections

### Error Handling Testing
1. **Test Error Conditions**: Verify proper handling of error scenarios
2. **Validate Error Messages**: Ensure error messages are meaningful
3. **Test Recovery**: Verify system recovery from error states
4. **Boundary Conditions**: Test edge cases and boundary values

### Performance Considerations
1. **Parallel Execution**: Design tests to run in parallel when possible
2. **Resource Cleanup**: Properly dispose of resources to prevent leaks
3. **Selective Testing**: Use categories to run different test suites
4. **Test Isolation**: Ensure tests don't interfere with each other

---

*Last Updated: December 2024*  
*Integration testing guide continuously updated with new patterns and best practices*