# Code Review Guidelines

## Overview
Comprehensive code review guidelines for the MealPrep project to ensure code quality, knowledge sharing, and consistent standards across the development team.

## Code Review Philosophy

### Goals
- **Quality Assurance**: Catch bugs and issues before they reach production
- **Knowledge Sharing**: Spread domain knowledge across the team
- **Consistency**: Maintain consistent coding patterns and standards
- **Learning**: Help team members improve their skills
- **Security**: Identify potential security vulnerabilities
- **Performance**: Ensure optimal performance characteristics

### Principles
- **Constructive**: Provide helpful, actionable feedback
- **Respectful**: Be kind and professional in all interactions
- **Collaborative**: Work together to find the best solution
- **Educational**: Explain the "why" behind feedback
- **Timely**: Review code promptly to avoid blocking team members

## Review Process

### When to Request Review

#### Mandatory Reviews
- All Pull Requests to `main` and `develop` branches
- New feature implementations
- Bug fixes affecting core functionality
- Security-related changes
- Performance optimization changes
- Database schema modifications
- API endpoint changes

#### Optional but Recommended
- Refactoring existing code
- Documentation updates
- Configuration changes
- Test additions/modifications

### Review Assignment

#### Automatic Assignment
- **Team Lead**: All PRs affecting architecture or critical functionality
- **Frontend Specialist**: All React/TypeScript changes
- **Backend Specialist**: All C#/API changes
- **AI Specialist**: All AI integration and prompt engineering changes
- **DevOps Engineer**: All deployment and infrastructure changes

#### Manual Assignment
- Request specific reviewers based on domain expertise
- Tag reviewers who worked on related code previously
- Include security specialist for authentication/authorization changes
- Include database specialist for complex query or schema changes

### Review Timeline

#### Response Time Expectations
- **Acknowledgment**: Within 4 business hours
- **Initial Review**: Within 24 business hours
- **Follow-up Reviews**: Within 12 business hours
- **Emergency/Hotfix**: Within 2 hours

#### Size Guidelines
- **Small PR** (< 200 lines): 30 minutes maximum review time
- **Medium PR** (200-500 lines): 1-2 hours review time
- **Large PR** (500+ lines): Consider breaking into smaller PRs

## Review Checklist

### Functionality Review

#### Core Functionality
- [ ] Code accomplishes the intended purpose
- [ ] Edge cases are handled appropriately
- [ ] Error conditions are properly managed
- [ ] Input validation is comprehensive
- [ ] Business logic is correct and complete

#### Integration Points
- [ ] API contracts are maintained
- [ ] Database operations are correct
- [ ] External service integrations work as expected
- [ ] AI service calls handle timeouts and failures
- [ ] Authentication/authorization is properly implemented

### Code Quality Review

#### Design Patterns
- [ ] Follows established architectural patterns (Clean Architecture, DDD)
- [ ] Proper separation of concerns
- [ ] Repository pattern usage is consistent
- [ ] Dependency injection is used appropriately
- [ ] SOLID principles are followed

#### Clean Code Principles
```csharp
// Good: Clear, descriptive naming
public async Task<List<MealSuggestion>> GeneratePersonalizedMealSuggestionsAsync(
    FamilyProfile family, 
    MealPlanningConstraints constraints)
{
    var familyPersona = await _personaService.BuildFamilyPersonaAsync(family);
    var aiRequest = _promptBuilder.CreateMealSuggestionPrompt(familyPersona, constraints);
    
    try
    {
        var aiResponse = await _geminiService.GenerateContentAsync(aiRequest);
        var suggestions = _responseParser.ParseMealSuggestions(aiResponse);
        
        return await _suggestionRanker.RankByFamilyFitAsync(suggestions, family);
    }
    catch (AiServiceException ex)
    {
        _logger.LogWarning(ex, "AI service failed, falling back to recipe database");
        return await _fallbackService.GetSimilarRecipesAsync(constraints);
    }
}

// Bad: Unclear naming, complex method
public async Task<List<object>> DoStuff(object data)
{
    // Complex, unclear implementation
}
```

#### Code Structure
- [ ] Methods are focused and have single responsibility
- [ ] Classes are cohesive and properly sized
- [ ] Code is readable without excessive comments
- [ ] Complex logic is well-documented
- [ ] Magic numbers/strings are replaced with constants

### Security Review

