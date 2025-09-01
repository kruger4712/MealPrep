# Database Schema Overview

## Overview
Comprehensive overview of the MealPrep application database schema, including entity relationships, data models, and design decisions optimized for family meal planning and AI-driven personalization.

## Database Architecture

### Technology Stack
- **Primary Database**: SQL Server 2022 (Production)
- **Development Database**: SQL Server LocalDB or MySQL 8.0
- **ORM**: Entity Framework Core 8.0
- **Migrations**: EF Core Code-First Migrations
- **Connection Pooling**: Built-in EF Core connection pooling
- **Caching Layer**: Redis for query result caching

### Schema Design Principles
1. **Family-Centric Design**: All data is organized around family units
2. **AI Optimization**: Schema optimized for machine learning queries
3. **Audit Trail**: Comprehensive tracking of changes for learning
4. **Performance**: Proper indexing for common query patterns
5. **Scalability**: Designed to handle millions of users
6. **Data Integrity**: Strong foreign key relationships and constraints

## Core Entity Relationships

### High-Level ERD
```
Users ???????????? Families ???????????? FamilyMembers
   ?                   ?                       ?
   ?                   ?                       ?
   ??? UserPreferences ?                       ??? MemberPreferences
                       ?                              ?
                       ?                              ?
Recipes ???????????????????????????????????????????????? MealRatings
   ?                   ?                              ?
   ?                   ?                              ?
   ??? Ingredients     ??? MenuPlans ??????????????????
   ?                          ?
   ?                          ?
   ??? Instructions           ??? PlannedMeals
                                     ?
                                     ?
ShoppingLists ?????????????????????????
```

## Entity Models

### User Management

#### Users Table
```sql
CREATE TABLE Users (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Email NVARCHAR(256) NOT NULL UNIQUE,
    PasswordHash NVARCHAR(MAX) NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    EmailConfirmed BIT NOT NULL DEFAULT 0,
    PhoneNumber NVARCHAR(50) NULL,
    TimeZone NVARCHAR(100) NOT NULL DEFAULT 'UTC',
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    LastLoginAt DATETIME2 NULL,
    IsActive BIT NOT NULL DEFAULT 1,
    SubscriptionTier NVARCHAR(50) NOT NULL DEFAULT 'Free',
    
    INDEX IX_Users_Email (Email),
    INDEX IX_Users_CreatedAt (CreatedAt),
    INDEX IX_Users_IsActive (IsActive)
);
```

#### UserPreferences Table
```sql
CREATE TABLE UserPreferences (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId UNIQUEIDENTIFIER NOT NULL,
    DefaultMealBudget DECIMAL(8,2) NULL,
    PreferredCookingTime INT NULL, -- Minutes
    DefaultServingSize INT NULL,
    NotificationPreferences NVARCHAR(MAX) NULL, -- JSON
    PrivacySettings NVARCHAR(MAX) NULL, -- JSON
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    
    FOREIGN KEY (UserId) REFERENCES Users(Id) ON DELETE CASCADE,
    INDEX IX_UserPreferences_UserId (UserId)
);
```

### Family Management

#### Families Table
```sql
CREATE TABLE Families (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Name NVARCHAR(200) NOT NULL,
    CreatedByUserId UNIQUEIDENTIFIER NOT NULL,
    Description NVARCHAR(1000) NULL,
    AverageMealBudget DECIMAL(8,2) NULL,
    PreferredShoppingDay NVARCHAR(20) NULL,
    MealPlanningStyle NVARCHAR(50) NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    IsActive BIT NOT NULL DEFAULT 1,
    
    FOREIGN KEY (CreatedByUserId) REFERENCES Users(Id),
    INDEX IX_Families_CreatedByUserId (CreatedByUserId),
    INDEX IX_Families_IsActive (IsActive)
);
```

