# Unit Testing Guide

## Overview
Comprehensive guide for writing, organizing, and maintaining unit tests in the MealPrep application, covering both backend C#/.NET and frontend React/TypeScript testing strategies.

## Testing Philosophy

### Testing Principles
1. **Test Behavior, Not Implementation**: Focus on what the code does, not how it does it
2. **Write Tests First**: Use Test-Driven Development (TDD) when possible
3. **Maintain Test Independence**: Each test should be able to run in isolation
4. **Keep Tests Simple**: Tests should be easy to read and understand
5. **Test Coverage Goals**: Aim for 80%+ code coverage with meaningful tests

### Testing Pyramid
```
    E2E Tests (10%)
   ?? Integration Tests (20%)
  ?? Unit Tests (70%)
```

- **Unit Tests**: Fast, isolated tests for individual components/methods
- **Integration Tests**: Test interactions between components
- **E2E Tests**: Full user journey testing

---

## Backend Unit Testing (.NET/C#)

### Testing Framework Setup

#### Core Testing Dependencies
```xml
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
<PackageReference Include="NUnit" Version="3.14.0" />
<PackageReference Include="NUnit3TestAdapter" Version="4.5.0" />
<PackageReference Include="Moq" Version="4.20.69" />
<PackageReference Include="FluentAssertions" Version="6.12.0" />
<PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="8.0.0" />
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
<PackageReference Include="AutoFixture" Version="4.18.0" />
<PackageReference Include="AutoFixture.AutoMoq" Version="4.18.0" />
```

#### Test Project Structure
```
MealPrep.Tests/
??? Unit/
?   ??? Services/
?   ?   ??? RecipeServiceTests.cs
?   ?   ??? FamilyServiceTests.cs
?   ?   ??? AiSuggestionServiceTests.cs
?   ?   ??? MenuPlanningServiceTests.cs
?   ??? Controllers/
?   ?   ??? RecipeControllerTests.cs
?   ?   ??? FamilyControllerTests.cs
?   ?   ??? AiSuggestionControllerTests.cs
?   ??? Models/
?   ?   ??? RecipeTests.cs
?   ?   ??? FamilyMemberTests.cs
?   ??? Utilities/
?       ??? MappingTests.cs
?       ??? ValidatorTests.cs
??? Integration/
??? Helpers/
?   ??? TestDataBuilder.cs
?   ??? MockFactory.cs
?   ??? DatabaseFixture.cs
??? TestBase.cs
```

### Service Layer Testing

