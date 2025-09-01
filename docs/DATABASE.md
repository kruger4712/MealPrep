# MealMenu Database Design

## Overview

The MealMenu database is designed using SQL Server with Entity Framework Core. The schema supports family meal planning, recipe management, AI-powered suggestions, and user preferences.

## Database Schema

### Core Tables

#### Users Table
Stores user account information and authentication data.

```sql
CREATE TABLE Users (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Email NVARCHAR(256) NOT NULL UNIQUE,
    PasswordHash NVARCHAR(256) NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    EmailConfirmed BIT NOT NULL DEFAULT 0,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    IsActive BIT NOT NULL DEFAULT 1
);
```

#### Families Table
Represents family units that share meal planning.

```sql
CREATE TABLE Families (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Name NVARCHAR(100) NOT NULL,
    CreatedById UNIQUEIDENTIFIER NOT NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_Families_CreatedBy FOREIGN KEY (CreatedById) REFERENCES Users(Id)
);
```

#### FamilyMembers Table
Individual family members with their preferences and dietary restrictions.

```sql
CREATE TABLE FamilyMembers (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FamilyId UNIQUEIDENTIFIER NOT NULL,
    Name NVARCHAR(100) NOT NULL,
    Age INT,
    DietaryRestrictions NVARCHAR(MAX), -- JSON array
    FoodPreferences NVARCHAR(MAX), -- JSON object
    Allergies NVARCHAR(MAX), -- JSON array
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_FamilyMembers_Family FOREIGN KEY (FamilyId) REFERENCES Families(Id)
);
```

### Recipe Management

#### Recipes Table
Core recipe information and metadata.

```sql
CREATE TABLE Recipes (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Name NVARCHAR(200) NOT NULL,
    Description NVARCHAR(1000),
    PrepTime INT NOT NULL, -- Minutes
    CookTime INT NOT NULL, -- Minutes
    Servings INT NOT NULL,
    Difficulty NVARCHAR(20) NOT NULL, -- Easy, Medium, Hard
    Category NVARCHAR(50) NOT NULL,
    ImageUrl NVARCHAR(500),
    CreatedById UNIQUEIDENTIFIER NOT NULL,
    IsPublic BIT NOT NULL DEFAULT 0,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_Recipes_CreatedBy FOREIGN KEY (CreatedById) REFERENCES Users(Id)
);
```

#### Ingredients Table
Master list of ingredients with nutritional information.

```sql
CREATE TABLE Ingredients (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Name NVARCHAR(100) NOT NULL UNIQUE,
    Category NVARCHAR(50),
    NutritionalInfo NVARCHAR(MAX), -- JSON object with calories, protein, etc.
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);
```

#### RecipeIngredients Table
Ingredients required for each recipe with quantities.

```sql
CREATE TABLE RecipeIngredients (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    RecipeId UNIQUEIDENTIFIER NOT NULL,
    IngredientId UNIQUEIDENTIFIER NOT NULL,
    Amount DECIMAL(10,2) NOT NULL,
    Unit NVARCHAR(20) NOT NULL,
    Notes NVARCHAR(200),
    CONSTRAINT FK_RecipeIngredients_Recipe FOREIGN KEY (RecipeId) REFERENCES Recipes(Id),
    CONSTRAINT FK_RecipeIngredients_Ingredient FOREIGN KEY (IngredientId) REFERENCES Ingredients(Id)
);
```

#### RecipeInstructions Table
Step-by-step cooking instructions.

```sql
CREATE TABLE RecipeInstructions (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    RecipeId UNIQUEIDENTIFIER NOT NULL,
    StepNumber INT NOT NULL,
    Description NVARCHAR(1000) NOT NULL,
    EstimatedTime INT, -- Minutes for this step
    CONSTRAINT FK_RecipeInstructions_Recipe FOREIGN KEY (RecipeId) REFERENCES Recipes(Id)
);
```

### Meal Planning