#### FamilyMembers Table
```sql
CREATE TABLE FamilyMembers (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FamilyId UNIQUEIDENTIFIER NOT NULL,
    Name NVARCHAR(100) NOT NULL,
    Age INT NOT NULL,
    Relationship NVARCHAR(50) NOT NULL,
    Gender NVARCHAR(20) NULL,
    ActivityLevel NVARCHAR(50) NOT NULL DEFAULT 'Moderate',
    DietaryRestrictions NVARCHAR(MAX) NULL, -- JSON array
    Allergies NVARCHAR(MAX) NULL, -- JSON array
    LikedIngredients NVARCHAR(MAX) NULL, -- JSON array
    DislikedIngredients NVARCHAR(MAX) NULL, -- JSON array
    CuisinePreferences NVARCHAR(MAX) NULL, -- JSON array
    SpiceToleranceLevel INT NOT NULL DEFAULT 5, -- 1-10 scale
    CookingSkillLevel NVARCHAR(50) NOT NULL DEFAULT 'Beginner',
    HealthGoals NVARCHAR(MAX) NULL, -- JSON array
    NutritionalNeeds NVARCHAR(MAX) NULL, -- JSON object
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    IsActive BIT NOT NULL DEFAULT 1,
    
    FOREIGN KEY (FamilyId) REFERENCES Families(Id) ON DELETE CASCADE,
    INDEX IX_FamilyMembers_FamilyId (FamilyId),
    INDEX IX_FamilyMembers_Age (Age),
    INDEX IX_FamilyMembers_IsActive (IsActive),
    
    CONSTRAINT CHK_FamilyMembers_Age CHECK (Age BETWEEN 0 AND 120),
    CONSTRAINT CHK_FamilyMembers_SpiceToleranceLevel CHECK (SpiceToleranceLevel BETWEEN 1 AND 10)
);
```

#### FamilyMemberships Table
```sql
CREATE TABLE FamilyMemberships (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId UNIQUEIDENTIFIER NOT NULL,
    FamilyId UNIQUEIDENTIFIER NOT NULL,
    Role NVARCHAR(50) NOT NULL DEFAULT 'Member',
    JoinedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    LeftAt DATETIME2 NULL,
    IsActive BIT NOT NULL DEFAULT 1,
    
    FOREIGN KEY (UserId) REFERENCES Users(Id),
    FOREIGN KEY (FamilyId) REFERENCES Families(Id) ON DELETE CASCADE,
    UNIQUE (UserId, FamilyId, IsActive), -- Prevent duplicate active memberships
    INDEX IX_FamilyMemberships_UserId (UserId),
    INDEX IX_FamilyMemberships_FamilyId (FamilyId)
);
```

### Recipe Management

#### Recipes Table
```sql
CREATE TABLE Recipes (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Name NVARCHAR(300) NOT NULL,
    Description NVARCHAR(2000) NULL,
    Instructions NVARCHAR(MAX) NOT NULL, -- JSON array
    PrepTime INT NOT NULL, -- Minutes
    CookTime INT NOT NULL, -- Minutes
    Servings INT NOT NULL,
    Difficulty NVARCHAR(20) NOT NULL DEFAULT 'Medium',
    Cuisine NVARCHAR(100) NULL,
    MealType NVARCHAR(50) NULL,
    EstimatedCost DECIMAL(8,2) NULL,
    NutritionalInfo NVARCHAR(MAX) NULL, -- JSON object
    Tags NVARCHAR(MAX) NULL, -- JSON array
    ImageUrl NVARCHAR(500) NULL,
    SourceUrl NVARCHAR(500) NULL,
    Source NVARCHAR(100) NOT NULL DEFAULT 'User Created',
    CreatedByUserId UNIQUEIDENTIFIER NULL,
    IsPublic BIT NOT NULL DEFAULT 0,
    IsVerified BIT NOT NULL DEFAULT 0,
    PopularityScore DECIMAL(5,2) NOT NULL DEFAULT 0.0,
    AverageRating DECIMAL(3,2) NULL,
    RatingCount INT NOT NULL DEFAULT 0,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    IsActive BIT NOT NULL DEFAULT 1,
    
    FOREIGN KEY (CreatedByUserId) REFERENCES Users(Id),
    INDEX IX_Recipes_Name (Name),
    INDEX IX_Recipes_Cuisine (Cuisine),
    INDEX IX_Recipes_MealType (MealType),
    INDEX IX_Recipes_Difficulty (Difficulty),
    INDEX IX_Recipes_PrepTime (PrepTime),
    INDEX IX_Recipes_CookTime (CookTime),
    INDEX IX_Recipes_EstimatedCost (EstimatedCost),
    INDEX IX_Recipes_IsPublic (IsPublic),
    INDEX IX_Recipes_PopularityScore (PopularityScore),
    INDEX IX_Recipes_AverageRating (AverageRating),
    INDEX IX_Recipes_CreatedAt (CreatedAt),
    
    CONSTRAINT CHK_Recipes_PrepTime CHECK (PrepTime >= 0),
    CONSTRAINT CHK_Recipes_CookTime CHECK (CookTime >= 0),
    CONSTRAINT CHK_Recipes_Servings CHECK (Servings > 0),
    CONSTRAINT CHK_Recipes_EstimatedCost CHECK (EstimatedCost >= 0),
    CONSTRAINT CHK_Recipes_PopularityScore CHECK (PopularityScore >= 0),
    CONSTRAINT CHK_Recipes_AverageRating CHECK (AverageRating BETWEEN 1 AND 5)
);
```

