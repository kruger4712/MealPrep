# Family Member Endpoints

## Overview
API endpoints for managing family members and their dietary preferences, restrictions, and meal planning profiles in the MealPrep application.

## Base URL
```
https://api.mealprep.com/v1/families
```

## Endpoints

### Get User's Family Members

**Endpoint:** `GET /members`

**Description:** Retrieve all family members for the authenticated user.

**Headers:**
```
Authorization: Bearer [JWT_TOKEN]
```

**Success Response (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "name": "John Doe",
      "age": 35,
      "relationship": "Self",
      "avatar": "https://images.mealprep.com/avatars/user123.jpg",
      "dietaryRestrictions": ["None"],
      "allergies": ["Nuts"],
      "likedIngredients": ["chicken", "pasta", "tomatoes"],
      "dislikedIngredients": ["seafood", "mushrooms"],
      "cuisinePreferences": ["Italian", "American", "Mexican"],
      "spiceToleranceLevel": 5,
      "cookingSkillLevel": "Intermediate",
      "preferredCookingTime": 30,
      "healthGoals": ["Weight Loss", "High Protein"],
      "isActive": true,
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-15T00:00:00Z"
    },
    {
      "id": 2,
      "name": "Jane Doe",
      "age": 32,
      "relationship": "Spouse",
      "avatar": "https://images.mealprep.com/avatars/user456.jpg",
      "dietaryRestrictions": ["Vegetarian"],
      "allergies": ["Dairy"],
      "likedIngredients": ["vegetables", "tofu", "quinoa"],
      "dislikedIngredients": ["meat", "heavy sauces"],
      "cuisinePreferences": ["Mediterranean", "Asian", "Indian"],
      "spiceToleranceLevel": 7,
      "cookingSkillLevel": "Advanced",
      "preferredCookingTime": 45,
      "healthGoals": ["Heart Health", "More Vegetables"],
      "isActive": true,
      "createdAt": "2024-01-02T00:00:00Z",
      "updatedAt": "2024-01-16T00:00:00Z"
    }
  ],
  "metadata": {
    "timestamp": "2024-01-20T00:00:00Z",
    "requestId": "req_12345",
    "totalMembers": 2,
    "activeMembers": 2
  }
}
```

### Get Family Member by ID

**Endpoint:** `GET /members/{id}`

**Description:** Retrieve detailed information for a specific family member.

**Path Parameters:**
- `id` (integer) - Family member ID

**Success Response (200 OK):**
```json
{
  "data": {
    "id": 1,
    "name": "John Doe",
    "age": 35,
    "relationship": "Self",
    "avatar": "https://images.mealprep.com/avatars/user123.jpg",
    "dietaryRestrictions": ["None"],
    "allergies": ["Nuts"],
    "likedIngredients": ["chicken", "pasta", "tomatoes", "garlic", "onions"],
    "dislikedIngredients": ["seafood", "mushrooms", "blue cheese"],
    "cuisinePreferences": ["Italian", "American", "Mexican"],
    "spiceToleranceLevel": 5,
    "cookingSkillLevel": "Intermediate",
    "preferredCookingTime": 30,
    "healthGoals": ["Weight Loss", "High Protein"],
    "nutritionalPreferences": {
      "targetCalories": 2000,
      "targetProtein": 150,
      "targetCarbs": 200,
      "targetFat": 80,
      "targetFiber": 25,
      "maxSodium": 2300
    },
    "mealPreferences": {
      "breakfastImportance": 8,
      "lunchImportance": 6,
      "dinnerImportance": 9,
      "snackPreference": "Light",
      "mealTiming": {
        "breakfast": "07:00",
        "lunch": "12:30",
        "dinner": "18:30"
      }
    },
    "kitchenSkills": {
      "availableAppliances": ["oven", "stovetop", "microwave", "air fryer"],
      "comfortableWithKnife": true,
      "canFollowComplexRecipes": true,
      "preferredCookingMethods": ["baking", "sautéing", "grilling"]
    },
    "isActive": true,
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-15T00:00:00Z"
  },
  "metadata": {
    "timestamp": "2024-01-20T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Create Family Member

**Endpoint:** `POST /members`

**Description:** Add a new family member to the user's family.

**Headers:**
```
Authorization: Bearer [JWT_TOKEN]
Content-Type: application/json
```

**Request Body:**
```json
{
  "name": "Emma Doe",
  "age": 8,
  "relationship": "Child",
  "avatar": "https://images.mealprep.com/uploads/temp/avatar789.jpg",
  "dietaryRestrictions": ["None"],
  "allergies": ["Peanuts"],
  "likedIngredients": ["pasta", "chicken nuggets", "cheese", "apples"],
  "dislikedIngredients": ["vegetables", "spicy food", "fish"],
  "cuisinePreferences": ["American", "Italian"],
  "spiceToleranceLevel": 1,
  "healthGoals": ["Balanced Nutrition", "More Vegetables"],
  "nutritionalPreferences": {
    "targetCalories": 1400,
    "targetProtein": 50,
    "targetCarbs": 180,
    "targetFat": 50,
    "targetFiber": 15,
    "maxSodium": 1500
  },
  "mealPreferences": {
    "breakfastImportance": 9,
    "lunchImportance": 8,
    "dinnerImportance": 7,
    "snackPreference": "Healthy",
    "mealTiming": {
      "breakfast": "07:30",
      "lunch": "12:00",
      "dinner": "17:30"
    }
  }
}
```

**Validation Rules:**
```csharp
public class CreateFamilyMemberRequest
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; } = string.Empty;

    [Range(1, 120)]
    public int Age { get; set; }

    [Required]
    [StringLength(50)]
    public string Relationship { get; set; } = string.Empty;

    [Url]
    public string? Avatar { get; set; }

    [Required]
    public List<string> DietaryRestrictions { get; set; } = new();

    public List<string> Allergies { get; set; } = new();

    public List<string> LikedIngredients { get; set; } = new();

    public List<string> DislikedIngredients { get; set; } = new();

    public List<string> CuisinePreferences { get; set; } = new();

    [Range(1, 10)]
    public int SpiceToleranceLevel { get; set; } = 5;

    public List<string> HealthGoals { get; set; } = new();

    public NutritionalPreferences? NutritionalPreferences { get; set; }

    public MealPreferences? MealPreferences { get; set; }
}
```

**Success Response (201 Created):**
```json
{
  "data": {
    "id": 3,
    "name": "Emma Doe",
    "age": 8,
    "relationship": "Child",
    "dietaryRestrictions": ["None"],
    "allergies": ["Peanuts"],
    "isActive": true,
    "createdAt": "2024-01-20T00:00:00Z"
  },
  "metadata": {
    "timestamp": "2024-01-20T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Update Family Member

**Endpoint:** `PUT /members/{id}`

**Description:** Update an existing family member's information.

**Path Parameters:**
- `id` (integer) - Family member ID

**Request Body:** Same as Create Family Member

**Success Response (200 OK):** Same format as Get Family Member by ID

### Partial Update Family Member

**Endpoint:** `PATCH /members/{id}`

**Description:** Partially update a family member's information.

**Request Body:**
```json
{
  "spiceToleranceLevel": 6,
  "likedIngredients": ["chicken", "pasta", "tomatoes", "garlic", "bell peppers"],
  "healthGoals": ["Weight Loss", "High Protein", "More Energy"]
}
```

### Delete Family Member

**Endpoint:** `DELETE /members/{id}`

**Description:** Remove a family member (soft delete).

**Success Response (204 No Content)**

### Get Family Member Meal History

**Endpoint:** `GET /members/{id}/meal-history`

**Description:** Get meal history and preferences for a family member.

**Query Parameters:**
- `days` (integer, default: 30) - Number of days to look back
- `mealType` (string) - Filter by meal type (breakfast, lunch, dinner, snack)

**Success Response (200 OK):**
```json
{
  "data": {
    "memberId": 1,
    "memberName": "John Doe",
    "periodDays": 30,
    "mealHistory": [
      {
        "date": "2024-01-19",
        "mealType": "dinner",
        "recipeName": "Chicken Parmesan",
        "rating": 5,
        "notes": "Loved it! Request again soon."
      },
      {
        "date": "2024-01-18",
        "mealType": "lunch",
        "recipeName": "Caesar Salad",
        "rating": 3,
        "notes": "Too much dressing"
      }
    ],
    "preferences": {
      "topLikedMeals": [
        "Chicken Parmesan",
        "Spaghetti Carbonara",
        "Grilled Salmon"
      ],
      "frequentCuisines": ["Italian", "American"],
      "averageRating": 4.2,
      "totalMealsRated": 45
    }
  }
}
```

### Update Family Member Preferences

**Endpoint:** `PUT /members/{id}/preferences`

**Description:** Update dietary preferences and restrictions for a family member.

**Request Body:**
```json
{
  "dietaryRestrictions": ["Vegetarian"],
  "allergies": ["Nuts", "Shellfish"],
  "likedIngredients": ["tofu", "quinoa", "vegetables"],
  "dislikedIngredients": ["meat", "fish"],
  "spiceToleranceLevel": 6,
  "healthGoals": ["Plant-Based Diet", "Heart Health"]
}
```

### Get Family Meal Compatibility

**Endpoint:** `GET /compatibility`

**Description:** Analyze meal compatibility across all family members.

**Success Response (200 OK):**
```json
{
  "data": {
    "familySize": 3,
    "activeMembers": 3,
    "compatibility": {
      "overallScore": 7.5,
      "commonDietaryRestrictions": ["None"],
      "sharedAllergies": [],
      "commonLikedIngredients": ["chicken", "pasta"],
      "commonDislikedIngredients": ["mushrooms"],
      "spiceToleranceRange": {
        "min": 1,
        "max": 7,
        "average": 4.3
      },
      "cuisineCompatibility": {
        "highCompatibility": ["Italian", "American"],
        "mediumCompatibility": ["Mexican"],
        "lowCompatibility": ["Indian", "Thai"]
      }
    },
    "recommendations": [
      "Consider mild spice levels for family meals",
      "Italian cuisine works well for everyone",
      "Avoid mushrooms in shared meals",
      "Emma needs kid-friendly versions of adult meals"
    ]
  }
}
```

### Generate Family Persona

**Endpoint:** `POST /persona`

**Description:** Generate AI-optimized family persona for meal suggestions.

**Request Body:**
```json
{
  "includeMembers": [1, 2, 3],
  "mealContext": {
    "mealType": "dinner",
    "occasion": "weeknight",
    "budget": 25.00,
    "maxPrepTime": 30
  }
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "familyPersona": {
      "familySize": 3,
      "ageRange": "8-35",
      "dietaryProfile": {
        "primaryRestrictions": ["Vegetarian (1 member)"],
        "criticalAllergies": ["Nuts", "Peanuts"],
        "spicePreference": "Mild to Medium",
        "cuisinePreferences": ["Italian", "American"]
      },
      "cookingProfile": {
        "skillLevel": "Intermediate",
        "timeConstraints": "30 minutes average",
        "equipmentAvailable": ["oven", "stovetop", "microwave", "air fryer"]
      },
      "nutritionalGoals": {
        "primary": ["Balanced Nutrition", "Heart Health"],
        "considerations": ["Child-friendly portions", "Vegetarian options"]
      },
      "aiPromptData": {
        "familyContext": "Family of 3 with mixed dietary needs...",
        "constraints": "Must accommodate vegetarian and child preferences...",
        "optimization": "Focus on mild flavors with healthy options..."
      }
    },
    "generatedAt": "2024-01-20T00:00:00Z",
    "validUntil": "2024-01-27T00:00:00Z"
  }
}
```

## Validation and Business Rules

### Family Member Constraints
- Maximum 10 family members per user
- At least one family member must be active
- Names must be unique within a family
- Age must be realistic (1-120 years)
- Spice tolerance level must be 1-10

### Dietary Restrictions
**Supported Values:**
- None
- Vegetarian
- Vegan
- Keto
- Paleo
- Mediterranean
- Low Carb
- Low Fat
- Gluten Free
- Dairy Free
- Nut Free
- Halal
- Kosher

### Relationship Types
**Supported Values:**
- Self
- Spouse/Partner
- Child
- Parent
- Sibling
- Other Family
- Roommate
- Friend

## Authorization

### Family Member Permissions
- **Create**: User can add family members to their own family
- **Read**: User can view their own family members only
- **Update**: User can modify their own family members only
- **Delete**: User can remove their own family members only

### Data Privacy
- Family member data is private to the user
- No sharing of family information between users
- Anonymized data may be used for AI training (opt-in only)

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| FAM_001 | 400 | Invalid family member data |
| FAM_002 | 404 | Family member not found |
| FAM_003 | 403 | Not authorized to access family member |
| FAM_004 | 409 | Family member name already exists |
| FAM_005 | 413 | Avatar image too large |
| FAM_006 | 422 | Maximum family members reached (10) |
| FAM_007 | 422 | Cannot delete last active family member |

## Rate Limiting

- **Read operations**: 1000 requests per hour per user
- **Create/Update**: 100 requests per hour per user
- **Persona generation**: 50 requests per hour per user

## Implementation Example

### Controller Implementation
```csharp
[ApiController]
[Route("api/v1/families")]
[Authorize]
public class FamilyMembersController : ControllerBase
{
    private readonly IFamilyMemberService _familyMemberService;
    private readonly ILogger<FamilyMembersController> _logger;

    public FamilyMembersController(
        IFamilyMemberService familyMemberService,
        ILogger<FamilyMembersController> logger)
    {
        _familyMemberService = familyMemberService;
        _logger = logger;
    }

    /// <summary>
    /// Get all family members for the authenticated user
    /// </summary>
    [HttpGet("members")]
    [ProducesResponseType(typeof(ApiResponse<List<FamilyMemberDto>>), StatusCodes.Status200OK)]
    public async Task<ActionResult<ApiResponse<List<FamilyMemberDto>>>> GetFamilyMembers()
    {
        var userId = User.GetUserId();
        var familyMembers = await _familyMemberService.GetFamilyMembersAsync(userId);
        
        return Ok(new ApiResponse<List<FamilyMemberDto>>
        {
            Data = familyMembers,
            Metadata = new RequestMetadata
            {
                Timestamp = DateTime.UtcNow,
                RequestId = HttpContext.TraceIdentifier,
                TotalMembers = familyMembers.Count,
                ActiveMembers = familyMembers.Count(m => m.IsActive)
            }
        });
    }

    /// <summary>
    /// Create a new family member
    /// </summary>
    [HttpPost("members")]
    [ProducesResponseType(typeof(ApiResponse<FamilyMemberDto>), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<ApiResponse<FamilyMemberDto>>> CreateFamilyMember(
        [FromBody] CreateFamilyMemberRequest request)
    {
        try
        {
            var userId = User.GetUserId();
            var familyMember = await _familyMemberService.CreateFamilyMemberAsync(userId, request);
            
            return CreatedAtAction(
                nameof(GetFamilyMember), 
                new { id = familyMember.Id }, 
                new ApiResponse<FamilyMemberDto>
                {
                    Data = familyMember,
                    Metadata = new RequestMetadata
                    {
                        Timestamp = DateTime.UtcNow,
                        RequestId = HttpContext.TraceIdentifier
                    }
                });
        }
        catch (MaxFamilyMembersExceededException)
        {
            return UnprocessableEntity(CreateErrorResponse("FAM_006", "Maximum family members reached (10)"));
        }
        catch (DuplicateNameException ex)
        {
            return Conflict(CreateErrorResponse("FAM_004", ex.Message, "name", request.Name));
        }
    }
}
```

---

These family member endpoints provide comprehensive family management functionality for the MealPrep application with proper validation, authorization, and family meal planning optimization.

*Documentation updated for API version 1.0*