# Ingredient Management Endpoints

## Overview
API endpoints for managing ingredients, nutritional information, substitutions, and pantry inventory in the MealPrep application.

## Base URL
```
https://api.mealprep.com/v1/ingredients
```

## Endpoints

### Get All Ingredients

**Endpoint:** `GET /`

**Description:** Retrieve paginated list of ingredients with optional filtering and search.

**Query Parameters:**
- `search` (string) - Search by ingredient name or description
- `category` (string) - Filter by ingredient category
- `isCommonAllergen` (boolean) - Filter by allergen status
- `season` (string) - Filter by seasonal availability
- `dietaryTags` (string) - Comma-separated dietary tags (vegan, keto, etc.)
- `page` (integer, default: 1) - Page number
- `pageSize` (integer, default: 50, max: 200) - Items per page
- `sortBy` (string) - Sort by: "name", "category", "popularity" (default: "name")

**Example Request:**
```
GET /api/v1/ingredients?search=chicken&category=protein&page=1&pageSize=20
```

**Success Response (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Chicken Breast",
      "description": "Boneless, skinless chicken breast",
      "category": "Protein",
      "commonUnits": ["lb", "piece", "oz"],
      "defaultUnit": "lb",
      "isCommonAllergen": false,
      "seasonality": ["year-round"],
      "averageShelfLife": 3,
      "storageInstructions": "Refrigerate at 40°F or below, use within 1-2 days",
      "nutritionalInfo": {
        "caloriesPerUnit": 185,
        "proteinGrams": 35,
        "carbGrams": 0,
        "fatGrams": 4,
        "fiberGrams": 0,
        "sodiumMg": 74,
        "unitSize": "100g"
      },
      "dietaryTags": ["high-protein", "low-carb", "keto-friendly"],
      "commonSubstitutions": [
        {
          "ingredientId": 2,
          "name": "Chicken Thigh",
          "ratio": 1.0,
          "notes": "More flavorful, slightly higher fat content"
        },
        {
          "ingredientId": 15,
          "name": "Turkey Breast",
          "ratio": 1.0,
          "notes": "Similar protein content, leaner option"
        }
      ],
      "averageCost": 6.99,
      "costUnit": "lb",
      "popularity": 95,
      "createdAt": "2024-01-01T00:00:00Z"
    }
  ],
  "metadata": {
    "timestamp": "2024-01-21T00:00:00Z",
    "pagination": {
      "page": 1,
      "pageSize": 20,
      "totalCount": 1247,
      "totalPages": 63
    }
  }
}
```

### Get Ingredient by ID

**Endpoint:** `GET /{id}`

**Description:** Retrieve detailed information for a specific ingredient.

**Path Parameters:**
- `id` (integer) - Ingredient ID

**Success Response (200 OK):**
```json
{
  "data": {
    "id": 1,
    "name": "Chicken Breast",
    "description": "Boneless, skinless chicken breast - versatile protein for numerous cooking methods",
    "category": "Protein",
    "subcategory": "Poultry",
    "commonUnits": ["lb", "piece", "oz", "kg"],
    "defaultUnit": "lb",
    "unitConversions": {
      "1 lb": "16 oz",
      "1 piece": "6 oz",
      "1 lb": "0.45 kg"
    },
    "isCommonAllergen": false,
    "allergenInfo": {
      "contains": [],
      "mayContain": [],
      "processingWarnings": ["May be processed in facilities that handle dairy"]
    },
    "seasonality": ["year-round"],
    "peakSeason": null,
    "averageShelfLife": 3,
    "storageInstructions": {
      "refrigerator": "Store at 40°F or below, use within 1-2 days of purchase",
      "freezer": "Can be frozen for up to 9 months at 0°F",
      "pantry": "Not suitable for pantry storage"
    },
    "nutritionalInfo": {
      "referenceAmount": "100g",
      "calories": 185,
      "macronutrients": {
        "protein": 35.0,
        "carbohydrates": 0.0,
        "fat": 4.0,
        "fiber": 0.0
      },
      "micronutrients": {
        "sodium": 74,
        "potassium": 256,
        "iron": 0.9,
        "calcium": 11,
        "vitaminC": 0,
        "vitaminA": 21
      },
      "units": "mg unless specified"
    },
    "dietaryTags": ["high-protein", "low-carb", "keto-friendly", "paleo", "whole30"],
    "cookingMethods": [
      {
        "method": "Grilling",
        "timeMinutes": 12,
        "temperature": "Medium-high heat",
        "notes": "Cook until internal temperature reaches 165°F"
      },
      {
        "method": "Baking",
        "timeMinutes": 25,
        "temperature": "375°F",
        "notes": "Bake in preheated oven, covered with foil if needed"
      },
      {
        "method": "Pan-searing",
        "timeMinutes": 8,
        "temperature": "Medium-high heat",
        "notes": "Sear 3-4 minutes per side until golden brown"
      }
    ],
    "commonSubstitutions": [
      {
        "ingredientId": 2,
        "name": "Chicken Thigh",
        "ratio": 1.0,
        "notes": "More flavorful and moist, adjust cooking time",
        "nutritionalDifference": "+50 calories, +8g fat per 100g"
      },
      {
        "ingredientId": 15,
        "name": "Turkey Breast",
        "ratio": 1.0,
        "notes": "Similar cooking methods, slightly leaner",
        "nutritionalDifference": "-15 calories, -1g fat per 100g"
      },
      {
        "ingredientId": 89,
        "name": "Tofu (Extra Firm)",
        "ratio": 0.8,
        "notes": "Vegetarian substitute, press before cooking",
        "nutritionalDifference": "-85 calories, -27g protein, +2g carbs per 100g"
      }
    ],
    "pricingInfo": {
      "averageCost": 6.99,
      "costUnit": "lb",
      "priceRange": {
        "low": 4.99,
        "high": 12.99
      },
      "organicPremium": 2.50,
      "seasonalVariation": 15,
      "lastUpdated": "2024-01-20T00:00:00Z"
    },
    "popularRecipes": [
      {
        "recipeId": 23,
        "recipeName": "Herb Crusted Chicken",
        "usageCount": 1247
      },
      {
        "recipeId": 45,
        "recipeName": "Chicken Stir Fry",
        "usageCount": 986
      }
    ],
    "popularity": 95,
    "usageFrequency": "Very High",
    "tags": ["versatile", "lean-protein", "family-friendly"],
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-20T00:00:00Z"
  }
}
```

### Search Ingredients

**Endpoint:** `GET /search`

**Description:** Advanced search for ingredients with fuzzy matching and suggestions.

**Query Parameters:**
- `q` (string, required) - Search query
- `includeSubstitutions` (boolean, default: true) - Include substitution suggestions
- `dietary` (string) - Filter by dietary requirements (comma-separated)
- `excludeAllergens` (string) - Exclude specific allergens (comma-separated)
- `maxResults` (integer, default: 20, max: 100) - Maximum results to return

**Example Request:**
```
GET /api/v1/ingredients/search?q=chiken&dietary=keto,low-carb&maxResults=10
```

**Success Response (200 OK):**
```json
{
  "data": {
    "query": "chiken",
    "correctedQuery": "chicken",
    "results": [
      {
        "id": 1,
        "name": "Chicken Breast",
        "category": "Protein",
        "relevanceScore": 0.95,
        "matchType": "corrected_spelling",
        "nutritionalInfo": {
          "calories": 185,
          "protein": 35,
          "carbs": 0,
          "fat": 4
        },
        "averageCost": 6.99,
        "dietaryCompatibility": ["keto", "low-carb", "high-protein"]
      },
      {
        "id": 2,
        "name": "Chicken Thigh",
        "category": "Protein",
        "relevanceScore": 0.92,
        "matchType": "corrected_spelling",
        "nutritionalInfo": {
          "calories": 235,
          "protein": 27,
          "carbs": 0,
          "fat": 13
        },
        "averageCost": 4.99,
        "dietaryCompatibility": ["keto", "low-carb", "high-protein"]
      }
    ],
    "suggestions": [
      "Did you mean: chicken?",
      "Related searches: chicken breast, chicken thigh, chicken wings"
    ],
    "substitutionSuggestions": [
      {
        "originalIngredient": "Chicken Breast",
        "alternatives": [
          {
            "id": 15,
            "name": "Turkey Breast",
            "reason": "Similar protein content and cooking methods"
          },
          {
            "id": 89,
            "name": "Extra Firm Tofu",
            "reason": "Vegetarian protein alternative"
          }
        ]
      }
    ]
  }
}
```

### Create Custom Ingredient

**Endpoint:** `POST /`

**Description:** Create a custom ingredient for user's personal ingredient library.

**Headers:**
```
Authorization: Bearer [JWT_TOKEN]
Content-Type: application/json
```

**Request Body:**
```json
{
  "name": "Homemade Almond Butter",
  "description": "Freshly ground almond butter made at home",
  "category": "Spreads & Condiments",
  "isCustom": true,
  "commonUnits": ["tbsp", "cup", "jar"],
  "defaultUnit": "tbsp",
  "nutritionalInfo": {
    "referenceAmount": "2 tbsp",
    "calories": 196,
    "protein": 7.3,
    "carbohydrates": 7.6,
    "fat": 17.8,
    "fiber": 4.1,
    "sodium": 1
  },
  "allergenInfo": {
    "contains": ["tree nuts"],
    "mayContain": [],
    "processingWarnings": []
  },
  "dietaryTags": ["vegan", "gluten-free", "keto-friendly", "paleo"],
  "storageInstructions": {
    "pantry": "Store in airtight container for up to 3 months",
    "refrigerator": "Extends shelf life to 6 months"
  },
  "averageCost": 8.50,
  "costUnit": "jar"
}
```

**Validation Rules:**
```csharp
public class CreateIngredientRequest
{
    [Required]
    [StringLength(200, MinimumLength = 2)]
    public string Name { get; set; } = string.Empty;