#### Ingredients Table
```sql
CREATE TABLE Ingredients (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Name NVARCHAR(200) NOT NULL UNIQUE,
    Description NVARCHAR(1000) NULL,
    Category NVARCHAR(100) NOT NULL,
    Subcategory NVARCHAR(100) NULL,
    CommonUnits NVARCHAR(MAX) NOT NULL, -- JSON array
    DefaultUnit NVARCHAR(50) NOT NULL,
    UnitConversions NVARCHAR(MAX) NULL, -- JSON object
    IsCommonAllergen BIT NOT NULL DEFAULT 0,
    AllergenInfo NVARCHAR(MAX) NULL, -- JSON object
    Seasonality NVARCHAR(MAX) NULL, -- JSON array
    AverageShelfLife INT NULL, -- Days
    StorageInstructions NVARCHAR(MAX) NULL, -- JSON object
    NutritionalInfo NVARCHAR(MAX) NULL, -- JSON object
    DietaryTags NVARCHAR(MAX) NULL, -- JSON array
    AverageCost DECIMAL(8,2) NULL,
    CostUnit NVARCHAR(50) NULL,
    Popularity INT NOT NULL DEFAULT 0,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    IsActive BIT NOT NULL DEFAULT 1,
    
    INDEX IX_Ingredients_Name (Name),
    INDEX IX_Ingredients_Category (Category),
    INDEX IX_Ingredients_IsCommonAllergen (IsCommonAllergen),
    INDEX IX_Ingredients_Popularity (Popularity),
    FULLTEXT INDEX FTX_Ingredients_Name_Description (Name, Description)
);
```

#### RecipeIngredients Table
```sql
CREATE TABLE RecipeIngredients (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    RecipeId UNIQUEIDENTIFIER NOT NULL,
    IngredientId UNIQUEIDENTIFIER NOT NULL,
    Quantity DECIMAL(10,3) NOT NULL,
    Unit NVARCHAR(50) NOT NULL,
    Preparation NVARCHAR(200) NULL, -- "diced", "chopped", etc.
    IsOptional BIT NOT NULL DEFAULT 0,
    DisplayOrder INT NOT NULL DEFAULT 0,
    Notes NVARCHAR(500) NULL,
    
    FOREIGN KEY (RecipeId) REFERENCES Recipes(Id) ON DELETE CASCADE,
    FOREIGN KEY (IngredientId) REFERENCES Ingredients(Id),
    UNIQUE (RecipeId, IngredientId),
    INDEX IX_RecipeIngredients_RecipeId (RecipeId),
    INDEX IX_RecipeIngredients_IngredientId (IngredientId),
    
    CONSTRAINT CHK_RecipeIngredients_Quantity CHECK (Quantity > 0)
);
```

### Menu Planning

#### MenuPlans Table
```sql
CREATE TABLE MenuPlans (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FamilyId UNIQUEIDENTIFIER NOT NULL,
    Name NVARCHAR(300) NOT NULL,
    Description NVARCHAR(1000) NULL,
    StartDate DATE NOT NULL,
    EndDate DATE NOT NULL,
    Status NVARCHAR(50) NOT NULL DEFAULT 'Draft',
    TotalEstimatedCost DECIMAL(10,2) NULL,
    MealTypes NVARCHAR(MAX) NOT NULL, -- JSON array
    Preferences NVARCHAR(MAX) NULL, -- JSON object
    AiGenerated BIT NOT NULL DEFAULT 0,
    CreatedByUserId UNIQUEIDENTIFIER NOT NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CompletedAt DATETIME2 NULL,
    
    FOREIGN KEY (FamilyId) REFERENCES Families(Id) ON DELETE CASCADE,
    FOREIGN KEY (CreatedByUserId) REFERENCES Users(Id),
    INDEX IX_MenuPlans_FamilyId (FamilyId),
    INDEX IX_MenuPlans_StartDate (StartDate),
    INDEX IX_MenuPlans_Status (Status),
    INDEX IX_MenuPlans_CreatedAt (CreatedAt),
    
    CONSTRAINT CHK_MenuPlans_DateRange CHECK (EndDate >= StartDate)
);
```