#### Recipe Service Testing Example
```csharp
[TestFixture]
public class RecipeServiceTests
{
    private Mock<IRecipeRepository> _mockRecipeRepository;
    private Mock<IFamilyService> _mockFamilyService;
    private Mock<IAiSuggestionService> _mockAiService;
    private Mock<ILogger<RecipeService>> _mockLogger;
    private Mock<IMapper> _mockMapper;
    private RecipeService _recipeService;
    private IFixture _fixture;

    [SetUp]
    public void SetUp()
    {
        _mockRecipeRepository = new Mock<IRecipeRepository>();
        _mockFamilyService = new Mock<IFamilyService>();
        _mockAiService = new Mock<IAiSuggestionService>();
        _mockLogger = new Mock<ILogger<RecipeService>>();
        _mockMapper = new Mock<IMapper>();
        
        _fixture = new Fixture().Customize(new AutoMoqCustomization());
        
        _recipeService = new RecipeService(
            _mockRecipeRepository.Object,
            _mockFamilyService.Object,
            _mockAiService.Object,
            _mockMapper.Object,
            _mockLogger.Object
        );
    }

    [Test]
    public async Task GetRecipeByIdAsync_ExistingRecipe_ReturnsRecipe()
    {
        // Arrange
        var recipeId = 123;
        var expectedRecipe = _fixture.Build<Recipe>()
            .With(r => r.Id, recipeId)
            .With(r => r.Name, "Test Recipe")
            .With(r => r.IsActive, true)
            .Create();

        var expectedDto = _fixture.Build<RecipeDto>()
            .With(r => r.Id, recipeId)
            .With(r => r.Name, "Test Recipe")
            .Create();

        _mockRecipeRepository
            .Setup(r => r.GetByIdAsync(recipeId))
            .ReturnsAsync(expectedRecipe);

        _mockMapper
            .Setup(m => m.Map<RecipeDto>(expectedRecipe))
            .Returns(expectedDto);

        // Act
        var result = await _recipeService.GetRecipeByIdAsync(recipeId);

        // Assert
        result.Should().NotBeNull();
        result.Should().BeEquivalentTo(expectedDto);
        
        _mockRecipeRepository.Verify(r => r.GetByIdAsync(recipeId), Times.Once);
        _mockMapper.Verify(m => m.Map<RecipeDto>(expectedRecipe), Times.Once);
    }

    [Test]
    public async Task GetRecipeByIdAsync_NonExistentRecipe_ReturnsNull()
    {
        // Arrange
        var recipeId = 999;
        
        _mockRecipeRepository
            .Setup(r => r.GetByIdAsync(recipeId))
            .ReturnsAsync((Recipe)null);

        // Act
        var result = await _recipeService.GetRecipeByIdAsync(recipeId);

        // Assert
        result.Should().BeNull();
        _mockRecipeRepository.Verify(r => r.GetByIdAsync(recipeId), Times.Once);
        _mockMapper.Verify(m => m.Map<RecipeDto>(It.IsAny<Recipe>()), Times.Never);
    }

    [Test]
    public async Task CreateRecipeAsync_ValidRecipe_ReturnsCreatedRecipe()
    {
        // Arrange
        var createDto = _fixture.Build<CreateRecipeDto>()
            .With(r => r.Name, "New Recipe")
            .With(r => r.PrepTimeMinutes, 30)
            .With(r => r.CookTimeMinutes, 45)
            .With(r => r.ServingSize, 4)
            .Create();

        var recipe = _fixture.Build<Recipe>()
            .With(r => r.Id, 0) // New recipe
            .With(r => r.Name, createDto.Name)
            .Create();

        var savedRecipe = _fixture.Build<Recipe>()
            .With(r => r.Id, 456) // Saved with ID
            .With(r => r.Name, createDto.Name)
            .Create();

        var resultDto = _fixture.Build<RecipeDto>()
            .With(r => r.Id, 456)
            .With(r => r.Name, createDto.Name)
            .Create();

        _mockMapper
            .Setup(m => m.Map<Recipe>(createDto))
            .Returns(recipe);

        _mockRecipeRepository
            .Setup(r => r.CreateAsync(recipe))
            .ReturnsAsync(savedRecipe);

        _mockMapper
            .Setup(m => m.Map<RecipeDto>(savedRecipe))
            .Returns(resultDto);

        // Act
        var result = await _recipeService.CreateRecipeAsync(createDto);

        // Assert
        result.Should().NotBeNull();
        result.Id.Should().Be(456);
        result.Name.Should().Be(createDto.Name);

        _mockMapper.Verify(m => m.Map<Recipe>(createDto), Times.Once);
        _mockRecipeRepository.Verify(r => r.CreateAsync(recipe), Times.Once);
        _mockMapper.Verify(m => m.Map<RecipeDto>(savedRecipe), Times.Once);
    }

    [Test]
    public async Task SearchRecipesAsync_WithCriteria_ReturnsFilteredResults()
    {
        // Arrange
        var criteria = new RecipeSearchCriteria
        {
            SearchTerm = "chicken",
            Cuisine = "Italian",
            MaxPrepTime = 30,
            Page = 1,
            PageSize = 10
        };

        var recipes = _fixture.CreateMany<Recipe>(5).ToList();
        var recipeDtos = _fixture.CreateMany<RecipeDto>(5).ToList();

        _mockRecipeRepository
            .Setup(r => r.SearchAsync(criteria))
            .ReturnsAsync(recipes);

        _mockMapper
            .Setup(m => m.Map<IEnumerable<RecipeDto>>(recipes))
            .Returns(recipeDtos);

        // Act
        var result = await _recipeService.SearchRecipesAsync(criteria);

        // Assert
        result.Should().NotBeNull();
        result.Should().HaveCount(5);
        result.Should().BeEquivalentTo(recipeDtos);

        _mockRecipeRepository.Verify(r => r.SearchAsync(criteria), Times.Once);
    }

    [Test]
    public void CreateRecipeAsync_NullInput_ThrowsArgumentNullException()
    {
        // Act & Assert
        _recipeService.Invoking(s => s.CreateRecipeAsync(null))
            .Should().ThrowAsync<ArgumentNullException>();
    }
}
```

