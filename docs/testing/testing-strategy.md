# Testing Strategy Guide

## Overview
Comprehensive testing strategy for the MealPrep application covering unit testing, integration testing, end-to-end testing, API testing, performance testing, and security testing across the full-stack .NET Core and React application.

## Testing Philosophy

### Testing Pyramid Strategy
```
                    ???????????????????
                    ?   E2E Tests     ? ? Few, High-Level, UI-driven
                    ?   (Cypress)     ?
                    ???????????????????
               ???????????????????????????????
               ?    Integration Tests        ? ? Some, API + DB + Services
               ?  (xUnit + TestContainers)   ?
               ???????????????????????????????
        ???????????????????????????????????????????????
        ?              Unit Tests                     ? ? Many, Fast, Isolated
        ?          (xUnit + Jest/Vitest)              ?
        ???????????????????????????????????????????????
```

### Testing Goals
- **Quality Assurance**: Ensure features work as expected
- **Regression Prevention**: Catch breaking changes early
- **Documentation**: Tests serve as living documentation
- **Confidence**: Enable safe refactoring and deployment
- **Performance**: Maintain application speed and reliability

### Testing Metrics Targets
- **Code Coverage**: 80%+ for critical business logic
- **Test Execution Time**: Unit tests < 5 minutes, Integration < 15 minutes
- **Test Reliability**: < 1% flaky test rate
- **Bug Detection**: 90%+ of bugs caught before production

## Backend Testing (.NET Core)

### Unit Testing with xUnit

#### Test Project Structure
```
Tests/
??? MealPrep.Domain.Tests/           # Domain logic tests
??? MealPrep.Application.Tests/      # Application service tests  
??? MealPrep.Infrastructure.Tests/   # Infrastructure tests
??? MealPrep.API.Tests/             # API controller tests
??? MealPrep.Tests.Shared/          # Common test utilities
```

#### Unit Test Examples
```csharp
// Tests/MealPrep.Domain.Tests/RecipeTests.cs
public class RecipeTests
{
    [Fact]
    public void Recipe_CreateWithValidData_ShouldSucceed()
    {
        // Arrange
        var recipe = new Recipe
        {
            Name = "Test Recipe",
            PrepTime = 15,
            CookTime = 30,
            Servings = 4
        };

        // Act & Assert
        Assert.Equal("Test Recipe", recipe.Name);
        Assert.Equal(45, recipe.TotalTime);
        Assert.True(recipe.IsValid());
    }

    [Theory]
    [InlineData(-1)]
    [InlineData(0)]
    public void Recipe_WithInvalidPrepTime_ShouldFail(int invalidPrepTime)
    {
        // Arrange & Act
        var recipe = new Recipe { PrepTime = invalidPrepTime };

        // Assert
        Assert.False(recipe.IsValid());
    }

    [Fact]
    public void Recipe_CalculateNutrition_ShouldSumIngredients()
    {
        // Arrange
        var recipe = new Recipe();
        recipe.AddIngredient(new Ingredient { Calories = 100, Protein = 10 });
        recipe.AddIngredient(new Ingredient { Calories = 50, Protein = 5 });

        // Act
        var nutrition = recipe.CalculateNutrition();

        // Assert
        Assert.Equal(150, nutrition.TotalCalories);
        Assert.Equal(15, nutrition.TotalProtein);
    }
}

// Tests/MealPrep.Application.Tests/RecipeServiceTests.cs
public class RecipeServiceTests
{
    private readonly Mock<IRecipeRepository> _mockRepository;
    private readonly Mock<INutritionService> _mockNutritionService;
    private readonly RecipeService _recipeService;

    public RecipeServiceTests()
    {
        _mockRepository = new Mock<IRecipeRepository>();
        _mockNutritionService = new Mock<INutritionService>();
        _recipeService = new RecipeService(_mockRepository.Object, _mockNutritionService.Object);
    }

    [Fact]
    public async Task CreateRecipeAsync_WithValidRecipe_ShouldReturnCreatedRecipe()
    {
        // Arrange
        var createRecipeDto = new CreateRecipeDto
        {
            Name = "Test Recipe",
            PrepTime = 15,
            CookTime = 30
        };

        var expectedRecipe = new Recipe
        {
            Id = Guid.NewGuid(),
            Name = createRecipeDto.Name,
            PrepTime = createRecipeDto.PrepTime,
            CookTime = createRecipeDto.CookTime
        };

        _mockRepository
            .Setup(r => r.CreateAsync(It.IsAny<Recipe>()))
            .ReturnsAsync(expectedRecipe);

        // Act
        var result = await _recipeService.CreateRecipeAsync(createRecipeDto);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(expectedRecipe.Name, result.Name);
        _mockRepository.Verify(r => r.CreateAsync(It.IsAny<Recipe>()), Times.Once);
    }

    [Fact]
    public async Task GetRecipesByDietaryRestrictions_ShouldFilterCorrectly()
    {
        // Arrange
        var restrictions = new List<DietaryRestriction> { DietaryRestriction.Vegetarian };
        var allRecipes = new List<Recipe>
        {
            new Recipe { Name = "Veg Recipe", DietaryRestrictions = new[] { DietaryRestriction.Vegetarian } },
            new Recipe { Name = "Meat Recipe", DietaryRestrictions = new DietaryRestriction[0] }
        };

        _mockRepository
            .Setup(r => r.GetByDietaryRestrictionsAsync(restrictions))
            .ReturnsAsync(allRecipes.Where(r => r.DietaryRestrictions.Any(d => restrictions.Contains(d))));

        // Act
        var result = await _recipeService.GetRecipesByDietaryRestrictionsAsync(restrictions);

        // Assert
        Assert.Single(result);
        Assert.Contains("Veg Recipe", result.First().Name);
    }
}
```

