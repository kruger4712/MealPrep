# Menu Planning Endpoints

## Overview
API endpoints for weekly and monthly menu planning, meal scheduling, shopping list generation, and meal prep organization in the MealPrep application.

## Base URL
```
https://api.mealprep.com/v1/menus
```

## Endpoints

### Get User's Menu Plans

**Endpoint:** `GET /plans`

**Description:** Retrieve all menu plans for the authenticated user with optional filtering.

**Headers:**
```
Authorization: Bearer [JWT_TOKEN]
```

**Query Parameters:**
- `status` (string) - Filter by status: "active", "completed", "draft"
- `startDate` (date) - Filter plans starting from this date
- `endDate` (date) - Filter plans ending before this date
- `page` (integer, default: 1) - Page number for pagination
- `pageSize` (integer, default: 20) - Items per page

**Example Request:**
```
GET /api/v1/menus/plans?status=active&startDate=2024-01-20&pageSize=10
```

**Success Response (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Family Week - January 22-28",
      "startDate": "2024-01-22",
      "endDate": "2024-01-28",
      "status": "active",
      "totalMeals": 21,
      "plannedMeals": 18,
      "estimatedCost": 142.75,
      "familyMembers": [1, 2, 3],
      "mealTypes": ["breakfast", "lunch", "dinner"],
      "preferences": {
        "varietyScore": 8,
        "healthScore": 7,
        "budgetCompliance": 95
      },
      "createdAt": "2024-01-20T00:00:00Z",
      "updatedAt": "2024-01-21T00:00:00Z",
      "isAiGenerated": true
    },
    {
      "id": 2,
      "name": "Quick Weekday Meals",
      "startDate": "2024-01-29",
      "endDate": "2024-02-02",
      "status": "draft",
      "totalMeals": 15,
      "plannedMeals": 8,
      "estimatedCost": 89.50,
      "familyMembers": [1, 2],
      "mealTypes": ["lunch", "dinner"],
      "isAiGenerated": false
    }
  ],
  "metadata": {
    "timestamp": "2024-01-21T00:00:00Z",
    "pagination": {
      "page": 1,
      "pageSize": 10,
      "totalCount": 2,
      "totalPages": 1
    }
  }
}
```

### Get Menu Plan by ID

**Endpoint:** `GET /plans/{id}`

**Description:** Retrieve detailed information for a specific menu plan including all scheduled meals.

**Path Parameters:**
- `id` (integer) - Menu plan ID

**Success Response (200 OK):**
```json
{
  "data": {
    "id": 1,
    "name": "Family Week - January 22-28",
    "description": "AI-generated weekly menu plan optimized for family preferences",
    "startDate": "2024-01-22",
    "endDate": "2024-01-28",
    "status": "active",
    "familyMembers": [
      {
        "id": 1,
        "name": "John Doe",
        "dietaryRestrictions": ["None"]
      },
      {
        "id": 2,
        "name": "Jane Doe",
        "dietaryRestrictions": ["Vegetarian"]
      }
    ],
    "weeklySchedule": [
      {
        "date": "2024-01-22",
        "dayOfWeek": "Monday",
        "meals": {
          "breakfast": {
            "id": 101,
            "recipeId": 45,
            "recipeName": "Overnight Oats with Berries",
            "servings": 4,
            "prepTime": 5,
            "estimatedCost": 4.50,
            "familyFitScore": 8,
            "status": "planned",
            "notes": "Prep Sunday night"
          },
          "lunch": {
            "id": 102,
            "recipeId": 67,
            "recipeName": "Turkey and Avocado Wrap",
            "servings": 2,
            "prepTime": 10,
            "estimatedCost": 8.75,
            "familyFitScore": 7,
            "status": "planned"
          },
          "dinner": {
            "id": 103,
            "recipeId": 23,
            "recipeName": "Sheet Pan Chicken and Vegetables",
            "servings": 4,
            "prepTime": 15,
            "cookTime": 25,
            "estimatedCost": 16.25,
            "familyFitScore": 9,
            "status": "planned",
            "mealPrepNotes": "Cut vegetables Sunday"
          }
        },
        "dailyTotals": {
          "estimatedCost": 29.50,
          "totalPrepTime": 30,
          "averageFamilyFit": 8.0
        }
      }
    ],
    "shoppingList": {
      "id": 201,
      "status": "generated",
      "totalEstimatedCost": 138.75,
      "itemCount": 45,
      "lastUpdated": "2024-01-21T00:00:00Z"
    },
    "mealPrepPlan": {
      "id": 301,
      "totalPrepTime": 180,
      "prepDays": ["Sunday"],
      "taskCount": 12
    },
    "nutritionalSummary": {
      "dailyAverageCalories": 1850,
      "weeklyProtein": 665,
      "weeklyFiber": 175,
      "nutritionalGoalCompliance": 87
    },
    "budgetSummary": {
      "plannedBudget": 150.00,
      "estimatedCost": 142.75,
      "budgetCompliance": 95,
      "costPerMeal": 6.80,
      "costPerPerson": 23.79
    }
  }
}
```

### Create Menu Plan

**Endpoint:** `POST /plans`

**Description:** Create a new menu plan manually or with AI assistance.

**Request Body:**
```json
{
  "name": "Custom Family Menu",
  "description": "Manually planned menu for special dietary needs",
  "startDate": "2024-01-29",
  "endDate": "2024-02-04",
  "familyMembers": [1, 2, 3],
  "mealTypes": ["breakfast", "lunch", "dinner"],
  "preferences": {
    "maxBudget": 160.00,
    "varietyImportance": 8,
    "healthImportance": 9,
    "convenienceImportance": 6,
    "useAiSuggestions": true,
    "allowLeftovers": true,
    "mealPrepDay": "Sunday"
  },
  "constraints": {
    "busyDays": ["Wednesday", "Friday"],
    "specialOccasions": [
      {
        "date": "2024-02-01",
        "occasion": "Birthday Dinner",
        "mealType": "dinner",
        "specialBudget": 50.00
      }
    ],
    "avoidIngredients": ["shellfish", "nuts"],
    "preferredCuisines": ["Italian", "Mediterranean", "American"]
  }
}
```

**Success Response (201 Created):**
```json
{
  "data": {
    "id": 3,
    "name": "Custom Family Menu",
    "startDate": "2024-01-29",
    "endDate": "2024-02-04",
    "status": "draft",
    "familyMembers": [1, 2, 3],
    "estimatedCost": 155.25,
    "totalMeals": 21,
    "plannedMeals": 0,
    "createdAt": "2024-01-21T00:00:00Z",
    "aiSuggestionsAvailable": true
  }
}
```

### Update Menu Plan

**Endpoint:** `PUT /plans/{id}`

**Description:** Update an existing menu plan's basic information and preferences.

**Request Body:** Same as Create Menu Plan

**Success Response (200 OK):** Same format as Get Menu Plan by ID

### Add Meal to Menu Plan

**Endpoint:** `POST /plans/{id}/meals`

**Description:** Add a specific meal to a menu plan for a particular date and meal type.

**Path Parameters:**
- `id` (integer) - Menu plan ID

**Request Body:**
```json
{
  "date": "2024-01-22",
  "mealType": "dinner",
  "recipeId": 45,
  "servings": 4,
  "notes": "Make extra for leftovers",
  "mealPrepInstructions": [
    "Marinate chicken night before",
    "Chop vegetables in the morning"
  ],
  "schedulePrep": {
    "prepDay": "Sunday",
    "prepTimeMinutes": 20,
    "tasks": ["marinate", "chop vegetables"]
  }
}
```

**Success Response (201 Created):**
```json
{
  "data": {
    "id": 105,
    "date": "2024-01-22",
    "mealType": "dinner",
    "recipe": {
      "id": 45,
      "name": "Herb Crusted Chicken",
      "prepTime": 20,
      "cookTime": 25
    },
    "servings": 4,
    "estimatedCost": 18.50,
    "familyFitScore": 9,
    "status": "planned",
    "notes": "Make extra for leftovers",
    "addedAt": "2024-01-21T00:00:00Z"
  }
}
```

### Update Meal in Menu Plan

**Endpoint:** `PUT /plans/{planId}/meals/{mealId}`

**Description:** Update a specific meal in a menu plan.

**Path Parameters:**
- `planId` (integer) - Menu plan ID
- `mealId` (integer) - Meal ID

**Request Body:**
```json
{
  "servings": 6,
  "notes": "Updated for dinner party",
  "status": "completed"
}
```

### Remove Meal from Menu Plan

**Endpoint:** `DELETE /plans/{planId}/meals/{mealId}`

**Description:** Remove a meal from a menu plan.

**Success Response (204 No Content)**

### Generate AI Menu Suggestions

**Endpoint:** `POST /plans/{id}/ai-suggestions`

**Description:** Generate AI-powered meal suggestions for empty slots in a menu plan.

**Path Parameters:**
- `id` (integer) - Menu plan ID

**Request Body:**
```json
{
  "targetDates": ["2024-01-23", "2024-01-24"],
  "mealTypes": ["lunch", "dinner"],
  "regenerateExisting": false,
  "preferences": {
    "prioritizeHealth": true,
    "maxPrepTime": 45,
    "costLimit": 20.00,
    "cuisineVariety": true
  }
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "suggestions": [
      {
        "date": "2024-01-23",
        "mealType": "lunch",
        "options": [
          {
            "recipeId": 78,
            "recipeName": "Mediterranean Quinoa Bowl",
            "familyFitScore": 8,
            "estimatedCost": 12.50,
            "prepTime": 20,
            "reasoning": "High protein, vegetarian-friendly, uses pantry ingredients"
          },
          {
            "recipeId": 82,
            "recipeName": "Chicken Caesar Salad",
            "familyFitScore": 7,
            "estimatedCost": 14.75,
            "prepTime": 15,
            "reasoning": "Quick preparation, family favorite ingredients"
          }
        ]
      }
    ],
    "overallRecommendations": [
      "Consider batch cooking grains on Sunday for quick lunch assembly",
      "Your family enjoys Mediterranean flavors - included 2 Mediterranean-style meals"
    ],
    "processingTime": "3.2s",
    "aiModel": "gemini-pro"
  }
}
```

### Get Shopping List

**Endpoint:** `GET /plans/{id}/shopping-list`

**Description:** Get the consolidated shopping list for a menu plan.

**Query Parameters:**
- `format` (string) - "grouped" or "store-optimized" (default: "grouped")
- `includeOwned` (boolean) - Include items user already owns (default: false)

**Success Response (200 OK):**
```json
{
  "data": {
    "id": 201,
    "menuPlanId": 1,
    "status": "generated",
    "generatedAt": "2024-01-21T00:00:00Z",
    "totalEstimatedCost": 138.75,
    "itemCount": 45,
    "categories": [
      {
        "name": "Proteins",
        "items": [
          {
            "name": "Chicken breast",
            "quantity": 3,
            "unit": "lbs",
            "estimatedCost": 18.00,
            "priority": "high",
            "usedInMeals": [
              {
                "date": "2024-01-22",
                "mealType": "dinner",
                "recipeName": "Sheet Pan Chicken"
              },
              {
                "date": "2024-01-24",
                "mealType": "lunch",
                "recipeName": "Chicken Caesar Salad"
              }
            ]
          }
        ],
        "categoryTotal": 45.50
      },
      {
        "name": "Vegetables",
        "items": [
          {
            "name": "Bell peppers",
            "quantity": 6,
            "unit": "pieces",
            "estimatedCost": 4.50,
            "priority": "medium"
          }
        ],
        "categoryTotal": 28.75
      }
    ],
    "storeOptimization": {
      "primaryStore": "Whole Foods",
      "primaryStoreItems": 38,
      "primaryStoreCost": 125.25,
      "secondaryStore": "Costco",
      "secondaryStoreItems": 7,
      "secondaryStoreCost": 13.50
    }
  }
}
```

### Update Shopping List

**Endpoint:** `PUT /plans/{planId}/shopping-list/{listId}`

**Description:** Update shopping list items (mark as purchased, adjust quantities, etc.).

**Request Body:**
```json
{
  "items": [
    {
      "name": "Chicken breast",
      "purchased": true,
      "actualCost": 19.50,
      "actualQuantity": 3.2,
      "store": "Whole Foods",
      "purchaseDate": "2024-01-21"
    }
  ]
}
```

### Get Meal Prep Plan

**Endpoint:** `GET /plans/{id}/meal-prep`

**Description:** Get the meal prep schedule and tasks for a menu plan.

**Success Response (200 OK):**
```json
{
  "data": {
    "id": 301,
    "menuPlanId": 1,
    "totalPrepTime": 180,
    "efficiency": 85,
    "prepSchedule": [
      {
        "day": "Sunday",
        "totalTime": 120,
        "tasks": [
          {
            "id": 1,
            "name": "Prep overnight oats for the week",
            "timeEstimate": 20,
            "difficulty": "Easy",
            "category": "breakfast-prep",
            "relatedMeals": ["Monday breakfast", "Tuesday breakfast"],
            "instructions": [
              "Mix oats, milk, and yogurt in 5 mason jars",
              "Add different toppings to each jar",
              "Refrigerate until ready to serve"
            ]
          },
          {
            "id": 2,
            "name": "Marinate chicken for Monday dinner",
            "timeEstimate": 10,
            "difficulty": "Easy",
            "category": "protein-prep",
            "relatedMeals": ["Monday dinner"],
            "instructions": [
              "Mix marinade ingredients",
              "Add chicken to marinade",
              "Refrigerate overnight"
            ]
          },
          {
            "id": 3,
            "name": "Wash and chop vegetables",
            "timeEstimate": 45,
            "difficulty": "Medium",
            "category": "vegetable-prep",
            "relatedMeals": ["Monday dinner", "Tuesday lunch", "Wednesday dinner"],
            "instructions": [
              "Wash all vegetables thoroughly",
              "Chop bell peppers and store in containers",
              "Prepare salad greens and store properly"
            ]
          }
        ]
      }
    ],
    "tips": [
      "Cook grains in bulk and portion for the week",
      "Prep vegetables right after grocery shopping for freshness",
      "Use meal prep containers for easy grab-and-go lunches"
    ]
  }
}
```

### Complete Menu Plan

**Endpoint:** `POST /plans/{id}/complete`

**Description:** Mark a menu plan as completed and generate analytics.

**Success Response (200 OK):**
```json
{
  "data": {
    "menuPlanId": 1,
    "completedAt": "2024-01-28T23:59:59Z",
    "status": "completed",
    "analytics": {
      "mealsCompleted": 18,
      "totalMealsPlanned": 21,
      "completionRate": 86,
      "actualCost": 145.30,
      "budgetVariance": -2.55,
      "familySatisfaction": {
        "averageRating": 4.2,
        "topRatedMeals": [
          "Sheet Pan Chicken and Vegetables",
          "Mediterranean Quinoa Bowl"
        ],
        "leastRatedMeals": [
          "Brussels Sprouts Salad"
        ]
      },
      "nutritionalGoalAchievement": 89,
      "mealPrepEfficiency": 92,
      "recommendations": [
        "Consider adding more Brussels sprouts recipes to improve family acceptance",
        "Your meal prep efficiency was excellent - keep Sunday prep routine",
        "Budget was well-managed with only $2.55 over estimate"
      ]
    }
  }
}
```

### Copy Menu Plan

**Endpoint:** `POST /plans/{id}/copy`

**Description:** Create a copy of an existing menu plan for a new date range.

**Request Body:**
```json
{
  "name": "Family Week - February 5-11",
  "startDate": "2024-02-05",
  "endDate": "2024-02-11",
  "adjustments": {
    "excludeMeals": [103, 107],
    "replaceMealTypes": {
      "breakfast": "quick-options"
    },
    "budgetAdjustment": 10.00
  }
}
```

## Validation Rules

### Menu Plan Validation
```csharp
public class CreateMenuPlanRequest
{
    [Required]
    [StringLength(200, MinimumLength = 3)]
    public string Name { get; set; } = string.Empty;