#### Family Service Testing Example
```csharp
[TestFixture]
public class FamilyServiceTests
{
    private Mock<IFamilyMemberRepository> _mockRepository;
    private Mock<IMapper> _mockMapper;
    private Mock<ILogger<FamilyService>> _mockLogger;
    private FamilyService _familyService;
    private IFixture _fixture;

    [SetUp]
    public void SetUp()
    {
        _mockRepository = new Mock<IFamilyMemberRepository>();
        _mockMapper = new Mock<IMapper>();
        _mockLogger = new Mock<ILogger<FamilyService>>();
        _fixture = new Fixture().Customize(new AutoMoqCustomization());

        _familyService = new FamilyService(
            _mockRepository.Object,
            _mockMapper.Object,
            _mockLogger.Object
        );
    }

    [Test]
    public async Task GetFamilyMembersAsync_ValidUserId_ReturnsFamilyMembers()
    {
        // Arrange
        var userId = 123;
        var familyMembers = _fixture.Build<FamilyMember>()
            .With(fm => fm.UserId, userId)
            .With(fm => fm.IsActive, true)
            .CreateMany(3)
            .ToList();

        var familyMemberDtos = _fixture.CreateMany<FamilyMemberDto>(3).ToList();

        _mockRepository
            .Setup(r => r.GetByUserIdAsync(userId, true))
            .ReturnsAsync(familyMembers);

        _mockMapper
            .Setup(m => m.Map<IEnumerable<FamilyMemberDto>>(familyMembers))
            .Returns(familyMemberDtos);

        // Act
        var result = await _familyService.GetFamilyMembersAsync(userId);

        // Assert
        result.Should().NotBeNull();
        result.Should().HaveCount(3);
        result.Should().BeEquivalentTo(familyMemberDtos);
    }

    [Test]
    public async Task AddFamilyMemberAsync_ExceedsMaxLimit_ThrowsBusinessException()
    {
        // Arrange
        var userId = 123;
        var createDto = _fixture.Create<CreateFamilyMemberDto>();
        
        var existingMembers = _fixture.Build<FamilyMember>()
            .With(fm => fm.UserId, userId)
            .With(fm => fm.IsActive, true)
            .CreateMany(10) // Max limit reached
            .ToList();

        _mockRepository
            .Setup(r => r.GetByUserIdAsync(userId, true))
            .ReturnsAsync(existingMembers);

        // Act & Assert
        await _familyService.Invoking(s => s.AddFamilyMemberAsync(userId, createDto))
            .Should().ThrowAsync<BusinessException>()
            .WithMessage("Maximum family size limit exceeded*");
    }
}
```

### Controller Testing

#### Recipe Controller Testing Example
```csharp
[TestFixture]
public class RecipeControllerTests
{
    private Mock<IRecipeService> _mockRecipeService;
    private Mock<ILogger<RecipeController>> _mockLogger;
    private RecipeController _controller;
    private IFixture _fixture;

    [SetUp]
    public void SetUp()
    {
        _mockRecipeService = new Mock<IRecipeService>();
        _mockLogger = new Mock<ILogger<RecipeController>>();
        _fixture = new Fixture().Customize(new AutoMoqCustomization());

        _controller = new RecipeController(_mockRecipeService.Object, _mockLogger.Object);
        
        // Setup controller context for authentication
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, "123"),
            new Claim(ClaimTypes.Email, "test@example.com")
        };
        var identity = new ClaimsIdentity(claims, "TestAuth");
        var principal = new ClaimsPrincipal(identity);
        
        _controller.ControllerContext = new ControllerContext
        {
            HttpContext = new DefaultHttpContext { User = principal }
        };
    }

    [Test]
    public async Task GetRecipe_ExistingId_ReturnsOkWithRecipe()
    {
        // Arrange
        var recipeId = 123;
        var recipeDto = _fixture.Build<RecipeDto>()
            .With(r => r.Id, recipeId)
            .Create();

        _mockRecipeService
            .Setup(s => s.GetRecipeByIdAsync(recipeId))
            .ReturnsAsync(recipeDto);

        // Act
        var result = await _controller.GetRecipe(recipeId);

        // Assert
        result.Should().BeOfType<OkObjectResult>();
        var okResult = result as OkObjectResult;
        okResult.Value.Should().BeEquivalentTo(recipeDto);
    }

    [Test]
    public async Task GetRecipe_NonExistentId_ReturnsNotFound()
    {
        // Arrange
        var recipeId = 999;
        
        _mockRecipeService
            .Setup(s => s.GetRecipeByIdAsync(recipeId))
            .ReturnsAsync((RecipeDto)null);

        // Act
        var result = await _controller.GetRecipe(recipeId);

        // Assert
        result.Should().BeOfType<NotFoundResult>();
    }

    [Test]
    public async Task CreateRecipe_ValidInput_ReturnsCreatedResult()
    {
        // Arrange
        var createDto = _fixture.Create<CreateRecipeDto>();
        var createdRecipe = _fixture.Build<RecipeDto>()
            .With(r => r.Id, 456)
            .Create();

        _mockRecipeService
            .Setup(s => s.CreateRecipeAsync(createDto))
            .ReturnsAsync(createdRecipe);

        // Act
        var result = await _controller.CreateRecipe(createDto);

        // Assert
        result.Should().BeOfType<CreatedAtActionResult>();
        var createdResult = result as CreatedAtActionResult;
        createdResult.Value.Should().BeEquivalentTo(createdRecipe);
        createdResult.ActionName.Should().Be(nameof(RecipeController.GetRecipe));
        createdResult.RouteValues["id"].Should().Be(456);
    }

    [Test]
    public async Task CreateRecipe_InvalidModel_ReturnsBadRequest()
    {
        // Arrange
        var createDto = _fixture.Create<CreateRecipeDto>();
        _controller.ModelState.AddModelError("Name", "Name is required");

        // Act
        var result = await _controller.CreateRecipe(createDto);

        // Assert
        result.Should().BeOfType<BadRequestObjectResult>();
        _mockRecipeService.Verify(s => s.CreateRecipeAsync(It.IsAny<CreateRecipeDto>()), Times.Never);
    }

    [Test]
    public async Task SearchRecipes_ValidCriteria_ReturnsOkWithResults()
    {
        // Arrange
        var criteria = new RecipeSearchCriteria
        {
            SearchTerm = "chicken",
            Page = 1,
            PageSize = 10
        };

        var recipes = _fixture.CreateMany<RecipeDto>(5).ToList();

        _mockRecipeService
            .Setup(s => s.SearchRecipesAsync(criteria))
            .ReturnsAsync(recipes);

        // Act
        var result = await _controller.SearchRecipes(criteria);

        // Assert
        result.Should().BeOfType<OkObjectResult>();
        var okResult = result as OkObjectResult;
        okResult.Value.Should().BeEquivalentTo(recipes);
    }
}
```