#### Test Utilities and Helpers
```csharp
// Tests/MealPrep.Tests.Shared/TestDataBuilder.cs
public class TestDataBuilder
{
    public static Recipe CreateValidRecipe(string name = "Test Recipe")
    {
        return new Recipe
        {
            Id = Guid.NewGuid(),
            Name = name,
            PrepTime = 15,
            CookTime = 30,
            Servings = 4,
            Ingredients = new List<RecipeIngredient>
            {
                new RecipeIngredient
                {
                    Ingredient = new Ingredient { Name = "Test Ingredient" },
                    Quantity = 1,
                    Unit = "cup"
                }
            },
            Instructions = new List<string> { "Test instruction" }
        };
    }

    public static User CreateValidUser(string email = "test@example.com")
    {
        return new User
        {
            Id = Guid.NewGuid(),
            Email = email,
            FirstName = "Test",
            LastName = "User",
            CreatedAt = DateTime.UtcNow
        };
    }

    public static Family CreateValidFamily(int memberCount = 2)
    {
        var family = new Family
        {
            Id = Guid.NewGuid(),
            Name = "Test Family",
            CreatedAt = DateTime.UtcNow
        };

        for (int i = 0; i < memberCount; i++)
        {
            family.Members.Add(new FamilyMember
            {
                Name = $"Member {i + 1}",
                Age = 20 + i * 10,
                DietaryRestrictions = new List<DietaryRestriction>()
            });
        }

        return family;
    }
}

// Tests/MealPrep.Tests.Shared/DatabaseFixture.cs
public class DatabaseFixture : IDisposable
{
    public MealPrepDbContext Context { get; private set; }

    public DatabaseFixture()
    {
        var options = new DbContextOptionsBuilder<MealPrepDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;

        Context = new MealPrepDbContext(options);
        Context.Database.EnsureCreated();
    }

    public void Dispose()
    {
        Context?.Dispose();
    }
}
```

### Integration Testing