#### PlannedMeals Table
```sql
CREATE TABLE PlannedMeals (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    MenuPlanId UNIQUEIDENTIFIER NOT NULL,
    RecipeId UNIQUEIDENTIFIER NOT NULL,
    PlannedDate DATE NOT NULL,
    MealType NVARCHAR(50) NOT NULL,
    Servings INT NOT NULL,
    EstimatedCost DECIMAL(8,2) NULL,
    FamilyFitScore DECIMAL(3,1) NULL,
    Status NVARCHAR(50) NOT NULL DEFAULT 'Planned',
    Notes NVARCHAR(1000) NULL,
    MealPrepInstructions NVARCHAR(MAX) NULL, -- JSON array
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CookedAt DATETIME2 NULL,
    
    FOREIGN KEY (MenuPlanId) REFERENCES MenuPlans(Id) ON DELETE CASCADE,
    FOREIGN KEY (RecipeId) REFERENCES Recipes(Id),
    UNIQUE (MenuPlanId, PlannedDate, MealType),
    INDEX IX_PlannedMeals_MenuPlanId (MenuPlanId),
    INDEX IX_PlannedMeals_RecipeId (RecipeId),
    INDEX IX_PlannedMeals_PlannedDate (PlannedDate),
    INDEX IX_PlannedMeals_MealType (MealType),
    INDEX IX_PlannedMeals_Status (Status),
    
    CONSTRAINT CHK_PlannedMeals_Servings CHECK (Servings > 0),
    CONSTRAINT CHK_PlannedMeals_FamilyFitScore CHECK (FamilyFitScore BETWEEN 1 AND 10)
);
```

### AI and Learning Data

#### MealRatings Table
```sql
CREATE TABLE MealRatings (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FamilyMemberId UNIQUEIDENTIFIER NOT NULL,
    RecipeId UNIQUEIDENTIFIER NOT NULL,
    PlannedMealId UNIQUEIDENTIFIER NULL,
    Rating INT NOT NULL,
    MealType NVARCHAR(50) NOT NULL,
    Context NVARCHAR(200) NULL,
    PositiveFeedback NVARCHAR(MAX) NULL, -- JSON array
    NegativeFeedback NVARCHAR(MAX) NULL, -- JSON array
    WouldMakeAgain BIT NULL,
    RatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    
    FOREIGN KEY (FamilyMemberId) REFERENCES FamilyMembers(Id) ON DELETE CASCADE,
    FOREIGN KEY (RecipeId) REFERENCES Recipes(Id),
    FOREIGN KEY (PlannedMealId) REFERENCES PlannedMeals(Id),
    UNIQUE (FamilyMemberId, RecipeId, PlannedMealId),
    INDEX IX_MealRatings_FamilyMemberId (FamilyMemberId),
    INDEX IX_MealRatings_RecipeId (RecipeId),
    INDEX IX_MealRatings_Rating (Rating),
    INDEX IX_MealRatings_RatedAt (RatedAt),
    
    CONSTRAINT CHK_MealRatings_Rating CHECK (Rating BETWEEN 1 AND 5)
);
```

#### AiInteractions Table
```sql
CREATE TABLE AiInteractions (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId UNIQUEIDENTIFIER NOT NULL,
    FamilyId UNIQUEIDENTIFIER NULL,
    RequestType NVARCHAR(100) NOT NULL,
    RequestData NVARCHAR(MAX) NOT NULL, -- JSON
    ResponseData NVARCHAR(MAX) NULL, -- JSON
    ProcessingTimeMs BIGINT NULL,
    TokensUsed INT NULL,
    EstimatedCost DECIMAL(10,6) NULL,
    QualityScore DECIMAL(3,2) NULL,
    UserSatisfaction INT NULL, -- 1-5 scale
    Status NVARCHAR(50) NOT NULL,
    ErrorMessage NVARCHAR(MAX) NULL,
    FallbackStrategy NVARCHAR(100) NULL,
    ModelVersion NVARCHAR(50) NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    
    FOREIGN KEY (UserId) REFERENCES Users(Id),
    FOREIGN KEY (FamilyId) REFERENCES Families(Id),
    INDEX IX_AiInteractions_UserId (UserId),
    INDEX IX_AiInteractions_FamilyId (FamilyId),
    INDEX IX_AiInteractions_RequestType (RequestType),
    INDEX IX_AiInteractions_Status (Status),
    INDEX IX_AiInteractions_CreatedAt (CreatedAt),
    INDEX IX_AiInteractions_QualityScore (QualityScore)
);
```