### Model/Entity Testing

#### Recipe Model Testing Example
```csharp
[TestFixture]
public class RecipeTests
{
    private IFixture _fixture;

    [SetUp]
    public void SetUp()
    {
        _fixture = new Fixture();
    }

    [Test]
    public void TotalTimeMinutes_ValidPrepAndCookTime_ReturnsCorrectSum()
    {
        // Arrange
        var recipe = new Recipe
        {
            PrepTimeMinutes = 15,
            CookTimeMinutes = 30
        };

        // Act
        var totalTime = recipe.TotalTimeMinutes;

        // Assert
        totalTime.Should().Be(45);
    }

    [Test]
    public void IsQuickMeal_TotalTimeUnder30Minutes_ReturnsTrue()
    {
        // Arrange
        var recipe = new Recipe
        {
            PrepTimeMinutes = 10,
            CookTimeMinutes = 15
        };

        // Act
        var isQuick = recipe.IsQuickMeal;

        // Assert
        isQuick.Should().BeTrue();
    }

    [Test]
    public void IsQuickMeal_TotalTimeOver30Minutes_ReturnsFalse()
    {
        // Arrange
        var recipe = new Recipe
        {
            PrepTimeMinutes = 20,
            CookTimeMinutes = 25
        };

        // Act
        var isQuick = recipe.IsQuickMeal;

        // Assert
        isQuick.Should().BeFalse();
    }

    [Test]
    public void AddIngredient_ValidIngredient_AddsToCollection()
    {
        // Arrange
        var recipe = new Recipe();
        var ingredient = new RecipeIngredient
        {
            IngredientName = "Chicken Breast",
            Quantity = 2,
            Unit = "lbs"
        };

        // Act
        recipe.AddIngredient(ingredient);

        // Assert
        recipe.Ingredients.Should().Contain(ingredient);
        recipe.Ingredients.Should().HaveCount(1);
    }

    [Test]
    public void UpdateAverageRating_MultipleRatings_CalculatesCorrectAverage()
    {
        // Arrange
        var recipe = new Recipe();
        var ratings = new List<RecipeRating>
        {
            new RecipeRating { Rating = 5 },
            new RecipeRating { Rating = 4 },
            new RecipeRating { Rating = 3 }
        };

        // Act
        recipe.UpdateAverageRating(ratings);

        // Assert
        recipe.AverageRating.Should().Be(4.0m);
        recipe.RatingCount.Should().Be(3);
    }
}
```

### Validation Testing