    [StringLength(1000)]
    public string Description { get; set; } = string.Empty;

    [Required]
    public DateTime StartDate { get; set; }

    [Required]
    public DateTime EndDate { get; set; }

    [Required]
    [MinLength(1)]
    public List<int> FamilyMembers { get; set; } = new();

    [Required]
    [MinLength(1)]
    public List<string> MealTypes { get; set; } = new();

    [Range(0, 10000)]
    public decimal MaxBudget { get; set; }

    [Range(1, 10)]
    public int VarietyImportance { get; set; } = 5;

    [Range(1, 10)]
    public int HealthImportance { get; set; } = 5;

    public MenuPlanPreferences Preferences { get; set; } = new();
}
```

### Business Rules
- Menu plans cannot exceed 4 weeks in duration
- Start date cannot be in the past (except for historical plans)
- Family members must belong to the authenticated user
- Maximum 100 meals per menu plan
- Budget must be positive value

## Authorization

### Menu Plan Permissions
- **Create**: Authenticated users can create menu plans for their families
- **Read**: Users can only view their own menu plans
- **Update**: Users can only modify their own menu plans
- **Delete**: Users can only delete their own menu plans
- **Copy**: Users can copy any of their own menu plans

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| MENU_001 | 400 | Invalid menu plan data |
| MENU_002 | 404 | Menu plan not found |
| MENU_003 | 403 | Not authorized to access menu plan |
| MENU_004 | 409 | Menu plan name conflict |
| MENU_005 | 422 | Date range exceeds maximum duration |
| MENU_006 | 422 | Meal already exists for date/time slot |
| MENU_007 | 424 | Shopping list generation failed |
| MENU_008 | 424 | AI suggestion service unavailable |

## Rate Limiting

- **Read operations**: 1000 requests per hour per user
- **Create/Update operations**: 200 requests per hour per user
- **AI suggestion generation**: 50 requests per hour per user
- **Shopping list generation**: 100 requests per hour per user

---

These menu planning endpoints provide comprehensive meal planning and organization functionality for the MealPrep application with intelligent scheduling, cost tracking, and meal prep optimization.

*Documentation updated for API version 1.0*