#### MealPlans Table
Weekly or monthly meal planning schedules.

```sql
CREATE TABLE MealPlans (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FamilyId UNIQUEIDENTIFIER NOT NULL,
    Name NVARCHAR(100) NOT NULL,
    StartDate DATE NOT NULL,
    EndDate DATE NOT NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_MealPlans_Family FOREIGN KEY (FamilyId) REFERENCES Families(Id)
);
```

#### PlannedMeals Table
Individual meals scheduled in meal plans.

```sql
CREATE TABLE PlannedMeals (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    MealPlanId UNIQUEIDENTIFIER NOT NULL,
    RecipeId UNIQUEIDENTIFIER NOT NULL,
    MealDate DATE NOT NULL,
    MealType NVARCHAR(20) NOT NULL, -- Breakfast, Lunch, Dinner, Snack
    ServingsPlanned INT NOT NULL,
    Notes NVARCHAR(500),
    CONSTRAINT FK_PlannedMeals_MealPlan FOREIGN KEY (MealPlanId) REFERENCES MealPlans(Id),
    CONSTRAINT FK_PlannedMeals_Recipe FOREIGN KEY (RecipeId) REFERENCES Recipes(Id)
);
```

### Shopping Lists

#### ShoppingLists Table
Generated shopping lists from meal plans.

```sql
CREATE TABLE ShoppingLists (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FamilyId UNIQUEIDENTIFIER NOT NULL,
    MealPlanId UNIQUEIDENTIFIER,
    Name NVARCHAR(100) NOT NULL,
    Status NVARCHAR(20) NOT NULL DEFAULT 'Active', -- Active, Completed, Archived
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_ShoppingLists_Family FOREIGN KEY (FamilyId) REFERENCES Families(Id),
    CONSTRAINT FK_ShoppingLists_MealPlan FOREIGN KEY (MealPlanId) REFERENCES MealPlans(Id)
);
```

#### ShoppingListItems Table
Individual items in shopping lists.

```sql
CREATE TABLE ShoppingListItems (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    ShoppingListId UNIQUEIDENTIFIER NOT NULL,
    IngredientId UNIQUEIDENTIFIER NOT NULL,
    Amount DECIMAL(10,2) NOT NULL,
    Unit NVARCHAR(20) NOT NULL,
    IsPurchased BIT NOT NULL DEFAULT 0,
    Notes NVARCHAR(200),
    CONSTRAINT FK_ShoppingListItems_ShoppingList FOREIGN KEY (ShoppingListId) REFERENCES ShoppingLists(Id),
    CONSTRAINT FK_ShoppingListItems_Ingredient FOREIGN KEY (IngredientId) REFERENCES Ingredients(Id)
);
```

### AI Integration

#### AISuggestions Table
Store AI-generated meal suggestions and recommendations.

```sql
CREATE TABLE AISuggestions (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FamilyId UNIQUEIDENTIFIER NOT NULL,
    SuggestionType NVARCHAR(50) NOT NULL, -- MealPlan, Recipe, Substitution
    RequestData NVARCHAR(MAX) NOT NULL, -- JSON request parameters
    SuggestionData NVARCHAR(MAX) NOT NULL, -- JSON AI response
    IsAccepted BIT,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_AISuggestions_Family FOREIGN KEY (FamilyId) REFERENCES Families(Id)
);
```

#### FamilyPreferences Table
AI learning data about family food preferences.

```sql
CREATE TABLE FamilyPreferences (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FamilyId UNIQUEIDENTIFIER NOT NULL,
    PreferenceType NVARCHAR(50) NOT NULL,
    PreferenceData NVARCHAR(MAX) NOT NULL, -- JSON data
    Confidence DECIMAL(3,2), -- AI confidence score 0-1
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_FamilyPreferences_Family FOREIGN KEY (FamilyId) REFERENCES Families(Id)
);
```

### User Activity

#### UserFavorites Table
Track user favorite recipes and meals.