#### Recipe Validator Testing Example
```csharp
[TestFixture]
public class CreateRecipeDtoValidatorTests
{
    private CreateRecipeDtoValidator _validator;
    private IFixture _fixture;

    [SetUp]
    public void SetUp()
    {
        _validator = new CreateRecipeDtoValidator();
        _fixture = new Fixture();
    }

    [Test]
    public void Validate_ValidRecipe_ReturnsNoErrors()
    {
        // Arrange
        var recipe = new CreateRecipeDto
        {
            Name = "Test Recipe",
            Description = "A test recipe",
            Instructions = "1. Do this\n2. Do that",
            PrepTimeMinutes = 15,
            CookTimeMinutes = 30,
            ServingSize = 4,
            DifficultyLevel = "Easy"
        };

        // Act
        var result = _validator.Validate(recipe);

        // Assert
        result.IsValid.Should().BeTrue();
        result.Errors.Should().BeEmpty();
    }

    [Test]
    public void Validate_EmptyName_ReturnsError()
    {
        // Arrange
        var recipe = _fixture.Build<CreateRecipeDto>()
            .With(r => r.Name, string.Empty)
            .Create();

        // Act
        var result = _validator.Validate(recipe);

        // Assert
        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => e.PropertyName == nameof(CreateRecipeDto.Name));
    }

    [Test]
    [TestCase(0)]
    [TestCase(-1)]
    public void Validate_InvalidPrepTime_ReturnsError(int prepTime)
    {
        // Arrange
        var recipe = _fixture.Build<CreateRecipeDto>()
            .With(r => r.PrepTimeMinutes, prepTime)
            .Create();

        // Act
        var result = _validator.Validate(recipe);

        // Assert
        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => e.PropertyName == nameof(CreateRecipeDto.PrepTimeMinutes));
    }

    [Test]
    [TestCase("Easy")]
    [TestCase("Medium")]
    [TestCase("Hard")]
    public void Validate_ValidDifficultyLevel_ReturnsNoError(string difficulty)
    {
        // Arrange
        var recipe = _fixture.Build<CreateRecipeDto>()
            .With(r => r.DifficultyLevel, difficulty)
            .Create();

        // Act
        var result = _validator.Validate(recipe);

        // Assert
        result.Errors.Should().NotContain(e => e.PropertyName == nameof(CreateRecipeDto.DifficultyLevel));
    }
}
```

---

## Frontend Unit Testing (React/TypeScript)

### Testing Framework Setup

#### Core Testing Dependencies
```json
{
  "devDependencies": {
    "@testing-library/react": "^13.4.0",
    "@testing-library/jest-dom": "^5.16.5",
    "@testing-library/user-event": "^14.4.3",
    "jest": "^27.5.1",
    "jest-environment-jsdom": "^27.5.1",
    "@types/jest": "^27.5.2",
    "msw": "^1.3.2",
    "jest-canvas-mock": "^2.4.0"
  }
}
```

#### Jest Configuration (jest.config.js)
```javascript
module.exports = {
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/src/setupTests.ts'],
  moduleNameMapping: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/components/$1',
    '^@pages/(.*)$': '<rootDir>/src/pages/$1',
    '^@services/(.*)$': '<rootDir>/src/services/$1',
    '^@utils/(.*)$': '<rootDir>/src/utils/$1',
    '^@types/(.*)$': '<rootDir>/src/types/$1',
    '^@hooks/(.*)$': '<rootDir>/src/hooks/$1',
    '\\.(css|less|scss|sass)$': 'identity-obj-proxy'
  },
  transform: {
    '^.+\\.(ts|tsx)$': 'ts-jest'
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/index.tsx',
    '!src/reportWebVitals.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
};
```

### Component Testing

