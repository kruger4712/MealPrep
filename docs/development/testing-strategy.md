# Testing Strategy

## Overview
Comprehensive testing approach for the MealPrep application ensuring quality, reliability, and maintainability across all layers of the application stack.

## Testing Philosophy

### Goals
- **Quality Assurance**: Catch bugs and issues before they reach production
- **Regression Prevention**: Ensure changes don't break existing functionality
- **Documentation**: Tests serve as living documentation of system behavior
- **Confidence**: Enable rapid development with confidence in code changes
- **Performance**: Maintain system performance characteristics

### Principles
- **Test Pyramid**: More unit tests, fewer integration tests, minimal E2E tests
- **Test Early**: Write tests during development, not after
- **Test Fast**: Optimize for quick feedback cycles
- **Test Independently**: Tests should not depend on each other
- **Test Realistically**: Use realistic test data and scenarios

## Testing Pyramid

### 1. Unit Tests (70% of total tests)
**Purpose**: Test individual components in isolation
- **Speed**: < 1 second per test
- **Scope**: Methods, classes, pure functions
- **Tools**: MSTest (C#), Jest (TypeScript)
- **Coverage Target**: 80% minimum, 90% preferred

### 2. Integration Tests (20% of total tests)
**Purpose**: Test component interactions and external dependencies
- **Speed**: 1-10 seconds per test
- **Scope**: API endpoints, database operations, service integrations
- **Tools**: WebApplicationFactory, Testcontainers, React Testing Library
- **Coverage Target**: All API endpoints, critical data flows

### 3. End-to-End Tests (10% of total tests)
**Purpose**: Test complete user workflows
- **Speed**: 10-60 seconds per test
- **Scope**: Critical user journeys, UI interactions
- **Tools**: Playwright, Cypress
- **Coverage Target**: Primary user flows, critical business scenarios

## Backend Testing Strategy

### Unit Testing Standards

#### Service Layer Testing
```csharp
[TestClass]
public class MealSuggestionServiceTests
{
    private Mock<IGeminiAiService> _mockAiService;
    private Mock<IFamilyRepository> _mockFamilyRepository;
    private Mock<ILogger<MealSuggestionService>> _mockLogger;
    private MealSuggestionService _service;

    [TestInitialize]
    public void Setup()
    {
        _mockAiService = new Mock<IGeminiAiService>();
        _mockFamilyRepository = new Mock<IFamilyRepository>();
        _mockLogger = new Mock<ILogger<MealSuggestionService>>();
        
        _service = new MealSuggestionService(
            _mockAiService.Object,
            _mockFamilyRepository.Object,
            _mockLogger.Object);
    }

    [TestMethod]
    public async Task GenerateMealSuggestions_ValidFamily_ReturnsPersonalizedSuggestions()
    {
        // Arrange
        var familyId = 1;
        var family = TestDataFactory.CreateTestFamily();
        var request = new MealSuggestionRequest
        {
            FamilyId = familyId,
            MealType = MealType.Dinner,
            Budget = 25.00m,
            MaxPrepTime = 30
        };

        _mockFamilyRepository
            .Setup(x => x.GetByIdAsync(familyId))
            .ReturnsAsync(family);

        var expectedSuggestions = TestDataFactory.CreateTestSuggestions();
        _mockAiService
            .Setup(x => x.GenerateMealSuggestionsAsync(It.IsAny<AiMealRequest>()))
            .ReturnsAsync(expectedSuggestions);

        // Act
        var result = await _service.GenerateMealSuggestionsAsync(request);

        // Assert
        Assert.IsNotNull(result);
        Assert.AreEqual(3, result.Count);
        Assert.IsTrue(result.All(s => s.EstimatedCost <= request.Budget));
        Assert.IsTrue(result.All(s => s.PrepTime <= request.MaxPrepTime));
        Assert.IsTrue(result.All(s => s.FamilyFitScore >= 1 && s.FamilyFitScore <= 10));

        // Verify AI service was called with correct parameters
        _mockAiService.Verify(
            x => x.GenerateMealSuggestionsAsync(
                It.Is<AiMealRequest>(r => 
                    r.FamilyMembers.Count == family.Members.Count &&
                    r.MealType == request.MealType.ToString())),
            Times.Once);
    }

    [TestMethod]
    public async Task GenerateMealSuggestions_FamilyWithAllergies_ExcludesAllergenRecipes()
    {
        // Arrange
        var family = TestDataFactory.CreateFamilyWithAllergies(new[] { "nuts", "dairy" });
        var request = new MealSuggestionRequest { FamilyId = family.Id };

        _mockFamilyRepository
            .Setup(x => x.GetByIdAsync(family.Id))
            .ReturnsAsync(family);

        var suggestionsWithAllergens = TestDataFactory.CreateSuggestionsWithAllergens();
        _mockAiService
            .Setup(x => x.GenerateMealSuggestionsAsync(It.IsAny<AiMealRequest>()))
            .ReturnsAsync(suggestionsWithAllergens);

        // Act
        var result = await _service.GenerateMealSuggestionsAsync(request);

        // Assert
        Assert.IsNotNull(result);
        Assert.IsTrue(result.All(s => !s.ContainsAllergens(new[] { "nuts", "dairy" })));
        Assert.IsTrue(result.Count > 0, "Should return at least one allergen-free suggestion");
    }

    [TestMethod]
    public async Task GenerateMealSuggestions_AiServiceFailure_UsesGracefulFallback()
    {
        // Arrange
        var familyId = 1;
        var family = TestDataFactory.CreateTestFamily();
        var request = new MealSuggestionRequest { FamilyId = familyId };

        _mockFamilyRepository
            .Setup(x => x.GetByIdAsync(familyId))
            .ReturnsAsync(family);

        _mockAiService
            .Setup(x => x.GenerateMealSuggestionsAsync(It.IsAny<AiMealRequest>()))
            .ThrowsAsync(new AiServiceException("AI service unavailable"));

        // Act
        var result = await _service.GenerateMealSuggestionsAsync(request);

        // Assert
        Assert.IsNotNull(result);
        Assert.IsTrue(result.Count > 0, "Should return fallback suggestions");
        Assert.IsTrue(result.All(s => s.Source == "Fallback"), "All suggestions should be from fallback");
    }
}
```

#### Repository Testing with In-Memory Database
```csharp
[TestClass]
public class RecipeRepositoryTests
{
    private DbContextOptions<MealPrepDbContext> _dbContextOptions;
    private MealPrepDbContext _context;
    private RecipeRepository _repository;

    [TestInitialize]
    public void Setup()
    {
        _dbContextOptions = new DbContextOptionsBuilder<MealPrepDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;

        _context = new MealPrepDbContext(_dbContextOptions);
        _repository = new RecipeRepository(_context);

        SeedTestData();
    }

    [TestCleanup]
    public void Cleanup()
    {
        _context.Dispose();
    }

    [TestMethod]
    public async Task GetByIdAsync_ExistingRecipe_ReturnsRecipeWithIngredients()
    {
        // Arrange
        var recipeId = 1;

        // Act
        var result = await _repository.GetByIdAsync(recipeId);

        // Assert
        Assert.IsNotNull(result);
        Assert.AreEqual(recipeId, result.Id);
        Assert.AreEqual("Test Recipe", result.Name);
        Assert.IsTrue(result.Ingredients.Any());
        Assert.IsTrue(result.Instructions.Any());
    }

    [TestMethod]
    public async Task SearchAsync_ByIngredient_ReturnsMatchingRecipes()
    {
        // Arrange
        var searchTerm = "chicken";

        // Act
        var result = await _repository.SearchAsync(searchTerm);

        // Assert
        Assert.IsNotNull(result);
        Assert.IsTrue(result.Any());
        Assert.IsTrue(result.All(r => 
            r.Name.Contains(searchTerm, StringComparison.OrdinalIgnoreCase) ||
            r.Ingredients.Any(i => i.Name.Contains(searchTerm, StringComparison.OrdinalIgnoreCase))));
    }

    [TestMethod]
    public async Task CreateAsync_ValidRecipe_PersistsToDatabase()
    {
        // Arrange
        var recipe = TestDataFactory.CreateTestRecipe("New Recipe");

        // Act
        var result = await _repository.CreateAsync(recipe);
        await _context.SaveChangesAsync();

        // Assert
        Assert.IsNotNull(result);
        Assert.IsTrue(result.Id > 0);

        var savedRecipe = await _context.Recipes
            .Include(r => r.Ingredients)
            .FirstOrDefaultAsync(r => r.Id == result.Id);
        
        Assert.IsNotNull(savedRecipe);
        Assert.AreEqual(recipe.Name, savedRecipe.Name);
        Assert.AreEqual(recipe.Ingredients.Count, savedRecipe.Ingredients.Count);
    }

    private void SeedTestData()
    {
        var recipes = new List<Recipe>
        {
            TestDataFactory.CreateTestRecipe("Test Recipe"),
            TestDataFactory.CreateTestRecipe("Chicken Parmesan"),
            TestDataFactory.CreateTestRecipe("Vegetarian Pasta")
        };

        _context.Recipes.AddRange(recipes);
        _context.SaveChanges();
    }
}
```

### Integration Testing

#### API Controller Testing
```csharp
[TestClass]
public class RecipesControllerIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;

    public RecipesControllerIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace with test database
                services.RemoveDbContext<MealPrepDbContext>();
                services.AddDbContext<MealPrepDbContext>(options =>
                {
                    options.UseInMemoryDatabase("TestDb");
                });

                // Replace AI service with mock
                services.Replace(ServiceDescriptor.Scoped<IGeminiAiService, MockGeminiAiService>());
            });
        });

        _client = _factory.CreateClient();
    }

    [TestMethod]
    public async Task GetRecipes_WithAuthentication_ReturnsRecipeList()
    {
        // Arrange
        var token = await GetAuthTokenAsync();
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);

        // Act
        var response = await _client.GetAsync("/api/v1/recipes");

        // Assert
        response.EnsureSuccessStatusCode();
        Assert.AreEqual("application/json", response.Content.Headers.ContentType?.MediaType);
        
        var content = await response.Content.ReadAsStringAsync();
        var recipes = JsonSerializer.Deserialize<ApiResponse<List<RecipeDto>>>(content);
        
        Assert.IsNotNull(recipes);
        Assert.IsNotNull(recipes.Data);
    }

    [TestMethod]
    public async Task CreateRecipe_ValidData_ReturnsCreatedRecipe()
    {
        // Arrange
        var token = await GetAuthTokenAsync();
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);

        var createRequest = new CreateRecipeRequest
        {
            Name = "Integration Test Recipe",
            Description = "Test recipe for integration testing",
            Instructions = "Mix ingredients and cook",
            PrepTimeMinutes = 15,
            CookTimeMinutes = 30,
            Servings = 4,
            Ingredients = new List<CreateRecipeIngredientRequest>
            {
                new CreateRecipeIngredientRequest
                {
                    Name = "Test Ingredient",
                    Quantity = 1,
                    Unit = "cup"
                }
            }
        };

        var json = JsonSerializer.Serialize(createRequest);
        var content = new StringContent(json, Encoding.UTF8, "application/json");

        // Act
        var response = await _client.PostAsync("/api/v1/recipes", content);

        // Assert
        Assert.AreEqual(HttpStatusCode.Created, response.StatusCode);
        
        var responseContent = await response.Content.ReadAsStringAsync();
        var createdRecipe = JsonSerializer.Deserialize<ApiResponse<RecipeDto>>(responseContent);
        
        Assert.IsNotNull(createdRecipe);
        Assert.AreEqual(createRequest.Name, createdRecipe.Data.Name);
        Assert.IsTrue(createdRecipe.Data.Id > 0);
    }

    [TestMethod]
    public async Task GetRecipes_WithoutAuthentication_ReturnsUnauthorized()
    {
        // Act
        var response = await _client.GetAsync("/api/v1/recipes");

        // Assert
        Assert.AreEqual(HttpStatusCode.Unauthorized, response.StatusCode);
    }

    private async Task<string> GetAuthTokenAsync()
    {
        var loginRequest = new LoginRequest
        {
            Email = "test@example.com",
            Password = "TestPassword123!"
        };

        var json = JsonSerializer.Serialize(loginRequest);
        var content = new StringContent(json, Encoding.UTF8, "application/json");

        var response = await _client.PostAsync("/api/v1/auth/login", content);
        response.EnsureSuccessStatusCode();

        var responseContent = await response.Content.ReadAsStringAsync();
        var authResponse = JsonSerializer.Deserialize<AuthResponse>(responseContent);

        return authResponse.Token;
    }
}
```

## Frontend Testing Strategy

### Component Testing with React Testing Library

#### React Component Tests
```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';
import { MealPlanner } from './MealPlanner';
import { mockFamilyMembers, mockMealSuggestions } from '../__mocks__/testData';
import * as aiService from '../services/aiService';

// Mock the AI service
jest.mock('../services/aiService');
const mockAiService = aiService as jest.Mocked<typeof aiService>;

const createTestQueryClient = () =>
  new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });

const renderWithProviders = (component: React.ReactElement) => {
  const queryClient = createTestQueryClient();
  return render(
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        {component}
      </BrowserRouter>
    </QueryClientProvider>
  );
};

describe('MealPlanner Component', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('renders meal planner with family members', () => {
    renderWithProviders(
      <MealPlanner
        familyMembers={mockFamilyMembers}
        onSuggestionSelect={jest.fn()}
      />
    );

    expect(screen.getByText('AI Meal Suggestions')).toBeInTheDocument();
    expect(screen.getByText('Family Members: 2')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /generate suggestions/i })).toBeInTheDocument();
  });

  test('shows loading state while generating suggestions', async () => {
    // Mock API delay
    mockAiService.getMealSuggestions.mockImplementation(
      () => new Promise(resolve => setTimeout(() => resolve(mockMealSuggestions), 100))
    );

    renderWithProviders(
      <MealPlanner
        familyMembers={mockFamilyMembers}
        onSuggestionSelect={jest.fn()}
      />
    );

    const generateButton = screen.getByRole('button', { name: /generate suggestions/i });
    fireEvent.click(generateButton);

    expect(screen.getByText('Generating meal suggestions...')).toBeInTheDocument();
    expect(screen.getByRole('progressbar')).toBeInTheDocument();

    await waitFor(() => {
      expect(screen.queryByText('Generating meal suggestions...')).not.toBeInTheDocument();
    });
  });

  test('displays meal suggestions after generation', async () => {
    mockAiService.getMealSuggestions.mockResolvedValue(mockMealSuggestions);

    renderWithProviders(
      <MealPlanner
        familyMembers={mockFamilyMembers}
        onSuggestionSelect={jest.fn()}
      />
    );

    const generateButton = screen.getByRole('button', { name: /generate suggestions/i });
    fireEvent.click(generateButton);

    await waitFor(() => {
      expect(screen.getByText('Chicken Parmesan')).toBeInTheDocument();
      expect(screen.getByText('Family Fit: 9/10')).toBeInTheDocument();
      expect(screen.getByText('$18.50')).toBeInTheDocument();
    });
  });

  test('handles API errors gracefully', async () => {
    const consoleError = jest.spyOn(console, 'error').mockImplementation(() => {});
    mockAiService.getMealSuggestions.mockRejectedValue(new Error('API Error'));

    renderWithProviders(
      <MealPlanner
        familyMembers={mockFamilyMembers}
        onSuggestionSelect={jest.fn()}
      />
    );

    const generateButton = screen.getByRole('button', { name: /generate suggestions/i });
    fireEvent.click(generateButton);

    await waitFor(() => {
      expect(screen.getByText('Failed to generate meal suggestions. Please try again.')).toBeInTheDocument();
    });

    consoleError.mockRestore();
  });

  test('meets accessibility requirements', () => {
    renderWithProviders(
      <MealPlanner
        familyMembers={mockFamilyMembers}
        onSuggestionSelect={jest.fn()}
      />
    );

    // Check for proper ARIA labels
    expect(screen.getByLabelText('Generate AI meal suggestions')).toBeInTheDocument();
    
    // Check for proper heading hierarchy
    expect(screen.getByRole('heading', { level: 2 })).toHaveTextContent('AI Meal Suggestions');
    
    // Check for keyboard navigation
    const generateButton = screen.getByRole('button', { name: /generate suggestions/i });
    expect(generateButton).toHaveAttribute('tabIndex', '0');
  });
});
```

#### Custom Hook Testing
```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useAiSuggestions } from './useAiSuggestions';
import { mockFamilyMembers } from '../__mocks__/testData';
import * as aiService from '../services/aiService';

jest.mock('../services/aiService');
const mockAiService = aiService as jest.Mocked<typeof aiService>;

const createWrapper = ({ children }: { children: React.ReactNode }) => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
};

describe('useAiSuggestions Hook', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('generates suggestions successfully', async () => {
    const mockSuggestions = [
      { id: '1', name: 'Test Recipe', familyFitScore: 8 }
    ];
    mockAiService.getMealSuggestions.mockResolvedValue(mockSuggestions);

    const { result } = renderHook(
      () => useAiSuggestions(mockFamilyMembers),
      { wrapper: createWrapper }
    );

    expect(result.current.isGenerating).toBe(false);
    expect(result.current.suggestions).toBeUndefined();

    await result.current.generateSuggestions({
      familyMembers: mockFamilyMembers,
      mealType: 'dinner',
      budget: 25
    });

    await waitFor(() => {
      expect(result.current.suggestions).toEqual(mockSuggestions);
      expect(result.current.isGenerating).toBe(false);
    });
  });
});
```

## End-to-End Testing

### Playwright E2E Tests
```typescript
import { test, expect } from '@playwright/test';

test.describe('Meal Planning Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login before each test
    await page.goto('/login');
    await page.fill('[data-testid="email"]', 'test@example.com');
    await page.fill('[data-testid="password"]', 'TestPassword123!');
    await page.click('[data-testid="login-button"]');
    await page.waitForURL('/dashboard');
  });

  test('complete meal planning journey', async ({ page }) => {
    // Navigate to meal planner
    await page.click('[data-testid="meal-planner-link"]');
    await page.waitForURL('/meal-planner');

    // Verify family members are loaded
    await expect(page.locator('[data-testid="family-member"]')).toHaveCount(2);

    // Set meal parameters
    await page.selectOption('[data-testid="meal-type"]', 'dinner');
    await page.fill('[data-testid="budget"]', '25');
    await page.fill('[data-testid="max-prep-time"]', '30');

    // Generate AI suggestions
    await page.click('[data-testid="generate-suggestions"]');

    // Wait for suggestions to load
    await page.waitForSelector('[data-testid="meal-suggestion"]');
    await expect(page.locator('[data-testid="meal-suggestion"]')).toHaveCount(3);

    // Select first suggestion
    const firstSuggestion = page.locator('[data-testid="meal-suggestion"]').first();
    await expect(firstSuggestion.locator('[data-testid="family-fit-score"]')).toBeVisible();
    await firstSuggestion.locator('[data-testid="add-to-plan"]').click();

    // Verify success message
    await expect(page.locator('[data-testid="success-message"]')).toBeVisible();
    await expect(page.locator('[data-testid="success-message"]')).toContainText('Added to meal plan');

    // Navigate to weekly planner
    await page.click('[data-testid="view-weekly-plan"]');
    await page.waitForURL('/weekly-planner');

    // Verify meal appears in calendar
    await expect(page.locator('[data-testid="planned-meal"]')).toBeVisible();
  });

  test('mobile responsive behavior', async ({ page }) => {
    // Set mobile viewport
    await page.setViewportSize({ width: 375, height: 667 });

    await page.goto('/dashboard');

    // Verify mobile navigation
    await expect(page.locator('[data-testid="mobile-nav"]')).toBeVisible();
    await expect(page.locator('[data-testid="desktop-nav"]')).not.toBeVisible();

    // Test hamburger menu
    await page.click('[data-testid="mobile-menu-toggle"]');
    await expect(page.locator('[data-testid="mobile-menu"]')).toBeVisible();
  });
});

test.describe('Accessibility', () => {
  test('keyboard navigation works throughout app', async ({ page }) => {
    await page.goto('/dashboard');

    // Test tab navigation
    await page.keyboard.press('Tab');
    await expect(page.locator(':focus')).toBeVisible();

    // Navigate through multiple elements
    for (let i = 0; i < 5; i++) {
      await page.keyboard.press('Tab');
      await expect(page.locator(':focus')).toBeVisible();
    }

    // Test Enter key activation
    await page.keyboard.press('Enter');
    // Verify appropriate action occurred
  });
});
```

## Test Data Management

### Test Data Factory
```csharp
public static class TestDataFactory
{
    public static Family CreateTestFamily(int memberCount = 2)
    {
        return new Family
        {
            Id = Random.Shared.Next(1, 1000),
            Name = $"Test Family {Guid.NewGuid().ToString()[..8]}",
            Members = Enumerable.Range(1, memberCount)
                .Select(i => CreateFamilyMember($"Member {i}"))
                .ToList()
        };
    }

    public static FamilyMember CreateFamilyMember(string name)
    {
        return new FamilyMember
        {
            Id = Random.Shared.Next(1, 1000),
            Name = name,
            Age = Random.Shared.Next(5, 65),
            DietaryRestrictions = new[] { DietaryRestriction.None },
            LikedIngredients = new[] { "pasta", "chicken" },
            DislikedIngredients = new[] { "seafood" },
            SpiceToleranceLevel = Random.Shared.Next(1, 10)
        };
    }

    public static Recipe CreateTestRecipe(string name = null)
    {
        return new Recipe
        {
            Id = Random.Shared.Next(1, 1000),
            Name = name ?? $"Test Recipe {Guid.NewGuid().ToString()[..8]}",
            Description = "A delicious test recipe",
            Instructions = "Mix ingredients and cook",
            PrepTimeMinutes = 15,
            CookTimeMinutes = 30,
            Servings = 4,
            CreatedByUserId = 1,
            Ingredients = new List<RecipeIngredient>
            {
                new RecipeIngredient
                {
                    IngredientName = "Test Ingredient",
                    Quantity = 1,
                    Unit = "cup"
                }
            }
        };
    }

    public static List<MealSuggestion> CreateTestSuggestions()
    {
        return new List<MealSuggestion>
        {
            new MealSuggestion
            {
                Id = "suggestion_1",
                Name = "Chicken Parmesan",
                FamilyFitScore = 9,
                EstimatedCost = 18.50m,
                PrepTime = 20,
                CookTime = 25
            },
            new MealSuggestion
            {
                Id = "suggestion_2", 
                Name = "Vegetarian Pasta",
                FamilyFitScore = 7,
                EstimatedCost = 12.00m,
                PrepTime = 15,
                CookTime = 20
            }
        };
    }
}
```

## Test Coverage Requirements

### Coverage Targets
- **Unit Tests**: 80% minimum, 90% target
- **Integration Tests**: 70% of API endpoints
- **E2E Tests**: 100% of critical user journeys

### Coverage Exclusions
- Third-party libraries
- Configuration files
- Database migrations
- Auto-generated code
- DTOs and simple models

### Reporting Configuration
```xml
<!-- coverlet.runsettings -->
<?xml version="1.0" encoding="utf-8" ?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat code coverage">
        <Configuration>
          <Format>cobertura</Format>
          <Exclude>[*.Tests]*,[*.IntegrationTests]*</Exclude>
          <ExcludeByAttribute>Obsolete,GeneratedCodeAttribute,CompilerGeneratedAttribute</ExcludeByAttribute>
          <IncludeDirectory>src</IncludeDirectory>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

## Continuous Integration

### GitHub Actions Workflow
```yaml
name: Test Suite

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x
        
    - name: Restore dependencies
      run: dotnet restore
      
    - name: Build
      run: dotnet build --no-restore
      
    - name: Run unit tests
      run: dotnet test --no-build --verbosity normal --collect:"XPlat Code Coverage" --settings coverlet.runsettings
      
    - name: Upload coverage reports
      uses: codecov/codecov-action@v3

  integration-tests:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: mealprep_test
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

    steps:
    - uses: actions/checkout@v3
    
    - name: Run integration tests
      run: dotnet test --filter Category=Integration
      env:
        ConnectionStrings__DefaultConnection: "Server=localhost;Database=mealprep_test;Uid=root;Pwd=root;"

  frontend-tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: meal-prep-frontend/package-lock.json
        
    - name: Install dependencies
      run: npm ci
      working-directory: meal-prep-frontend
      
    - name: Run tests
      run: npm test -- --coverage --watchAll=false
      working-directory: meal-prep-frontend

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        
    - name: Install Playwright
      run: npx playwright install --with-deps
      
    - name: Run E2E tests
      run: npx playwright test
      working-directory: meal-prep-frontend
```

## Quality Gates

### Definition of Done
- [ ] All unit tests pass
- [ ] Code coverage meets threshold (80%)
- [ ] Integration tests pass
- [ ] No critical security vulnerabilities
- [ ] Performance tests meet requirements
- [ ] Accessibility tests pass
- [ ] Code review completed

### Performance Testing
- API response times < 200ms (95th percentile)
- Frontend render times < 2 seconds
- Database queries < 100ms
- Memory usage within acceptable limits

This comprehensive testing strategy ensures the MealPrep application maintains high quality, reliability, and performance across all components and user interactions.

*This document should be updated as testing practices evolve and new tools are adopted.*