#### Integration Test Setup
```csharp
// Tests/MealPrep.API.Tests/IntegrationTestBase.cs
public abstract class IntegrationTestBase : IClassFixture<WebApplicationFactory<Program>>
{
    protected readonly HttpClient Client;
    protected readonly WebApplicationFactory<Program> Factory;
    protected readonly MealPrepDbContext Context;

    protected IntegrationTestBase(WebApplicationFactory<Program> factory)
    {
        Factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Remove the app DbContext registration
                var descriptor = services.SingleOrDefault(
                    d => d.ServiceType == typeof(DbContextOptions<MealPrepDbContext>));
                if (descriptor != null)
                    services.Remove(descriptor);

                // Add DbContext using test database
                services.AddDbContext<MealPrepDbContext>(options =>
                {
                    options.UseInMemoryDatabase("TestDb");
                });

                // Override external dependencies with mocks
                services.AddSingleton<IGeminiAIService, MockGeminiAIService>();
            });
        });

        Client = Factory.CreateClient();
        Context = Factory.Services.CreateScope().ServiceProvider
            .GetRequiredService<MealPrepDbContext>();
    }

    protected async Task<string> GetJwtTokenAsync(User user = null)
    {
        user ??= TestDataBuilder.CreateValidUser();
        Context.Users.Add(user);
        await Context.SaveChangesAsync();

        var tokenService = Factory.Services.GetRequiredService<ITokenService>();
        return tokenService.GenerateToken(user);
    }

    protected void SetAuthorizationHeader(string token)
    {
        Client.DefaultRequestHeaders.Authorization = 
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
    }
}

// Tests/MealPrep.API.Tests/Controllers/RecipesControllerTests.cs
public class RecipesControllerTests : IntegrationTestBase
{
    public RecipesControllerTests(WebApplicationFactory<Program> factory) : base(factory) { }

    [Fact]
    public async Task GetRecipes_WithoutAuth_ShouldReturn401()
    {
        // Act
        var response = await Client.GetAsync("/api/recipes");

        // Assert
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }

    [Fact]
    public async Task GetRecipes_WithAuth_ShouldReturnRecipes()
    {
        // Arrange
        var user = TestDataBuilder.CreateValidUser();
        var token = await GetJwtTokenAsync(user);
        SetAuthorizationHeader(token);

        var recipe = TestDataBuilder.CreateValidRecipe();
        recipe.UserId = user.Id;
        Context.Recipes.Add(recipe);
        await Context.SaveChangesAsync();

        // Act
        var response = await Client.GetAsync("/api/recipes");

        // Assert
        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadAsStringAsync();
        var recipes = JsonSerializer.Deserialize<List<RecipeDto>>(content);
        
        Assert.Single(recipes);
        Assert.Equal(recipe.Name, recipes[0].Name);
    }

    [Fact]
    public async Task CreateRecipe_WithValidData_ShouldReturnCreated()
    {
        // Arrange
        var user = TestDataBuilder.CreateValidUser();
        var token = await GetJwtTokenAsync(user);
        SetAuthorizationHeader(token);

        var createRecipeDto = new CreateRecipeDto
        {
            Name = "Integration Test Recipe",
            PrepTime = 15,
            CookTime = 30,
            Servings = 4,
            Instructions = new List<string> { "Test instruction" }
        };

        var json = JsonSerializer.Serialize(createRecipeDto);
        var content = new StringContent(json, Encoding.UTF8, "application/json");

        // Act
        var response = await Client.PostAsync("/api/recipes", content);

        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        
        var responseContent = await response.Content.ReadAsStringAsync();
        var createdRecipe = JsonSerializer.Deserialize<RecipeDto>(responseContent);
        
        Assert.Equal(createRecipeDto.Name, createdRecipe.Name);
        Assert.NotEqual(Guid.Empty, createdRecipe.Id);
    }
}
```

## Frontend Testing (React)

### Unit Testing with Vitest and React Testing Library

#### Test Configuration
```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/test/',
        '**/*.d.ts',
        '**/*.config.*',
        '**/coverage/**'
      ]
    }
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
})

// src/test/setup.ts
import '@testing-library/jest-dom'
import { cleanup } from '@testing-library/react'
import { afterEach, vi } from 'vitest'

// Mock window.matchMedia
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: vi.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
})

// Cleanup after each test
afterEach(() => {
  cleanup()
})
```