#### Recipe Card Component Testing
```typescript
// RecipeCard.test.tsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import '@testing-library/jest-dom';
import { RecipeCard } from '@components/RecipeCard';
import { Recipe } from '@types/Recipe';

const mockRecipe: Recipe = {
  id: 1,
  name: 'Grilled Chicken',
  description: 'Delicious grilled chicken breast',
  prepTimeMinutes: 15,
  cookTimeMinutes: 25,
  servingSize: 4,
  difficulty: 'Easy',
  cuisine: 'American',
  averageRating: 4.5,
  ratingCount: 12,
  imageUrl: 'https://example.com/chicken.jpg',
  isAiGenerated: false,
  familyFitScore: 8
};

describe('RecipeCard', () => {
  const mockOnFavorite = jest.fn();
  const mockOnRate = jest.fn();
  const mockOnClick = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders recipe information correctly', () => {
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    expect(screen.getByText('Grilled Chicken')).toBeInTheDocument();
    expect(screen.getByText('Delicious grilled chicken breast')).toBeInTheDocument();
    expect(screen.getByText('15 min prep')).toBeInTheDocument();
    expect(screen.getByText('25 min cook')).toBeInTheDocument();
    expect(screen.getByText('Serves 4')).toBeInTheDocument();
    expect(screen.getByText('Easy')).toBeInTheDocument();
    expect(screen.getByText('American')).toBeInTheDocument();
  });

  it('displays rating information correctly', () => {
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    expect(screen.getByText('4.5')).toBeInTheDocument();
    expect(screen.getByText('(12 reviews)')).toBeInTheDocument();
  });

  it('shows AI badge when recipe is AI generated', () => {
    const aiRecipe = { ...mockRecipe, isAiGenerated: true };
    
    render(
      <RecipeCard
        recipe={aiRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    expect(screen.getByText('AI Generated')).toBeInTheDocument();
  });

  it('calls onClick when card is clicked', async () => {
    const user = userEvent.setup();
    
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    await user.click(screen.getByTestId('recipe-card'));

    expect(mockOnClick).toHaveBeenCalledWith(mockRecipe);
  });

  it('calls onFavorite when favorite button is clicked', async () => {
    const user = userEvent.setup();
    
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    await user.click(screen.getByTestId('favorite-button'));

    expect(mockOnFavorite).toHaveBeenCalledWith(mockRecipe.id);
    expect(mockOnClick).not.toHaveBeenCalled(); // Should not trigger card click
  });

  it('displays family fit score when provided', () => {
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    expect(screen.getByText('Family Fit: 8/10')).toBeInTheDocument();
  });

  it('handles missing image gracefully', () => {
    const recipeWithoutImage = { ...mockRecipe, imageUrl: undefined };
    
    render(
      <RecipeCard
        recipe={recipeWithoutImage}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    const image = screen.getByRole('img');
    expect(image).toHaveAttribute('src', '/placeholder-recipe.jpg');
  });
});
```

#### Recipe Search Component Testing
```typescript
// RecipeSearch.test.tsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { RecipeSearch } from '@components/RecipeSearch';
import { RecipeSearchCriteria } from '@types/Recipe';

describe('RecipeSearch', () => {
  const mockOnSearch = jest.fn();
  const mockOnClear = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders all search fields', () => {
    render(<RecipeSearch onSearch={mockOnSearch} onClear={mockOnClear} />);

    expect(screen.getByPlaceholderText('Search recipes...')).toBeInTheDocument();
    expect(screen.getByLabelText('Cuisine')).toBeInTheDocument();
    expect(screen.getByLabelText('Difficulty')).toBeInTheDocument();
    expect(screen.getByLabelText('Max Prep Time')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: 'Search' })).toBeInTheDocument();
    expect(screen.getByRole('button', { name: 'Clear' })).toBeInTheDocument();
  });

  it('calls onSearch with correct criteria when search button is clicked', async () => {
    const user = userEvent.setup();
    
    render(<RecipeSearch onSearch={mockOnSearch} onClear={mockOnClear} />);

    await user.type(screen.getByPlaceholderText('Search recipes...'), 'chicken');
    await user.selectOptions(screen.getByLabelText('Cuisine'), 'Italian');
    await user.selectOptions(screen.getByLabelText('Difficulty'), 'Easy');
    await user.type(screen.getByLabelText('Max Prep Time'), '30');

    await user.click(screen.getByRole('button', { name: 'Search' }));

    expect(mockOnSearch).toHaveBeenCalledWith({
      searchTerm: 'chicken',
      cuisine: 'Italian',
      difficulty: 'Easy',
      maxPrepTime: 30
    });
  });

  it('calls onClear when clear button is clicked', async () => {
    const user = userEvent.setup();
    
    render(<RecipeSearch onSearch={mockOnSearch} onClear={mockOnClear} />);

    await user.type(screen.getByPlaceholderText('Search recipes...'), 'chicken');
    await user.click(screen.getByRole('button', { name: 'Clear' }));

    expect(mockOnClear).toHaveBeenCalled();
    expect(screen.getByPlaceholderText('Search recipes...')).toHaveValue('');
  });

  it('disables search button when no search criteria is provided', () => {
    render(<RecipeSearch onSearch={mockOnSearch} onClear={mockOnClear} />);

    expect(screen.getByRole('button', { name: 'Search' })).toBeDisabled();
  });

  it('enables search button when search criteria is provided', async () => {
    const user = userEvent.setup();
    
    render(<RecipeSearch onSearch={mockOnSearch} onClear={mockOnClear} />);

    await user.type(screen.getByPlaceholderText('Search recipes...'), 'chicken');

    expect(screen.getByRole('button', { name: 'Search' })).toBeEnabled();
  });
});
```

### Custom Hook Testing

