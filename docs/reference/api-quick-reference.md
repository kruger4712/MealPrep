# API Quick Reference

## Overview
Quick reference guide for all MealPrep API endpoints. For detailed documentation, see the [complete API documentation](../API.md).

## Base URLs
- **Development**: `https://localhost:7001/api`
- **Staging**: `https://staging-api.mealprep.com/api`
- **Production**: `https://api.mealprep.com/api`

## Authentication
All API requests require JWT authentication (except registration and login).

```http
Authorization: Bearer {your_jwt_token}
```

## Response Format
```json
{
  "success": true,
  "message": "Operation completed successfully",
  "data": { /* response data */ }
}
```

---

## Authentication Endpoints

### Register User
```http
POST /auth/register
Content-Type: application/json

{
  "firstName": "John",
  "lastName": "Doe", 
  "email": "john@example.com",
  "password": "SecurePassword123!",
  "confirmPassword": "SecurePassword123!"
}
```

### Login User
```http
POST /auth/login
Content-Type: application/json

{
  "email": "john@example.com",
  "password": "SecurePassword123!"
}
```

### Refresh Token
```http
POST /auth/refresh
Content-Type: application/json

{
  "refreshToken": "{refresh_token}"
}
```

### Logout
```http
POST /auth/logout
Authorization: Bearer {token}
```

---

## Recipe Endpoints

### Get Recipes (Paginated)
```http
GET /recipes?page=1&pageSize=12&search=chicken&cuisine=Italian&difficulty=Easy
Authorization: Bearer {token}
```

**Query Parameters:**
- `page` (int): Page number (default: 1)
- `pageSize` (int): Items per page (max: 50, default: 12)
- `search` (string): Search term for name/description
- `cuisine` (string): Filter by cuisine type
- `difficulty` (string): Easy | Medium | Hard
- `isAiGenerated` (bool): Filter AI-generated recipes
- `maxPrepTime` (int): Maximum prep time in minutes

### Get Recipe by ID
```http
GET /recipes/{id}
Authorization: Bearer {token}
```

### Create Recipe
```http
POST /recipes
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Grilled Salmon",
  "description": "Healthy grilled salmon with herbs",
  "instructions": "1. Preheat grill...\n2. Season salmon...",
  "prepTimeMinutes": 10,
  "cookTimeMinutes": 15,
  "servings": 4,
  "difficulty": "Easy",
  "cuisine": "American",
  "estimatedCost": 25.00,
  "imageUrl": "https://example.com/salmon.jpg",
  "tags": ["healthy", "seafood", "grilled"],
  "ingredients": [
    {
      "ingredientId": 3,
      "quantity": 2,
      "unit": "lb",
      "notes": "salmon fillets",
      "isOptional": false
    }
  ]
}
```

### Update Recipe
```http
PUT /recipes/{id}
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Updated Recipe Name",
  "description": "Updated description",
  // ... other fields
}
```

### Delete Recipe
```http
DELETE /recipes/{id}
Authorization: Bearer {token}
```

### Rate Recipe
```http
POST /recipes/{id}/rate
Authorization: Bearer {token}
Content-Type: application/json

{
  "rating": 5,
  "comment": "Excellent recipe!"
}
```

### Favorite Recipe
```http
POST /recipes/{id}/favorite
Authorization: Bearer {token}
```

### Unfavorite Recipe
```http
DELETE /recipes/{id}/favorite
Authorization: Bearer {token}
```

---

## Family Member Endpoints

### Get Family Members
```http
GET /family
Authorization: Bearer {token}
```

### Get Family Member by ID
```http
GET /family/{id}
Authorization: Bearer {token}
```

### Create Family Member
```http
POST /family
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Sarah Doe",
  "age": 28,
  "profileImage": "https://example.com/profiles/sarah.jpg",
  "likedIngredients": ["vegetables", "pasta", "cheese"],
  "dislikedIngredients": ["seafood"],
  "allergies": ["shellfish"],
  "dietaryRestrictions": ["vegetarian"],
  "cuisinePreferences": "Mediterranean, Asian",
  "spiceToleranceLevel": 4,
  "cookingSkillLevel": "Beginner",
  "preferredCookingTime": 30,
  "kitchenAppliances": ["oven", "microwave"],
  "healthGoals": "Increased vegetable intake"
}
```

### Update Family Member
```http
PUT /family/{id}
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Updated Name",
  "age": 29,
  // ... other fields
}
```

### Delete Family Member
```http
DELETE /family/{id}
Authorization: Bearer {token}
```