#### Component Testing Examples
```typescript
// src/components/RecipeCard/RecipeCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { BrowserRouter } from 'react-router-dom'
import { RecipeCard } from './RecipeCard'
import { Recipe } from '@/types/recipe'

const mockRecipe: Recipe = {
  id: '1',
  name: 'Test Recipe',
  description: 'A delicious test recipe',
  prepTime: 15,
  cookTime: 30,
  servings: 4,
  difficulty: 'Easy',
  imageUrl: 'https://example.com/image.jpg',
  ingredients: [
    { name: 'Test Ingredient', quantity: 1, unit: 'cup' }
  ],
  instructions: ['Mix ingredients', 'Cook for 30 minutes'],
  nutrition: {
    calories: 250,
    protein: 15,
    carbs: 30,
    fat: 8
  },
  tags: ['quick', 'easy'],
  userId: 'user1',
  createdAt: new Date().toISOString(),
  updatedAt: new Date().toISOString()
}

const TestWrapper: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  })

  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        {children}
      </BrowserRouter>
    </QueryClientProvider>
  )
}

describe('RecipeCard', () => {
  const mockOnClick = vi.fn()

  beforeEach(() => {
    mockOnClick.mockClear()
  })

  it('renders recipe information correctly', () => {
    render(
      <TestWrapper>
        <RecipeCard recipe={mockRecipe} onClick={mockOnClick} />
      </TestWrapper>
    )

    expect(screen.getByText('Test Recipe')).toBeInTheDocument()
    expect(screen.getByText('A delicious test recipe')).toBeInTheDocument()
    expect(screen.getByText('15m prep')).toBeInTheDocument()
    expect(screen.getByText('30m cook')).toBeInTheDocument()
    expect(screen.getByText('4 servings')).toBeInTheDocument()
    expect(screen.getByText('Easy')).toBeInTheDocument()
  })

  it('displays recipe image with correct alt text', () => {
    render(
      <TestWrapper>
        <RecipeCard recipe={mockRecipe} onClick={mockOnClick} />
      </TestWrapper>
    )

    const image = screen.getByAltText('Test Recipe - A delicious test recipe')
    expect(image).toBeInTheDocument()
    expect(image).toHaveAttribute('src', mockRecipe.imageUrl)
  })

  it('calls onClick when card is clicked', () => {
    render(
      <TestWrapper>
        <RecipeCard recipe={mockRecipe} onClick={mockOnClick} />
      </TestWrapper>
    )

    const card = screen.getByRole('article')
    fireEvent.click(card)

    expect(mockOnClick).toHaveBeenCalledWith(mockRecipe)
  })

  it('shows difficulty badge with correct styling', () => {
    render(
      <TestWrapper>
        <RecipeCard recipe={mockRecipe} onClick={mockOnClick} />
      </TestWrapper>
    )

    const difficultyBadge = screen.getByText('Easy')
    expect(difficultyBadge).toHaveClass('bg-green-100', 'text-green-800')
  })

  it('renders tags correctly', () => {
    render(
      <TestWrapper>
        <RecipeCard recipe={mockRecipe} onClick={mockOnClick} />
      </TestWrapper>
    )

    expect(screen.getByText('quick')).toBeInTheDocument()
    expect(screen.getByText('easy')).toBeInTheDocument()
  })
})
```

