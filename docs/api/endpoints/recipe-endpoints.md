# Recipe Management Endpoints

## Overview
API endpoints for creating, reading, updating, and deleting recipes in the MealPrep application.

## Base URL
```
https://api.mealprep.com/v1/recipes
```

## Endpoints

### Get All Recipes

**Endpoint:** `GET /`

**Description:** Retrieve paginated list of recipes with optional filtering.

**Query Parameters:**
- `page` (integer, default: 1) - Page number
- `pageSize` (integer, default: 20, max: 100) - Items per page
- `cuisine` (string) - Filter by cuisine type
- `maxPrepTime` (integer) - Maximum preparation time in minutes
- `difficulty` (string) - Filter by difficulty (Easy, Medium, Hard)
- `search` (string) - Search in recipe names and descriptions

**Example Request:**
```
GET /api/v1/recipes?page=1&pageSize=10&cuisine=italian&maxPrepTime=30
```

**Success Response (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Chicken Parmesan",
      "description": "Classic Italian chicken dish",
      "prepTimeMinutes": 20,
      "cookTimeMinutes": 25,
      "servings": 4,
      "difficulty": "Medium",
      "cuisine": "Italian",
      "estimatedCost": 18.50,
      "familyFitScore": 8,
      "imageUrl": "https://images.mealprep.com/recipes/chicken-parmesan.jpg",
      "createdAt": "2024-01-01T00:00:00Z",
      "author": {
        "id": "user123",
        "name": "John Doe"
      }
    }
  ],
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "pagination": {
      "page": 1,
      "pageSize": 10,
      "totalCount": 150,
      "totalPages": 15,
      "hasNext": true,
      "hasPrevious": false
    }
  }
}
```

### Get Recipe by ID

**Endpoint:** `GET /{id}`

**Description:** Retrieve detailed information for a specific recipe.

**Path Parameters:**
- `id` (integer) - Recipe ID

**Success Response (200 OK):**
```json
{
  "data": {
    "id": 1,
    "name": "Chicken Parmesan",
    "description": "Classic Italian chicken dish with crispy coating",
    "instructions": [
      {
        "step": 1,
        "instruction": "Pound chicken to 1/4 inch thickness",
        "timeEstimate": 5
      },
      {
        "step": 2,
        "instruction": "Set up breading station with flour, eggs, and breadcrumbs",
        "timeEstimate": 3
      }
    ],
    "ingredients": [
      {
        "id": 1,
        "name": "Chicken breast",
        "quantity": 4,
        "unit": "pieces",
        "isOptional": false,
        "substitutions": ["chicken thighs"]
      },
      {
        "id": 2,
        "name": "Breadcrumbs",
        "quantity": 1,
        "unit": "cup",
        "isOptional": false
      }
    ],
    "nutritionalInfo": {
      "calories": 350,
      "protein": 25,
      "carbohydrates": 20,
      "fat": 15,
      "fiber": 3,
      "sodium": 650
    },
    "prepTimeMinutes": 20,
    "cookTimeMinutes": 25,
    "totalTimeMinutes": 45,
    "servings": 4,
    "difficulty": "Medium",
    "cuisine": "Italian",
    "estimatedCost": 18.50,
    "familyFitScore": 8,
    "tags": ["dinner", "family-friendly", "italian"],
    "imageUrl": "https://images.mealprep.com/recipes/chicken-parmesan.jpg",
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-02T00:00:00Z",
    "author": {
      "id": "user123",
      "name": "John Doe",
      "avatarUrl": "https://images.mealprep.com/users/user123.jpg"
    }
  }
}
```

### Create Recipe

**Endpoint:** `POST /`

**Description:** Create a new recipe.

**Headers:**
```
Authorization: Bearer [JWT_TOKEN]
Content-Type: application/json
```

**Request Body:**
```json
{
  "name": "Chicken Parmesan",
  "description": "Classic Italian chicken dish",
  "instructions": [
    {
      "step": 1,
      "instruction": "Pound chicken to 1/4 inch thickness",
      "timeEstimate": 5
    }
  ],
  "ingredients": [
    {
      "name": "Chicken breast",
      "quantity": 4,
      "unit": "pieces",
      "isOptional": false
    }
  ],
  "prepTimeMinutes": 20,
  "cookTimeMinutes": 25,
  "servings": 4,
  "difficulty": "Medium",
  "cuisine": "Italian",
  "tags": ["dinner", "family-friendly"],
  "imageUrl": "https://images.mealprep.com/uploads/temp/abc123.jpg"
}
```

**Success Response (201 Created):**
```json
{
  "data": {
    "id": 1,
    "name": "Chicken Parmesan",
    "description": "Classic Italian chicken dish",
    "prepTimeMinutes": 20,
    "cookTimeMinutes": 25,
    "servings": 4,
    "difficulty": "Medium",
    "cuisine": "Italian",
    "createdAt": "2024-01-01T00:00:00Z",
    "author": {
      "id": "user123",
      "name": "John Doe"
    }
  }
}
```

### Update Recipe

**Endpoint:** `PUT /{id}`

**Description:** Update an existing recipe (full update).

**Path Parameters:**
- `id` (integer) - Recipe ID

**Headers:**
```
Authorization: Bearer [JWT_TOKEN]
Content-Type: application/json
```

**Request Body:** Same as Create Recipe

**Success Response (200 OK):** Same format as Get Recipe by ID

### Partial Update Recipe

**Endpoint:** `PATCH /{id}`

**Description:** Partially update a recipe.

**Request Body:**
```json
{
  "name": "Updated Chicken Parmesan",
  "prepTimeMinutes": 15,
  "tags": ["dinner", "quick-meal", "family-friendly"]
}
```

### Delete Recipe

**Endpoint:** `DELETE /{id}`

**Description:** Delete a recipe (soft delete).

**Success Response (204 No Content)**

### Search Recipes

**Endpoint:** `GET /search`

**Description:** Full-text search across recipes.

**Query Parameters:**
- `q` (string, required) - Search query
- `page` (integer, default: 1)
- `pageSize` (integer, default: 20)

**Example:**
```
GET /api/v1/recipes/search?q=chicken+pasta&page=1&pageSize=10
```

### Get Recipe Nutrition

**Endpoint:** `GET /{id}/nutrition`

**Description:** Get detailed nutritional information for a recipe.

**Success Response (200 OK):**
```json
{
  "data": {
    "perServing": {
      "calories": 350,
      "protein": 25,
      "carbohydrates": 20,
      "fat": 15,
      "fiber": 3,
      "sodium": 650,
      "sugar": 8,
      "cholesterol": 75
    },
    "totalRecipe": {
      "calories": 1400,
      "protein": 100,
      "carbohydrates": 80,
      "fat": 60
    },
    "macroBreakdown": {
      "proteinPercent": 29,
      "carbPercent": 23,
      "fatPercent": 48
    },
    "dietaryInfo": {
      "isVegetarian": false,
      "isVegan": false,
      "isGlutenFree": false,
      "isDairyFree": false,
      "isKeto": false
    }
  }
}
```

### Scale Recipe

**Endpoint:** `POST /{id}/scale`

**Description:** Scale recipe ingredients for different serving sizes.

**Request Body:**
```json
{
  "targetServings": 6
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "originalServings": 4,
    "targetServings": 6,
    "scaleFactor": 1.5,
    "scaledIngredients": [
      {
        "name": "Chicken breast",
        "originalQuantity": 4,
        "scaledQuantity": 6,
        "unit": "pieces"
      }
    ],
    "adjustedTimes": {
      "prepTimeMinutes": 25,
      "cookTimeMinutes": 30
    }
  }
}
```

### Fork Recipe

**Endpoint:** `POST /{id}/fork`

**Description:** Create a copy of a recipe for modification.

**Success Response (201 Created):** Returns new recipe with incremented name

### Rate Recipe

**Endpoint:** `POST /{id}/rating`

**Description:** Add or update user rating for a recipe.

**Request Body:**
```json
{
  "rating": 5,
  "review": "Delicious and easy to make!"
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "userRating": 5,
    "userReview": "Delicious and easy to make!",
    "averageRating": 4.3,
    "totalRatings": 127
  }
}
```

### Get Recipe Comments

**Endpoint:** `GET /{id}/comments`

**Description:** Get comments and reviews for a recipe.

**Success Response (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "user": {
        "id": "user456",
        "name": "Jane Smith",
        "avatarUrl": "https://images.mealprep.com/users/user456.jpg"
      },
      "rating": 5,
      "comment": "Amazing recipe! My family loved it.",
      "createdAt": "2024-01-01T00:00:00Z"
    }
  ]
}
```