### Shopping and Inventory

#### ShoppingLists Table
```sql
CREATE TABLE ShoppingLists (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FamilyId UNIQUEIDENTIFIER NOT NULL,
    MenuPlanId UNIQUEIDENTIFIER NULL,
    Name NVARCHAR(300) NOT NULL,
    Status NVARCHAR(50) NOT NULL DEFAULT 'Active',
    TotalEstimatedCost DECIMAL(10,2) NULL,
    ActualCost DECIMAL(10,2) NULL,
    ShoppingDate DATE NULL,
    Store NVARCHAR(200) NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CompletedAt DATETIME2 NULL,
    
    FOREIGN KEY (FamilyId) REFERENCES Families(Id) ON DELETE CASCADE,
    FOREIGN KEY (MenuPlanId) REFERENCES MenuPlans(Id),
    INDEX IX_ShoppingLists_FamilyId (FamilyId),
    INDEX IX_ShoppingLists_MenuPlanId (MenuPlanId),
    INDEX IX_ShoppingLists_Status (Status),
    INDEX IX_ShoppingLists_ShoppingDate (ShoppingDate)
);
```

#### ShoppingListItems Table
```sql
CREATE TABLE ShoppingListItems (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    ShoppingListId UNIQUEIDENTIFIER NOT NULL,
    IngredientId UNIQUEIDENTIFIER NOT NULL,
    Quantity DECIMAL(10,3) NOT NULL,
    Unit NVARCHAR(50) NOT NULL,
    EstimatedCost DECIMAL(8,2) NULL,
    ActualCost DECIMAL(8,2) NULL,
    Priority NVARCHAR(20) NOT NULL DEFAULT 'Medium',
    Category NVARCHAR(100) NULL,
    IsPurchased BIT NOT NULL DEFAULT 0,
    PurchasedAt DATETIME2 NULL,
    Store NVARCHAR(200) NULL,
    Notes NVARCHAR(500) NULL,
    
    FOREIGN KEY (ShoppingListId) REFERENCES ShoppingLists(Id) ON DELETE CASCADE,
    FOREIGN KEY (IngredientId) REFERENCES Ingredients(Id),
    INDEX IX_ShoppingListItems_ShoppingListId (ShoppingListId),
    INDEX IX_ShoppingListItems_IngredientId (IngredientId),
    INDEX IX_ShoppingListItems_Category (Category),
    INDEX IX_ShoppingListItems_IsPurchased (IsPurchased),
    
    CONSTRAINT CHK_ShoppingListItems_Quantity CHECK (Quantity > 0)
);
```

#### PantryItems Table
```sql
CREATE TABLE PantryItems (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FamilyId UNIQUEIDENTIFIER NOT NULL,
    IngredientId UNIQUEIDENTIFIER NOT NULL,
    Quantity DECIMAL(10,3) NOT NULL,
    Unit NVARCHAR(50) NOT NULL,
    PurchaseDate DATE NULL,
    ExpirationDate DATE NULL,
    Cost DECIMAL(8,2) NULL,
    Location NVARCHAR(100) NULL, -- "Pantry", "Refrigerator", "Freezer"
    Notes NVARCHAR(500) NULL,
    IsLowStock BIT NOT NULL DEFAULT 0,
    ReorderThreshold DECIMAL(10,3) NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    
    FOREIGN KEY (FamilyId) REFERENCES Families(Id) ON DELETE CASCADE,
    FOREIGN KEY (IngredientId) REFERENCES Ingredients(Id),
    INDEX IX_PantryItems_FamilyId (FamilyId),
    INDEX IX_PantryItems_IngredientId (IngredientId),
    INDEX IX_PantryItems_ExpirationDate (ExpirationDate),
    INDEX IX_PantryItems_Location (Location),
    INDEX IX_PantryItems_IsLowStock (IsLowStock),
    
    CONSTRAINT CHK_PantryItems_Quantity CHECK (Quantity >= 0)
);
```