#### Hook Testing
```typescript
// src/hooks/useRecipes/useRecipes.test.ts
import { renderHook, waitFor } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { useRecipes } from './useRecipes'
import { recipeService } from '@/services/recipeService'

// Mock the recipe service
vi.mock('@/services/recipeService')

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  })

  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}

describe('useRecipes', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('fetches recipes successfully', async () => {
    const mockRecipes = [
      { id: '1', name: 'Recipe 1' },
      { id: '2', name: 'Recipe 2' }
    ]

    vi.mocked(recipeService.getRecipes).mockResolvedValue(mockRecipes)

    const { result } = renderHook(() => useRecipes(), {
      wrapper: createWrapper()
    })

    expect(result.current.isLoading).toBe(true)

    await waitFor(() => {
      expect(result.current.isLoading).toBe(false)
    })

    expect(result.current.data).toEqual(mockRecipes)
    expect(result.current.error).toBeNull()
  })

  it('handles recipe fetch error', async () => {
    const mockError = new Error('Failed to fetch recipes')
    vi.mocked(recipeService.getRecipes).mockRejectedValue(mockError)

    const { result } = renderHook(() => useRecipes(), {
      wrapper: createWrapper()
    })

    await waitFor(() => {
      expect(result.current.isLoading).toBe(false)
    })

    expect(result.current.error).toBe(mockError)
    expect(result.current.data).toBeUndefined()
  })

  it('refetches data when invalidated', async () => {
    const mockRecipes = [{ id: '1', name: 'Recipe 1' }]
    vi.mocked(recipeService.getRecipes).mockResolvedValue(mockRecipes)

    const { result } = renderHook(() => useRecipes(), {
      wrapper: createWrapper()
    })

    await waitFor(() => {
      expect(result.current.isLoading).toBe(false)
    })

    // Simulate refetch
    result.current.refetch()

    expect(vi.mocked(recipeService.getRecipes)).toHaveBeenCalledTimes(2)
  })
})
```

## End-to-End Testing

### Cypress E2E Tests
```typescript
// cypress/e2e/recipe-management.cy.ts
describe('Recipe Management', () => {
  beforeEach(() => {
    // Mock API calls
    cy.intercept('GET', '/api/auth/me', { fixture: 'user.json' })
    cy.intercept('GET', '/api/recipes', { fixture: 'recipes.json' })
    cy.intercept('POST', '/api/recipes', { fixture: 'created-recipe.json' })

    // Login
    cy.login('test@example.com', 'password')
    cy.visit('/recipes')
  })

  it('should display recipe list', () => {
    cy.get('[data-cy=recipe-card]').should('have.length.gte', 1)
    cy.get('[data-cy=recipe-title]').first().should('contain.text', 'Chicken Alfredo')
  })

  it('should create a new recipe', () => {
    cy.get('[data-cy=create-recipe-btn]').click()
    
    // Fill out recipe form
    cy.get('[data-cy=recipe-name-input]').type('Test Recipe')
    cy.get('[data-cy=recipe-description-input]').type('A test recipe for e2e testing')
    cy.get('[data-cy=prep-time-input]').type('15')
    cy.get('[data-cy=cook-time-input]').type('30')
    cy.get('[data-cy=servings-input]').type('4')
    
    // Add ingredients
    cy.get('[data-cy=add-ingredient-btn]').click()
    cy.get('[data-cy=ingredient-name-input]').first().type('Test Ingredient')
    cy.get('[data-cy=ingredient-quantity-input]').first().type('1')
    cy.get('[data-cy=ingredient-unit-select]').first().select('cup')
    
    // Add instructions
    cy.get('[data-cy=add-instruction-btn]').click()
    cy.get('[data-cy=instruction-input]').first().type('Mix all ingredients together')
    
    // Submit form
    cy.get('[data-cy=save-recipe-btn]').click()
    
    // Verify redirect and success message
    cy.url().should('include', '/recipes')
    cy.get('[data-cy=success-message]').should('contain.text', 'Recipe created successfully')
  })

  it('should search and filter recipes', () => {
    // Test search functionality
    cy.get('[data-cy=recipe-search-input]').type('chicken')
    cy.get('[data-cy=recipe-card]').should('contain.text', 'Chicken')
    
    // Clear search
    cy.get('[data-cy=recipe-search-input]').clear()
    
    // Test dietary filter
    cy.get('[data-cy=dietary-filter-select]').select('Vegetarian')
    cy.get('[data-cy=recipe-card]').each($card => {
      cy.wrap($card).should('contain.text', 'Vegetarian')
    })
  })

  it('should view recipe details', () => {
    cy.get('[data-cy=recipe-card]').first().click()
    
    // Verify recipe details page
    cy.url().should('include', '/recipes/')
    cy.get('[data-cy=recipe-title]').should('be.visible')
    cy.get('[data-cy=recipe-ingredients]').should('be.visible')
    cy.get('[data-cy=recipe-instructions]').should('be.visible')
    cy.get('[data-cy=recipe-nutrition]').should('be.visible')
  })
})

// cypress/support/commands.ts
declare global {
  namespace Cypress {
    interface Chainable {
      login(email: string, password: string): Chainable<void>
      getByDataCy(selector: string): Chainable<JQuery<HTMLElement>>
    }
  }
}

Cypress.Commands.add('login', (email: string, password: string) => {
  cy.session([email, password], () => {
    cy.visit('/login')
    cy.get('[data-cy=email-input]').type(email)
    cy.get('[data-cy=password-input]').type(password)
    cy.get('[data-cy=login-btn]').click()
    cy.url().should('not.include', '/login')
  })
})

Cypress.Commands.add('getByDataCy', (selector: string) => {
  return cy.get(`[data-cy=${selector}]`)
})
```