### Add Recipe Comment

**Endpoint:** `POST /{id}/comments`

**Description:** Add a comment to a recipe.

**Request Body:**
```json
{
  "comment": "Great recipe! Added extra garlic.",
  "rating": 4
}
```

## Validation Rules

### Recipe Validation
```csharp
public class CreateRecipeRequest
{
    [Required]
    [StringLength(200, MinimumLength = 3)]
    public string Name { get; set; }

    [StringLength(1000)]
    public string Description { get; set; }

    [Required]
    [MinLength(1)]
    public List<RecipeInstruction> Instructions { get; set; }

    [Required]
    [MinLength(1)]
    public List<RecipeIngredient> Ingredients { get; set; }

    [Range(1, 1440)]
    public int PrepTimeMinutes { get; set; }

    [Range(1, 1440)]
    public int CookTimeMinutes { get; set; }

    [Range(1, 20)]
    public int Servings { get; set; }

    [RegularExpression("^(Easy|Medium|Hard)$")]
    public string Difficulty { get; set; }

    [StringLength(100)]
    public string Cuisine { get; set; }

    [Url]
    public string ImageUrl { get; set; }
}
```

## Authorization

### Recipe Permissions
- **Create**: Authenticated users only
- **Read**: Public recipes visible to all, private recipes only to owner
- **Update**: Recipe owner only
- **Delete**: Recipe owner only
- **Rate/Comment**: Authenticated users only

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| RECIPE_001 | 400 | Invalid recipe data |
| RECIPE_002 | 404 | Recipe not found |
| RECIPE_003 | 403 | Not authorized to modify recipe |
| RECIPE_004 | 409 | Recipe name already exists for user |
| RECIPE_005 | 413 | Recipe image too large |

## Rate Limiting

- **Read operations**: 1000 requests per hour per user
- **Create/Update**: 50 requests per hour per user
- **Search**: 100 requests per hour per user

---

These recipe endpoints provide comprehensive recipe management functionality for the MealPrep application with proper validation, authorization, and error handling.

*Documentation updated for API version 1.0*