#### useRecipes Hook Testing
```typescript
// useRecipes.test.tsx
import { renderHook, waitFor } from '@testing-library/react';
import { useRecipes } from '@hooks/useRecipes';
import { recipeService } from '@services/recipeService';
import { Recipe } from '@types/Recipe';

// Mock the service
jest.mock('@services/recipeService');
const mockedRecipeService = recipeService as jest.Mocked<typeof recipeService>;

const mockRecipes: Recipe[] = [
  {
    id: 1,
    name: 'Recipe 1',
    description: 'Test recipe 1',
    prepTimeMinutes: 15,
    cookTimeMinutes: 30,
    servingSize: 4,
    difficulty: 'Easy',
    cuisine: 'Italian'
  },
  {
    id: 2,
    name: 'Recipe 2',
    description: 'Test recipe 2',
    prepTimeMinutes: 20,
    cookTimeMinutes: 25,
    servingSize: 2,
    difficulty: 'Medium',
    cuisine: 'Mexican'
  }
];

describe('useRecipes', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('loads recipes on mount', async () => {
    mockedRecipeService.searchRecipes.mockResolvedValue(mockRecipes);

    const { result } = renderHook(() => useRecipes());

    expect(result.current.loading).toBe(true);
    expect(result.current.recipes).toEqual([]);

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.recipes).toEqual(mockRecipes);
    expect(result.current.error).toBeNull();
  });

  it('handles search correctly', async () => {
    const searchCriteria = { searchTerm: 'chicken' };
    mockedRecipeService.searchRecipes.mockResolvedValue(mockRecipes);

    const { result } = renderHook(() => useRecipes());

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    result.current.searchRecipes(searchCriteria);

    expect(result.current.loading).toBe(true);
    
    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(mockedRecipeService.searchRecipes).toHaveBeenCalledWith(searchCriteria);
  });

  it('handles errors correctly', async () => {
    const errorMessage = 'Failed to load recipes';
    mockedRecipeService.searchRecipes.mockRejectedValue(new Error(errorMessage));

    const { result } = renderHook(() => useRecipes());

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.error).toBe(errorMessage);
    expect(result.current.recipes).toEqual([]);
  });
});
```

### Service Testing with MSW

#### Recipe Service Testing
```typescript
// recipeService.test.ts
import { rest } from 'msw';
import { setupServer } from 'msw/node';
import { recipeService } from '@services/recipeService';
import { Recipe, CreateRecipeDto } from '@types/Recipe';

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

const server = setupServer(
  rest.get('/api/recipes', (req, res, ctx) => {
    const searchTerm = req.url.searchParams.get('searchTerm');
    
    if (searchTerm === 'error') {
      return res(ctx.status(500), ctx.json({ message: 'Server error' }));
    }
    
    return res(ctx.json(mockRecipes));
  }),

  rest.get('/api/recipes/:id', (req, res, ctx) => {
    const { id } = req.params;
    
    if (id === '999') {
      return res(ctx.status(404));
    }
    
    return res(ctx.json(mockRecipes[0]));
  }),

  rest.post('/api/recipes', (req, res, ctx) => {
    return res(
      ctx.status(201),
      ctx.json({ ...mockRecipes[0], id: 999 })
    );
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('recipeService', () => {
  it('searches recipes successfully', async () => {
    const criteria = { searchTerm: 'chicken' };
    const result = await recipeService.searchRecipes(criteria);

    expect(result).toEqual(mockRecipes);
  });

  it('handles search errors', async () => {
    const criteria = { searchTerm: 'error' };

    await expect(recipeService.searchRecipes(criteria))
      .rejects
      .toThrow('Server error');
  });

  it('gets recipe by id successfully', async () => {
    const result = await recipeService.getRecipeById(1);

    expect(result).toEqual(mockRecipes[0]);
  });

  it('handles recipe not found', async () => {
    await expect(recipeService.getRecipeById(999))
      .rejects
      .toThrow('Recipe not found');
  });

  it('creates recipe successfully', async () => {
    const createDto: CreateRecipeDto = {
      name: 'New Recipe',
      description: 'A new test recipe',
      instructions: '1. Do this\n2. Do that',
      prepTimeMinutes: 10,
      cookTimeMinutes: 20,
      servingSize: 2,
      difficulty: 'Easy'
    };

    const result = await recipeService.createRecipe(createDto);

    expect(result.id).toBe(999);
    expect(result.name).toBe(mockRecipes[0].name);
  });
});
```

---

## Test Data Management