    [StringLength(1000)]
    public string Description { get; set; } = string.Empty;

    [Required]
    [StringLength(100)]
    public string Category { get; set; } = string.Empty;

    [Required]
    [MinLength(1)]
    public List<string> CommonUnits { get; set; } = new();

    [Required]
    public string DefaultUnit { get; set; } = string.Empty;

    public NutritionalInfo NutritionalInfo { get; set; } = new();

    public AllergenInfo AllergenInfo { get; set; } = new();

    public List<string> DietaryTags { get; set; } = new();

    [Range(0, 999999.99)]
    public decimal AverageCost { get; set; }
}
```

**Success Response (201 Created):**
```json
{
  "data": {
    "id": 5001,
    "name": "Homemade Almond Butter",
    "category": "Spreads & Condiments",
    "isCustom": true,
    "createdBy": "user123",
    "visibility": "private",
    "createdAt": "2024-01-21T00:00:00Z"
  }
}
```

### Update Ingredient

**Endpoint:** `PUT /{id}`

**Description:** Update an existing ingredient (only custom ingredients can be updated by users).

**Request Body:** Same as Create Custom Ingredient

### Get Ingredient Substitutions

**Endpoint:** `GET /{id}/substitutions`

**Description:** Get detailed substitution options for a specific ingredient.

**Query Parameters:**
- `dietary` (string) - Filter by dietary requirements
- `reason` (string) - Filter by substitution reason: "allergen", "dietary", "availability", "cost"
- `includeRatios` (boolean, default: true) - Include conversion ratios

**Success Response (200 OK):**
```json
{
  "data": {
    "ingredientId": 1,
    "ingredientName": "Chicken Breast",
    "substitutions": [
      {
        "id": 2,
        "name": "Chicken Thigh",
        "category": "Direct Substitute",
        "ratio": 1.0,
        "conversionNotes": "Use same quantity, adjust cooking time by 5-10 minutes",
        "nutritionalImpact": {
          "calories": "+50 per 100g",
          "protein": "-8g per 100g",
          "fat": "+9g per 100g"
        },
        "flavorImpact": "Richer, more savory flavor",
        "textureImpact": "Slightly more tender and moist",
        "costImpact": -2.00,
        "availabilityScore": 95,
        "recommendationScore": 9,
        "suitableFor": ["all recipes", "slow cooking", "grilling"]
      },
      {
        "id": 89,
        "name": "Extra Firm Tofu",
        "category": "Vegetarian Alternative",
        "ratio": 0.8,
        "conversionNotes": "Press tofu to remove moisture, marinate for flavor",
        "nutritionalImpact": {
          "calories": "-85 per 100g",
          "protein": "-27g per 100g",
          "carbs": "+2g per 100g"
        },
        "flavorImpact": "Mild flavor, takes on marinade well",
        "textureImpact": "Firmer, less tender than chicken",
        "costImpact": -1.50,
        "availabilityScore": 85,
        "recommendationScore": 7,
        "suitableFor": ["stir-fries", "curries", "marinated dishes"],
        "preparationTips": [
          "Press tofu for 30 minutes before cooking",
          "Freeze and thaw for meatier texture",
          "Marinate for at least 2 hours for best flavor"
        ]
      }
    ],
    "substitutionTips": [
      "When substituting protein sources, consider cooking time adjustments",
      "Plant-based proteins often benefit from longer marinating times",
      "Consider texture preferences when choosing substitutions"
    ]
  }
}
```

### Get Ingredient Categories

**Endpoint:** `GET /categories`

**Description:** Get all ingredient categories with counts and popular items.

**Success Response (200 OK):**
```json
{
  "data": [
    {
      "name": "Proteins",
      "description": "Meat, poultry, fish, and plant-based proteins",
      "count": 156,
      "subcategories": [
        {
          "name": "Poultry",
          "count": 23,
          "popularItems": ["Chicken Breast", "Chicken Thigh", "Turkey Breast"]
        },
        {
          "name": "Seafood",
          "count": 45,
          "popularItems": ["Salmon", "Shrimp", "Tuna"]
        },
        {
          "name": "Plant-Based",
          "count": 34,
          "popularItems": ["Tofu", "Tempeh", "Lentils"]
        }
      ],
      "topIngredients": [
        {
          "id": 1,
          "name": "Chicken Breast",
          "popularity": 95
        },
        {
          "id": 45,
          "name": "Ground Beef",
          "popularity": 92
        }
      ]
    }
  ]
}
```

### Get Seasonal Ingredients

**Endpoint:** `GET /seasonal`

**Description:** Get ingredients that are currently in season or approaching peak season.

**Query Parameters:**
- `season` (string) - Specific season: "spring", "summer", "fall", "winter"
- `location` (string) - Geographic location for seasonal data (default: "US")
- `upcoming` (boolean, default: false) - Include ingredients coming into season

**Success Response (200 OK):**
```json
{
  "data": {
    "currentSeason": "winter",
    "location": "United States",
    "inSeason": [
      {
        "id": 234,
        "name": "Brussels Sprouts",
        "category": "Vegetables",
        "peakMonths": ["December", "January", "February"],
        "qualityScore": 95,
        "costAdvantage": 30,
        "nutritionalHighlights": ["Vitamin C", "Vitamin K", "Fiber"],
        "cookingTips": "Roast at high heat to caramelize and reduce bitterness"
      },
      {
        "id": 187,
        "name": "Winter Squash",
        "category": "Vegetables",
        "varieties": ["Butternut", "Acorn", "Delicata"],
        "peakMonths": ["November", "December", "January"],
        "qualityScore": 90,
        "storageLife": "3-6 months",
        "costAdvantage": 25
      }
    ],
    "comingSoon": [
      {
        "id": 345,
        "name": "Asparagus",
        "category": "Vegetables",
        "expectedPeak": "March",
        "weeksUntilPeak": 6
      }
    ]
  }
}
```

### Get User's Pantry

**Endpoint:** `GET /pantry`

**Description:** Get user's pantry inventory with expiration tracking.

**Headers:**
```
Authorization: Bearer [JWT_TOKEN]
```

**Query Parameters:**
- `expiringSoon` (boolean) - Show only items expiring within 7 days
- `category` (string) - Filter by ingredient category
- `lowStock` (boolean) - Show only items with low stock levels

**Success Response (200 OK):**
```json
{
  "data": {
    "pantryItems": [
      {
        "id": 501,
        "ingredientId": 1,
        "ingredientName": "Chicken Breast",
        "quantity": 2.5,
        "unit": "lbs",
        "purchaseDate": "2024-01-19",
        "expirationDate": "2024-01-22",
        "daysUntilExpiration": 1,
        "status": "expiring_soon",
        "location": "Refrigerator",
        "cost": 17.48,
        "notes": "From Whole Foods, organic"
      },
      {
        "id": 502,
        "ingredientId": 67,
        "ingredientName": "Olive Oil",
        "quantity": 0.5,
        "unit": "bottles",
        "purchaseDate": "2024-01-10",
        "expirationDate": "2025-01-10",
        "daysUntilExpiration": 354,
        "status": "good",
        "location": "Pantry",
        "cost": 12.99,
        "isLowStock": true,
        "reorderThreshold": 0.25
      }
    ],
    "summary": {
      "totalItems": 47,
      "expiringSoon": 3,
      "lowStock": 8,
      "estimatedValue": 234.67,
      "lastUpdated": "2024-01-21T00:00:00Z"
    },
    "alerts": [
      "Chicken Breast expires tomorrow",
      "8 items are running low and should be added to shopping list"
    ]
  }
}
```

### Add Item to Pantry

**Endpoint:** `POST /pantry`

**Description:** Add an ingredient to user's pantry inventory.

**Request Body:**
```json
{
  "ingredientId": 1,
  "quantity": 2.0,
  "unit": "lbs",
  "purchaseDate": "2024-01-21",
  "expirationDate": "2024-01-24",
  "cost": 13.98,
  "location": "Refrigerator",
  "notes": "Organic, on sale"
}
```

### Update Pantry Item

**Endpoint:** `PUT /pantry/{itemId}`

**Description:** Update quantity, location, or other details of a pantry item.

**Request Body:**
```json
{
  "quantity": 1.5,
  "location": "Freezer",
  "notes": "Moved to freezer to extend shelf life"
}
```

### Remove from Pantry

**Endpoint:** `DELETE /pantry/{itemId}`

**Description:** Remove an item from pantry (used up, expired, etc.).

**Query Parameters:**
- `reason` (string) - Reason for removal: "used", "expired", "spoiled", "given_away"
- `usedInRecipe` (integer) - Recipe ID if used in cooking

## Validation Rules

### Ingredient Constraints
- Name must be unique within user's custom ingredients
- Categories must be from predefined list or user-created custom categories
- Nutritional information must have valid reference amounts
- Cost information must be positive values
- Allergen information must use standardized allergen codes

### Pantry Constraints
- Quantities must be positive numbers
- Expiration dates cannot be in the past (with 1-day grace period)
- Location must be from valid storage locations
- Users can have maximum 500 pantry items

## Authorization

### Ingredient Permissions
- **Read Public Ingredients**: All users can view public ingredient database
- **Create Custom Ingredients**: Authenticated users only
- **Update Custom Ingredients**: Only the creator can modify
- **Pantry Management**: Users can only manage their own pantry

### Data Privacy
- Custom ingredients are private by default
- Users can optionally share custom ingredients with community
- Pantry data is completely private to each user

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| ING_001 | 400 | Invalid ingredient data |
| ING_002 | 404 | Ingredient not found |
| ING_003 | 409 | Ingredient name already exists |
| ING_004 | 422 | Invalid nutritional information |
| ING_005 | 422 | Invalid unit conversion |
| ING_006 | 403 | Cannot modify public ingredient |
| PAN_001 | 400 | Invalid pantry item data |
| PAN_002 | 404 | Pantry item not found |
| PAN_003 | 422 | Pantry storage limit exceeded |

## Rate Limiting

- **Read operations**: 2000 requests per hour per user
- **Search operations**: 500 requests per hour per user
- **Create/Update operations**: 200 requests per hour per user
- **Pantry operations**: 1000 requests per hour per user

---

These ingredient management endpoints provide comprehensive ingredient database functionality with personalization features, substitution intelligence, and pantry management for the MealPrep application.

*Documentation updated for API version 1.0*