### Update Family Member Preferences
```http
PUT /family/{id}/preferences
Authorization: Bearer {token}
Content-Type: application/json

{
  "likedIngredients": ["chicken", "rice", "vegetables"],
  "dislikedIngredients": ["mushrooms"],
  "allergies": ["nuts"],
  "dietaryRestrictions": ["low-sodium"],
  "spiceToleranceLevel": 6
}
```

---

## AI Suggestion Endpoints

### Get Meal Suggestions
```http
POST /ai-suggestions/meal-suggestions
Authorization: Bearer {token}
Content-Type: application/json

{
  "familyMembers": [
    {
      "name": "John",
      "age": 35,
      "likedIngredients": ["chicken", "rice", "vegetables"],
      "dislikedIngredients": ["mushrooms"],
      "allergies": ["nuts"],
      "dietaryRestrictions": [],
      "cuisinePreferences": "Italian, American",
      "spiceToleranceLevel": 6,
      "cookingSkillLevel": "Intermediate",
      "preferredCookingTime": 45,
      "kitchenAppliances": ["oven", "grill"],
      "healthGoals": "Weight maintenance"
    }
  ],
  "mealType": "Dinner",
  "targetDate": "2024-01-15T18:00:00Z",
  "availableIngredients": ["chicken", "rice", "broccoli"],
  "budget": 25.00,
  "occasion": "Regular",
  "prepTimeLimit": 60,
  "recentMeals": ["spaghetti", "tacos"]
}
```

### Generate Weekly Menu
```http
POST /ai-suggestions/weekly-menu
Authorization: Bearer {token}
Content-Type: application/json

{
  "familyMembers": [/* family member objects */],
  "weekStartDate": "2024-01-15T00:00:00Z",
  "weeklyBudget": 150.00,
  "mealTypes": ["Breakfast", "Lunch", "Dinner"],
  "specialOccasions": ["Birthday dinner on Friday"],
  "preferredCookingTime": 45,
  "cuisineVariety": ["Italian", "Mexican", "American", "Asian"],
  "availableIngredients": ["chicken", "rice", "vegetables"]
}
```

### Personalize Recipe
```http
POST /ai-suggestions/personalize-recipe
Authorization: Bearer {token}
Content-Type: application/json

{
  "recipeId": 123,
  "familyMembers": [/* family member objects */],
  "customizations": {
    "dietaryAdaptations": ["vegetarian", "gluten-free"],
    "spiceLevelAdjustment": "mild",
    "portionAdjustment": 6,
    "timeConstraints": 30,
    "availableIngredients": ["specific", "ingredients"],
    "unwantedIngredients": ["ingredients", "to", "avoid"]
  }
}
```

### Get Ingredient Substitutions
```http
POST /ai-suggestions/ingredient-substitutions
Authorization: Bearer {token}
Content-Type: application/json

{
  "ingredient": "milk",
  "dietaryRestrictions": ["lactose-free", "vegan"],
  "recipeContext": "baking",
  "quantity": "1 cup"
}
```

---

## Menu Planning Endpoints

### Get Weekly Menus
```http
GET /menus?startDate=2024-01-15&endDate=2024-01-21
Authorization: Bearer {token}
```

### Get Menu by ID
```http
GET /menus/{id}
Authorization: Bearer {token}
```

### Create Weekly Menu
```http
POST /menus
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Week of January 15th",
  "startDate": "2024-01-15T00:00:00Z",
  "endDate": "2024-01-21T23:59:59Z",
  "meals": [
    {
      "dayOfWeek": "Monday",
      "mealType": "Breakfast",
      "recipeId": 123,
      "notes": "Prep night before",
      "servings": 4
    },
    {
      "dayOfWeek": "Monday", 
      "mealType": "Dinner",
      "recipeId": 456,
      "notes": "Use leftover vegetables",
      "servings": 4
    }
  ]
}
```

### Update Menu
```http
PUT /menus/{id}
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Updated Menu Name",
  "meals": [/* updated meals array */]
}
```

### Delete Menu
```http
DELETE /menus/{id}
Authorization: Bearer {token}
```

### Generate Shopping List from Menu
```http
POST /menus/{id}/shopping-list
Authorization: Bearer {token}
Content-Type: application/json

{
  "consolidateItems": true,
  "organizeByCategory": true,
  "excludeOwnedItems": ["rice", "salt", "olive oil"]
}
```

---

## Ingredient Endpoints

### Get Ingredients
```http
GET /ingredients?search=chicken&category=protein&page=1&pageSize=20
Authorization: Bearer {token}
```