## Entity Framework Models

### Base Entity
```csharp
public abstract class BaseEntity
{
    public Guid Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    public bool IsActive { get; set; } = true;
}

public abstract class AuditableEntity : BaseEntity
{
    public Guid? CreatedByUserId { get; set; }
    public Guid? UpdatedByUserId { get; set; }
    
    public virtual User? CreatedBy { get; set; }
    public virtual User? UpdatedBy { get; set; }
}
```

### Core Entities
```csharp
public class User : BaseEntity
{
    public string Email { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public bool EmailConfirmed { get; set; }
    public string? PhoneNumber { get; set; }
    public string TimeZone { get; set; } = "UTC";
    public DateTime? LastLoginAt { get; set; }
    public SubscriptionTier SubscriptionTier { get; set; } = SubscriptionTier.Free;
    
    // Navigation properties
    public virtual UserPreferences? Preferences { get; set; }
    public virtual ICollection<FamilyMembership> FamilyMemberships { get; set; } = new List<FamilyMembership>();
    public virtual ICollection<Recipe> CreatedRecipes { get; set; } = new List<Recipe>();
    public virtual ICollection<MenuPlan> CreatedMenuPlans { get; set; } = new List<MenuPlan>();
    public virtual ICollection<AiInteraction> AiInteractions { get; set; } = new List<AiInteraction>();
    
    // Computed properties
    public string FullName => $"{FirstName} {LastName}";
    public bool IsSubscriptionActive => SubscriptionTier != SubscriptionTier.Free;
}

public class Family : BaseEntity
{
    public string Name { get; set; } = string.Empty;
    public Guid CreatedByUserId { get; set; }
    public string? Description { get; set; }
    public decimal? AverageMealBudget { get; set; }
    public string? PreferredShoppingDay { get; set; }
    public MealPlanningStyle? MealPlanningStyle { get; set; }
    
    // Navigation properties
    public virtual User CreatedBy { get; set; } = null!;
    public virtual ICollection<FamilyMember> Members { get; set; } = new List<FamilyMember>();
    public virtual ICollection<FamilyMembership> Memberships { get; set; } = new List<FamilyMembership>();
    public virtual ICollection<MenuPlan> MenuPlans { get; set; } = new List<MenuPlan>();
    public virtual ICollection<ShoppingList> ShoppingLists { get; set; } = new List<ShoppingList>();
    public virtual ICollection<PantryItem> PantryItems { get; set; } = new List<PantryItem>();
}

public class FamilyMember : BaseEntity
{
    public Guid FamilyId { get; set; }
    public string Name { get; set; } = string.Empty;
    public int Age { get; set; }
    public FamilyRelationship Relationship { get; set; }
    public Gender? Gender { get; set; }
    public ActivityLevel ActivityLevel { get; set; } = ActivityLevel.Moderate;
    public List<DietaryRestriction> DietaryRestrictions { get; set; } = new();
    public List<string> Allergies { get; set; } = new();
    public List<string> LikedIngredients { get; set; } = new();
    public List<string> DislikedIngredients { get; set; } = new();
    public List<string> CuisinePreferences { get; set; } = new();
    public int SpiceToleranceLevel { get; set; } = 5;
    public CookingSkillLevel CookingSkillLevel { get; set; } = CookingSkillLevel.Beginner;
    public List<HealthGoal> HealthGoals { get; set; } = new();
    public NutritionalNeeds? NutritionalNeeds { get; set; }
    
    // Navigation properties
    public virtual Family Family { get; set; } = null!;
    public virtual ICollection<MealRating> MealRatings { get; set; } = new List<MealRating>();
}
```

## Database Configuration

### Connection Strings
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=MealPrepDev;Trusted_Connection=true;MultipleActiveResultSets=true",
    "ProductionConnection": "Server=mealprep-sql.database.windows.net;Database=MealPrepProd;User Id=mealprep-admin;Password={password};Encrypt=true;TrustServerCertificate=false;",
    "ReadOnlyConnection": "Server=mealprep-sql-readonly.database.windows.net;Database=MealPrepProd;User Id=mealprep-reader;Password={password};Encrypt=true;"
  }
}
```

### EF Core Configuration
```csharp
public class MealPrepDbContext : DbContext
{
    public MealPrepDbContext(DbContextOptions<MealPrepDbContext> options) : base(options) { }
    