### Test Data Builders
```csharp
// TestDataBuilder.cs
public class RecipeTestDataBuilder
{
    private Recipe _recipe;

    public RecipeTestDataBuilder()
    {
        _recipe = new Recipe
        {
            Id = 1,
            Name = "Test Recipe",
            Description = "A test recipe",
            Instructions = "1. Do this\n2. Do that",
            PrepTimeMinutes = 15,
            CookTimeMinutes = 30,
            ServingSize = 4,
            DifficultyLevel = "Easy",
            Cuisine = "Italian",
            IsActive = true,
            CreatedDate = DateTime.UtcNow,
            CreatedByUserId = 1
        };
    }

    public RecipeTestDataBuilder WithId(int id)
    {
        _recipe.Id = id;
        return this;
    }

    public RecipeTestDataBuilder WithName(string name)
    {
        _recipe.Name = name;
        return this;
    }

    public RecipeTestDataBuilder WithDifficulty(string difficulty)
    {
        _recipe.DifficultyLevel = difficulty;
        return this;
    }

    public RecipeTestDataBuilder WithTimes(int prepTime, int cookTime)
    {
        _recipe.PrepTimeMinutes = prepTime;
        _recipe.CookTimeMinutes = cookTime;
        return this;
    }

    public RecipeTestDataBuilder AsAiGenerated()
    {
        _recipe.IsAiGenerated = true;
        return this;
    }

    public Recipe Build() => _recipe;

    public static implicit operator Recipe(RecipeTestDataBuilder builder) => builder.Build();
}

// Usage in tests
var recipe = new RecipeTestDataBuilder()
    .WithId(123)
    .WithName("Chicken Parmesan")
    .WithDifficulty("Medium")
    .WithTimes(20, 35)
    .AsAiGenerated()
    .Build();
```

### Mock Factories
```csharp
// MockFactory.cs
public static class MockFactory
{
    public static Mock<IRecipeRepository> CreateRecipeRepositoryMock()
    {
        var mock = new Mock<IRecipeRepository>();
        
        // Setup common default behaviors
        mock.Setup(r => r.GetByIdAsync(It.IsAny<int>()))
            .ReturnsAsync((int id) => new RecipeTestDataBuilder().WithId(id).Build());
            
        mock.Setup(r => r.CreateAsync(It.IsAny<Recipe>()))
            .ReturnsAsync((Recipe recipe) => 
            {
                recipe.Id = new Random().Next(1000, 9999);
                recipe.CreatedDate = DateTime.UtcNow;
                return recipe;
            });
            
        return mock;
    }

    public static Mock<IMapper> CreateMapperMock()
    {
        var mock = new Mock<IMapper>();
        
        // Setup common mappings
        mock.Setup(m => m.Map<RecipeDto>(It.IsAny<Recipe>()))
            .Returns((Recipe recipe) => new RecipeDto
            {
                Id = recipe.Id,
                Name = recipe.Name,
                Description = recipe.Description,
                PrepTimeMinutes = recipe.PrepTimeMinutes,
                CookTimeMinutes = recipe.CookTimeMinutes,
                ServingSize = recipe.ServingSize,
                DifficultyLevel = recipe.DifficultyLevel,
                Cuisine = recipe.Cuisine
            });
            
        return mock;
    }
}
```

---

## Testing Best Practices

### General Principles
1. **AAA Pattern**: Arrange, Act, Assert
2. **One Assertion Per Test**: Focus on testing one behavior
3. **Descriptive Test Names**: Test names should describe what is being tested
4. **Independent Tests**: Tests should not depend on each other
5. **Fast Execution**: Unit tests should run quickly

### Naming Conventions
```csharp
// Pattern: MethodName_Scenario_ExpectedResult
[Test]
public void GetRecipeById_ExistingId_ReturnsRecipe() { }

[Test]
public void GetRecipeById_NonExistentId_ReturnsNull() { }

[Test]
public void CreateRecipe_NullInput_ThrowsArgumentNullException() { }
```

### Common Anti-Patterns to Avoid
1. **Testing Implementation Details**: Test behavior, not implementation
2. **Brittle Tests**: Tests that break with minor changes
3. **Slow Tests**: Unit tests should run in milliseconds
4. **Testing Multiple Things**: Each test should verify one behavior
5. **Hidden Dependencies**: All dependencies should be explicit

### Code Coverage Guidelines
- **Aim for 80%+ coverage** but focus on quality over quantity
- **Cover critical paths** and business logic thoroughly
- **Don't chase 100%** - some code (like DTOs) may not need testing
- **Use coverage to find untested code**, not as a quality metric

---

*Last Updated: December 2024*  
*Unit testing guide continuously updated with new patterns and best practices*