**Query Parameters:**
- `search` (string): Search term for ingredient name
- `category` (string): Filter by category (protein, vegetable, dairy, etc.)
- `page` (int): Page number (default: 1)
- `pageSize` (int): Items per page (max: 50, default: 20)

### Get Ingredient by ID
```http
GET /ingredients/{id}
Authorization: Bearer {token}
```

### Create Ingredient
```http
POST /ingredients
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Organic Chicken Breast",
  "category": "Protein",
  "commonUnits": ["lb", "kg", "piece"],
  "averageCostPerUnit": 8.99,
  "nutritionalInfo": {
    "caloriesPerUnit": 165,
    "proteinGrams": 31,
    "carbGrams": 0,
    "fatGrams": 3.6,
    "fiberGrams": 0,
    "sodiumMilligrams": 74
  },
  "allergens": [],
  "dietaryTags": ["high-protein", "low-carb"]
}
```

### Update Ingredient
```http
PUT /ingredients/{id}
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Updated Ingredient Name",
  "averageCostPerUnit": 9.99,
  // ... other fields
}
```

### Delete Ingredient
```http
DELETE /ingredients/{id}
Authorization: Bearer {token}
```

### Get Ingredient Nutrition
```http
GET /ingredients/{id}/nutrition
Authorization: Bearer {token}
```

---

## Shopping List Endpoints

### Get Shopping Lists
```http
GET /shopping-lists
Authorization: Bearer {token}
```

### Get Shopping List by ID
```http
GET /shopping-lists/{id}
Authorization: Bearer {token}
```

### Create Shopping List
```http
POST /shopping-lists
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Weekly Groceries - Jan 15",
  "items": [
    {
      "ingredientId": 123,
      "quantity": 2,
      "unit": "lb",
      "notes": "organic preferred",
      "category": "Protein",
      "estimatedCost": 16.99,
      "isPriority": true,
      "isOptional": false
    }
  ]
}
```

### Update Shopping List
```http
PUT /shopping-lists/{id}
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Updated List Name",
  "items": [/* updated items array */]
}
```

### Add Item to Shopping List
```http
POST /shopping-lists/{id}/items
Authorization: Bearer {token}
Content-Type: application/json

{
  "ingredientId": 456,
  "quantity": 1,
  "unit": "package",
  "notes": "whole wheat",
  "category": "Pantry"
}
```

### Update Shopping List Item
```http
PUT /shopping-lists/{listId}/items/{itemId}
Authorization: Bearer {token}
Content-Type: application/json

{
  "quantity": 3,
  "isPurchased": true,
  "actualCost": 12.50
}
```

### Delete Shopping List Item
```http
DELETE /shopping-lists/{listId}/items/{itemId}
Authorization: Bearer {token}
```

### Mark Item as Purchased
```http
PUT /shopping-lists/{listId}/items/{itemId}/purchased
Authorization: Bearer {token}
Content-Type: application/json

{
  "isPurchased": true,
  "actualCost": 12.50,
  "purchaseDate": "2024-01-15T14:30:00Z"
}
```

---

## User Profile Endpoints

### Get User Profile
```http
GET /profile
Authorization: Bearer {token}
```

### Update User Profile
```http
PUT /profile
Authorization: Bearer {token}
Content-Type: application/json

{
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@example.com",
  "profileImage": "https://example.com/profiles/john.jpg",
  "preferences": {
    "units": "imperial",
    "theme": "light",
    "language": "en-US",
    "notifications": {
      "email": true,
      "push": true,
      "sms": false
    }
  }
}
```

### Change Password
```http
POST /profile/change-password
Authorization: Bearer {token}
Content-Type: application/json

{
  "currentPassword": "OldPassword123!",
  "newPassword": "NewPassword456!",
  "confirmPassword": "NewPassword456!"
}
```

### Delete Account
```http
DELETE /profile
Authorization: Bearer {token}
Content-Type: application/json

{
  "password": "CurrentPassword123!",
  "confirmDeletion": "DELETE"
}
```

---

## Status Codes

| Code | Status | Description |
|------|--------|-------------|
| 200 | OK | Request successful |
| 201 | Created | Resource created successfully |
| 204 | No Content | Request successful, no content returned |
| 400 | Bad Request | Invalid request data |
| 401 | Unauthorized | Authentication required |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource not found |
| 409 | Conflict | Resource conflict (e.g., duplicate email) |
| 422 | Unprocessable Entity | Validation failed |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |

## Rate Limiting

- **General API**: 100 requests per minute per user
- **AI Endpoints**: 20 requests per minute per user
- **Authentication**: 10 requests per minute per IP
- **Image Upload**: 5 requests per minute per user