#### Authentication & Authorization
- [ ] JWT tokens are properly validated
- [ ] User permissions are checked at appropriate levels
- [ ] Sensitive data is not logged
- [ ] API endpoints have proper authorization attributes
- [ ] User input is validated and sanitized

#### Data Protection
```csharp
// Good: Proper input validation
[HttpPost]
public async Task<ActionResult<Recipe>> CreateRecipe([FromBody] CreateRecipeRequest request)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);

    // Sanitize input
    var sanitizedRequest = _inputSanitizer.SanitizeRecipeRequest(request);
    
    // Validate user can create recipes
    var userId = User.GetUserId();
    if (!await _authService.CanUserCreateRecipe(userId))
        return Forbid();

    var recipe = await _recipeService.CreateRecipeAsync(sanitizedRequest, userId);
    return CreatedAtAction(nameof(GetRecipe), new { id = recipe.Id }, recipe);
}

// Bad: No validation, potential injection
[HttpPost]
public async Task<Recipe> CreateRecipe(string name, string instructions)
{
    var sql = $"INSERT INTO Recipes (Name, Instructions) VALUES ('{name}', '{instructions}')";
    await _database.ExecuteSqlAsync(sql); // SQL injection risk!
}
```

#### Common Security Issues
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention in frontend code
- [ ] CSRF protection for state-changing operations
- [ ] Proper error handling without information leakage
- [ ] Secure handling of API keys and secrets

### Performance Review

#### Backend Performance
- [ ] Database queries are optimized
- [ ] Appropriate use of async/await
- [ ] Caching is implemented where beneficial
- [ ] N+1 query problems are avoided
- [ ] Large datasets use pagination

```csharp
// Good: Optimized query with includes
public async Task<List<Recipe>> GetRecipesWithIngredientsAsync(int userId, int page, int pageSize)
{
    return await _context.Recipes
        .Include(r => r.Ingredients)
        .ThenInclude(ri => ri.Ingredient)
        .Where(r => r.CreatedByUserId == userId)
        .OrderBy(r => r.Name)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();
}

// Bad: N+1 query problem
public async Task<List<Recipe>> GetRecipesWithIngredients(int userId)
{
    var recipes = await _context.Recipes.Where(r => r.CreatedByUserId == userId).ToListAsync();
    
    foreach (var recipe in recipes)
    {
        recipe.Ingredients = await _context.RecipeIngredients
            .Where(ri => ri.RecipeId == recipe.Id)
            .ToListAsync(); // N+1 problem!
    }
    
    return recipes;
}
```

#### Frontend Performance
- [ ] Components use React.memo when appropriate
- [ ] useCallback and useMemo are used for expensive operations
- [ ] Unnecessary re-renders are avoided
- [ ] Large lists use virtualization
- [ ] Images are optimized and lazy-loaded

### Testing Review

#### Test Coverage
- [ ] New functionality has corresponding unit tests
- [ ] Integration tests cover API endpoints
- [ ] Edge cases and error conditions are tested
- [ ] Mocks are used appropriately
- [ ] Test names clearly describe what is being tested

```csharp
// Good: Clear test structure and naming
[TestMethod]
public async Task GenerateMealSuggestions_FamilyWithAllergies_ExcludesAllergenRecipes()
{
    // Arrange
    var family = CreateFamilyWithAllergies(new[] { "nuts", "dairy" });
    var constraints = new MealPlanningConstraints { Budget = 25.00m };
    var suggestionsWithAllergens = CreateSuggestionsWithAllergens();
    
    _mockAiService.Setup(x => x.GenerateContentAsync(It.IsAny<string>()))
        .ReturnsAsync(JsonSerializer.Serialize(suggestionsWithAllergens));

    // Act
    var result = await _mealSuggestionService.GenerateSuggestionsAsync(family, constraints);

    // Assert
    Assert.IsNotNull(result);
    Assert.IsTrue(result.All(s => !s.ContainsAllergens(new[] { "nuts", "dairy" })));
    Assert.IsTrue(result.Count > 0, "Should return at least one allergen-free suggestion");
}
```

#### Test Quality
- [ ] Tests are independent and can run in any order
- [ ] Test data is created within each test method
- [ ] Assertions are specific and meaningful
- [ ] Tests follow AAA pattern (Arrange, Act, Assert)
- [ ] Integration tests use realistic test data

## Review Feedback Guidelines

### Providing Feedback

#### Feedback Categories
Use these prefixes to categorize feedback:

- **MUST**: Critical issues that must be fixed
- **SHOULD**: Important improvements that should be addressed
- **CONSIDER**: Suggestions for improvement
- **QUESTION**: Questions for clarification
- **PRAISE**: Positive feedback for good practices

#### Example Feedback
```
MUST: This method is vulnerable to SQL injection. Use parameterized queries.

SHOULD: Consider extracting this complex logic into a separate service for better testability.

CONSIDER: Using React.memo here could improve performance for this frequently re-rendered component.

QUESTION: What happens if the AI service is down? Should we have a fallback?

PRAISE: Excellent use of the repository pattern here! The code is very clean and testable.
```

#### Constructive Feedback Examples
```
// Good feedback
"This method is doing too much. Consider breaking it into smaller methods:
1. ValidateInput()
2. ProcessFamilyData()
3. GenerateAiPrompt()
4. HandleAiResponse()

This would improve readability and make unit testing easier."

// Poor feedback
"This code is bad."
```

### Receiving Feedback

#### Professional Response
- **Acknowledge**: Thank reviewers for their time and feedback
- **Clarify**: Ask questions if feedback is unclear
- **Discuss**: Engage in constructive discussion about different approaches
- **Implement**: Make requested changes or explain why changes aren't needed
- **Learn**: Use feedback as an opportunity to improve skills

#### Example Responses
```
"Thanks for the feedback! You're right about the SQL injection risk. 
I'll update to use parameterized queries. 

Regarding the service extraction suggestion - that makes sense for testability. 
Let me refactor that and update the PR."

"Good catch on the performance issue. I've implemented React.memo and 
also added useMemo for the expensive calculation. 
The component now re-renders much less frequently."
```

## Review Automation

### Automated Checks
Before manual review, ensure these automated checks pass:

#### Code Quality
- [ ] SonarQube analysis passes
- [ ] ESLint/TSLint checks pass
- [ ] Code formatting is consistent (Prettier/EditorConfig)
- [ ] No security vulnerabilities detected
- [ ] Build succeeds without warnings

#### Testing
- [ ] All unit tests pass
- [ ] Code coverage meets minimum threshold (80%)
- [ ] Integration tests pass
- [ ] No test flakiness detected

### CI/CD Integration
```yaml
# Example GitHub Actions check
name: Code Review Checks
on:
  pull_request:
    branches: [main, develop]

jobs:
  quality-checks:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run SonarQube Analysis
      run: dotnet sonarscanner begin /k:"MealPrep"
    
    - name: Build and Test
      run: |
        dotnet build --configuration Release
        dotnet test --configuration Release --collect:"XPlat Code Coverage"
    
    - name: Frontend Quality Checks
      run: |
        cd meal-prep-frontend
        npm run lint
        npm run type-check
        npm test -- --coverage --watchAll=false
```

## Special Review Cases

### AI Integration Reviews
When reviewing AI-related code:

- [ ] Prompt engineering follows established patterns
- [ ] Error handling includes AI service failures
- [ ] Response parsing is robust and handles unexpected formats
- [ ] Cost optimization strategies are implemented
- [ ] Family persona data is properly structured
- [ ] Fallback strategies are in place

### Database Schema Changes
When reviewing database modifications:

- [ ] Migration scripts are reversible
- [ ] Existing data is preserved
- [ ] Performance impact is considered
- [ ] Indexes are updated appropriately
- [ ] Foreign key relationships are maintained

### Security-Critical Changes
For authentication, authorization, or data protection changes:

- [ ] Security specialist is included in review
- [ ] Threat modeling is considered
- [ ] Compliance requirements are met (GDPR, etc.)
- [ ] Security testing is included
- [ ] Documentation is updated

## Review Metrics

### Quality Metrics
Track these metrics to improve the review process:

- **Review Response Time**: Average time to first response
- **Review Completion Time**: Average time to approval
- **Defect Detection Rate**: Issues caught in review vs production
- **Review Coverage**: Percentage of changes reviewed
- **Review Participation**: Team member involvement in reviews

### Improvement Actions
- Weekly review of metrics and process improvements
- Quarterly retrospectives on review effectiveness
- Training sessions on review best practices
- Tool improvements to streamline the process

---

These code review guidelines ensure high-quality code, effective knowledge sharing, and continuous improvement across the MealPrep development team.

*This document should be updated based on team feedback and evolving best practices.*