```sql
CREATE TABLE UserFavorites (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId UNIQUEIDENTIFIER NOT NULL,
    RecipeId UNIQUEIDENTIFIER NOT NULL,
    FavoritedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_UserFavorites_User FOREIGN KEY (UserId) REFERENCES Users(Id),
    CONSTRAINT FK_UserFavorites_Recipe FOREIGN KEY (RecipeId) REFERENCES Recipes(Id),
    CONSTRAINT UK_UserFavorites UNIQUE (UserId, RecipeId)
);
```

#### ActivityLogs Table
Track user interactions for analytics and AI learning.

```sql
CREATE TABLE ActivityLogs (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId UNIQUEIDENTIFIER NOT NULL,
    ActivityType NVARCHAR(50) NOT NULL,
    EntityType NVARCHAR(50),
    EntityId UNIQUEIDENTIFIER,
    ActivityData NVARCHAR(MAX), -- JSON activity details
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_ActivityLogs_User FOREIGN KEY (UserId) REFERENCES Users(Id)
);
```

## Indexes

### Performance Indexes
```sql
-- User lookup performance
CREATE INDEX IX_Users_Email ON Users(Email);
CREATE INDEX IX_Users_Active ON Users(IsActive) WHERE IsActive = 1;

-- Recipe search performance
CREATE INDEX IX_Recipes_Category ON Recipes(Category);
CREATE INDEX IX_Recipes_Difficulty ON Recipes(Difficulty);
CREATE INDEX IX_Recipes_Public ON Recipes(IsPublic) WHERE IsPublic = 1;

-- Meal planning performance
CREATE INDEX IX_PlannedMeals_Date ON PlannedMeals(MealDate);
CREATE INDEX IX_PlannedMeals_Type ON PlannedMeals(MealType);

-- Family relationship performance
CREATE INDEX IX_FamilyMembers_Family ON FamilyMembers(FamilyId);
CREATE INDEX IX_MealPlans_Family ON MealPlans(FamilyId);
CREATE INDEX IX_ShoppingLists_Family ON ShoppingLists(FamilyId);
```

### Full-Text Search
```sql
-- Enable full-text search on recipes
CREATE FULLTEXT INDEX ON Recipes(Name, Description) KEY INDEX PK_Recipes;
```

## Entity Framework Configuration

### DbContext Example
```csharp
public class MealMenuDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<Family> Families { get; set; }
    public DbSet<FamilyMember> FamilyMembers { get; set; }
    public DbSet<Recipe> Recipes { get; set; }
    public DbSet<Ingredient> Ingredients { get; set; }
    public DbSet<MealPlan> MealPlans { get; set; }
    // ... other DbSets

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Configure relationships and constraints
        modelBuilder.Entity<User>()
            .HasIndex(u => u.Email)
            .IsUnique();

        modelBuilder.Entity<FamilyMember>()
            .Property(fm => fm.DietaryRestrictions)
            .HasConversion(
                v => JsonSerializer.Serialize(v),
                v => JsonSerializer.Deserialize<List<string>>(v));
    }
}
```

## Migration Strategy

### Development Migrations
- Use code-first migrations with Entity Framework
- Maintain migration scripts in version control
- Test migrations against sample data

### Production Deployment
- Generate SQL scripts from migrations
- Review and approve all schema changes
- Backup database before applying migrations
- Use blue-green deployment for zero downtime

## Data Seeding

### Initial Data
- Default ingredient catalog
- Sample recipe categories
- Common dietary restrictions
- Default user roles and permissions

### Test Data
- Sample families and members
- Test recipes with various complexities
- Mock AI suggestions for development

## Performance Considerations

### Query Optimization
- Use appropriate indexes for common queries
- Implement read replicas for heavy read operations
- Cache frequently accessed data in Redis
- Use pagination for large result sets

### Data Archiving
- Archive old meal plans and activity logs
- Implement soft deletes for user data
- Regular cleanup of expired AI suggestions