    // DbSets
    public DbSet<User> Users { get; set; }
    public DbSet<Family> Families { get; set; }
    public DbSet<FamilyMember> FamilyMembers { get; set; }
    public DbSet<Recipe> Recipes { get; set; }
    public DbSet<Ingredient> Ingredients { get; set; }
    public DbSet<MenuPlan> MenuPlans { get; set; }
    public DbSet<PlannedMeal> PlannedMeals { get; set; }
    public DbSet<MealRating> MealRatings { get; set; }
    public DbSet<AiInteraction> AiInteractions { get; set; }
    public DbSet<ShoppingList> ShoppingLists { get; set; }
    public DbSet<PantryItem> PantryItems { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Apply all configurations
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(MealPrepDbContext).Assembly);
        
        // Global query filters
        modelBuilder.Entity<User>().HasQueryFilter(u => u.IsActive);
        modelBuilder.Entity<Family>().HasQueryFilter(f => f.IsActive);
        modelBuilder.Entity<Recipe>().HasQueryFilter(r => r.IsActive);
        
        // Seed data
        SeedData(modelBuilder);
    }
    
    public override int SaveChanges()
    {
        UpdateTimestamps();
        return base.SaveChanges();
    }
    
    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        UpdateTimestamps();
        return base.SaveChangesAsync(cancellationToken);
    }
    
    private void UpdateTimestamps()
    {
        var entries = ChangeTracker.Entries<BaseEntity>();
        
        foreach (var entry in entries)
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = DateTime.UtcNow;
                    entry.Entity.UpdatedAt = DateTime.UtcNow;
                    break;
                case EntityState.Modified:
                    entry.Entity.UpdatedAt = DateTime.UtcNow;
                    break;
            }
        }
    }
}
```

## Performance Optimization

### Key Indexes
```sql
-- High-traffic query indexes
CREATE INDEX IX_Recipes_Search ON Recipes (MealType, Cuisine, Difficulty) 
    INCLUDE (Name, PrepTime, CookTime, EstimatedCost);

CREATE INDEX IX_FamilyMembers_Preferences ON FamilyMembers (FamilyId, Age, IsActive)
    INCLUDE (DietaryRestrictions, Allergies, CuisinePreferences);

CREATE INDEX IX_PlannedMeals_Calendar ON PlannedMeals (MenuPlanId, PlannedDate, MealType)
    INCLUDE (RecipeId, Status, FamilyFitScore);

CREATE INDEX IX_MealRatings_Analysis ON MealRatings (FamilyMemberId, Rating, RatedAt)
    INCLUDE (RecipeId, MealType, WouldMakeAgain);

-- AI query optimization
CREATE INDEX IX_AiInteractions_Analytics ON AiInteractions (UserId, RequestType, CreatedAt)
    INCLUDE (QualityScore, UserSatisfaction, TokensUsed);
```

### Query Optimization Patterns
```csharp
// Optimized family preferences query
public async Task<List<Recipe>> GetFamilyRecommendationsAsync(Guid familyId, int count = 10)
{
    return await _context.Recipes
        .Where(r => r.IsPublic && r.IsActive)
        .Where(r => !r.RecipeIngredients.Any(ri => 
            _context.FamilyMembers
                .Where(fm => fm.FamilyId == familyId)
                .SelectMany(fm => fm.Allergies)
                .Contains(ri.Ingredient.Name)))
        .OrderByDescending(r => r.PopularityScore)
        .Take(count)
        .Include(r => r.RecipeIngredients)
            .ThenInclude(ri => ri.Ingredient)
        .ToListAsync();
}

// Optimized meal planning query
public async Task<MenuPlan> GetMenuPlanWithMealsAsync(Guid menuPlanId)
{
    return await _context.MenuPlans
        .Include(mp => mp.PlannedMeals)
            .ThenInclude(pm => pm.Recipe)
                .ThenInclude(r => r.RecipeIngredients)
                    .ThenInclude(ri => ri.Ingredient)
        .Include(mp => mp.Family)
            .ThenInclude(f => f.Members)
        .AsSplitQuery()
        .FirstOrDefaultAsync(mp => mp.Id == menuPlanId);
}
```

This database schema provides a robust foundation for the MealPrep application with proper normalization, performance optimization, and support for AI-driven personalization features.

*This schema should be updated as new features are added and performance requirements evolve.*