## API Testing

### Postman/Newman API Tests
```json
{
  "info": {
    "name": "MealPrep API Tests",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Authentication",
      "item": [
        {
          "name": "Login",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status code is 200', function () {",
                  "    pm.response.to.have.status(200);",
                  "});",
                  "",
                  "pm.test('Response has token', function () {",
                  "    var jsonData = pm.response.json();",
                  "    pm.expect(jsonData).to.have.property('token');",
                  "    pm.globals.set('authToken', jsonData.token);",
                  "});"
                ]
              }
            }
          ],
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"email\": \"test@example.com\",\n  \"password\": \"TestPassword123!\"\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/api/auth/login",
              "host": ["{{baseUrl}}"],
              "path": ["api", "auth", "login"]
            }
          }
        }
      ]
    },
    {
      "name": "Recipes",
      "item": [
        {
          "name": "Get Recipes",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status code is 200', function () {",
                  "    pm.response.to.have.status(200);",
                  "});",
                  "",
                  "pm.test('Response is array', function () {",
                  "    var jsonData = pm.response.json();",
                  "    pm.expect(jsonData).to.be.an('array');",
                  "});",
                  "",
                  "pm.test('Recipe has required fields', function () {",
                  "    var jsonData = pm.response.json();",
                  "    if (jsonData.length > 0) {",
                  "        pm.expect(jsonData[0]).to.have.property('id');",
                  "        pm.expect(jsonData[0]).to.have.property('name');",
                  "        pm.expect(jsonData[0]).to.have.property('prepTime');",
                  "    }",
                  "});"
                ]
              }
            }
          ],
          "request": {
            "method": "GET",
            "header": [
              {
                "key": "Authorization",
                "value": "Bearer {{authToken}}"
              }
            ],
            "url": {
              "raw": "{{baseUrl}}/api/recipes",
              "host": ["{{baseUrl}}"],
              "path": ["api", "recipes"]
            }
          }
        }
      ]
    }
  ]
}
```

## Performance Testing

### Load Testing with Artillery
```yaml
# artillery/load-test.yml
config:
  target: 'http://localhost:5000'
  phases:
    - duration: 60
      arrivalRate: 5
      name: "Warm up"
    - duration: 120
      arrivalRate: 10
      name: "Normal load"
    - duration: 60
      arrivalRate: 20
      name: "Peak load"
  defaults:
    headers:
      Content-Type: 'application/json'

scenarios:
  - name: "Recipe API Load Test"
    weight: 100
    flow:
      - post:
          url: "/api/auth/login"
          json:
            email: "test@example.com"
            password: "TestPassword123!"
          capture:
            - json: "$.token"
              as: "authToken"
      - get:
          url: "/api/recipes"
          headers:
            Authorization: "Bearer {{ authToken }}"
      - think: 2
      - get:
          url: "/api/recipes/{{ $randomString() }}"
          headers:
            Authorization: "Bearer {{ authToken }}"
          expect:
            - statusCode: [200, 404]
```

This comprehensive testing strategy ensures robust quality assurance across the entire MealPrep application stack with automated testing at every level.

*This testing strategy should be continuously updated as new features are added and testing tools evolve.*