Rate limit headers:
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642678800
```

## Error Response Format

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    {
      "field": "email",
      "message": "Invalid email format"
    },
    {
      "field": "password", 
      "message": "Password must be at least 8 characters"
    }
  ],
  "errorCode": "VALIDATION_ERROR",
  "timestamp": "2024-01-15T10:30:00Z",
  "requestId": "12345-67890-abcdef"
}
```

## Common Query Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `page` | int | Page number (1-based) | `?page=2` |
| `pageSize` | int | Items per page (max 50) | `?pageSize=20` |
| `search` | string | Search term | `?search=chicken` |
| `sortBy` | string | Sort field | `?sortBy=name` |
| `sortOrder` | string | asc \| desc | `?sortOrder=desc` |
| `filter` | string | Filter criteria | `?filter=category:protein` |

## Request Headers

### Required for Authenticated Requests
```http
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

### Optional Headers
```http
X-Request-ID: unique-request-identifier
X-Client-Version: 1.0.0
Accept-Language: en-US,en;q=0.9
```

## WebSocket Endpoints

### Real-time Notifications
```javascript
const ws = new WebSocket('wss://api.mealprep.com/ws/notifications');
ws.send(JSON.stringify({
  type: 'authenticate',
  token: 'your_jwt_token'
}));
```

### Menu Planning Collaboration
```javascript
const ws = new WebSocket('wss://api.mealprep.com/ws/menu-planning');
ws.send(JSON.stringify({
  type: 'join_session',
  menuId: 123,
  token: 'your_jwt_token'
}));
```

---

## Quick Examples

### Complete Meal Planning Flow
```bash
# 1. Get family members
curl -H "Authorization: Bearer {token}" \
  https://api.mealprep.com/api/family

# 2. Generate meal suggestions
curl -X POST -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"familyMembers": [...], "mealType": "Dinner"}' \
  https://api.mealprep.com/api/ai-suggestions/meal-suggestions

# 3. Create weekly menu
curl -X POST -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"name": "Week 1", "meals": [...]}' \
  https://api.mealprep.com/api/menus

# 4. Generate shopping list
curl -X POST -H "Authorization: Bearer {token}" \
  https://api.mealprep.com/api/menus/123/shopping-list
```

### Recipe Management Flow
```bash
# 1. Search recipes
curl -H "Authorization: Bearer {token}" \
  "https://api.mealprep.com/api/recipes?search=chicken&cuisine=Italian"

# 2. Get recipe details
curl -H "Authorization: Bearer {token}" \
  https://api.mealprep.com/api/recipes/123

# 3. Rate recipe
curl -X POST -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"rating": 5, "comment": "Excellent!"}' \
  https://api.mealprep.com/api/recipes/123/rate

# 4. Add to favorites
curl -X POST -H "Authorization: Bearer {token}" \
  https://api.mealprep.com/api/recipes/123/favorite
```

---

## SDK Examples

### JavaScript/TypeScript
```typescript
import { MealPrepClient } from '@mealprep/api-client';

const client = new MealPrepClient({
  baseUrl: 'https://api.mealprep.com/api',
  apiKey: 'your_api_key'
});

// Get meal suggestions
const suggestions = await client.ai.getMealSuggestions({
  familyMembers: familyData,
  mealType: 'Dinner',
  budget: 25.00
});

// Create recipe
const recipe = await client.recipes.create({
  name: 'Grilled Salmon',
  prepTimeMinutes: 10,
  // ... other fields
});
```

### C# .NET
```csharp
using MealPrep.ApiClient;

var client = new MealPrepApiClient("https://api.mealprep.com/api");
await client.AuthenticateAsync("your_jwt_token");

// Get family members
var family = await client.Family.GetAllAsync();

// Generate meal suggestions
var suggestions = await client.AiSuggestions.GetMealSuggestionsAsync(new MealSuggestionRequest
{
    FamilyMembers = family,
    MealType = "Dinner",
    Budget = 25.00m
});
```

### Python
```python
from mealprep_client import MealPrepClient

client = MealPrepClient(
    base_url='https://api.mealprep.com/api',
    api_key='your_api_key'
)

# Get recipes
recipes = client.recipes.search(
    search='chicken',
    cuisine='Italian',
    page=1,
    page_size=20
)

# Create shopping list
shopping_list = client.shopping_lists.create({
    'name': 'Weekly Groceries',
    'items': [...]
})
```

---

*Last Updated: December 2024*  
*For detailed documentation and examples, see the [complete API documentation](../API.md)*