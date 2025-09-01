# MealMenu API Documentation

## Overview

The MealMenu API is a RESTful web service that provides comprehensive endpoints for managing family meal planning, recipes, AI-powered suggestions, and user management. The API follows REST principles and returns JSON responses.

## Base URL

- **Development**: `https://localhost:7001/api`
- **Production**: `https://api.mealmenu.com/api`

## Authentication

All API endpoints (except authentication endpoints) require a valid JWT Bearer token.

### Headers
```
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

## API Versioning

The API uses URL versioning:
- Current version: `v1`
- Example: `/api/v1/recipes`

## Response Format

### Success Response
```json
{
  "success": true,
  "data": {
    // Response data
  },
  "message": "Operation completed successfully"
}
```

### Error Response
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Error description",
    "details": {}
  }
}
```

## Core Endpoints

### Authentication Endpoints

#### POST /api/v1/auth/login
Login with email and password.

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "token": "jwt_token_here",
    "refreshToken": "refresh_token_here",
    "user": {
      "id": "user_id",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe"
    }
  }
}
```

#### POST /api/v1/auth/register
Register a new user account.

#### POST /api/v1/auth/refresh
Refresh an expired JWT token.

#### POST /api/v1/auth/logout
Logout and invalidate tokens.

### Family Management Endpoints

#### GET /api/v1/families/current
Get current user's family information.

#### POST /api/v1/families
Create a new family.

#### PUT /api/v1/families/{id}
Update family information.

#### POST /api/v1/families/{id}/members
Add a family member.

#### GET /api/v1/families/{id}/members
Get all family members.

#### PUT /api/v1/families/{familyId}/members/{memberId}
Update family member information.

#### DELETE /api/v1/families/{familyId}/members/{memberId}
Remove a family member.

### Recipe Management Endpoints

#### GET /api/v1/recipes
Get recipes with filtering and pagination.

**Query Parameters:**
- `page` (int): Page number (default: 1)
- `pageSize` (int): Items per page (default: 20)
- `category` (string): Filter by category
- `difficulty` (string): Filter by difficulty level
- `prepTime` (int): Maximum preparation time in minutes
- `dietary` (string): Dietary restrictions filter
- `search` (string): Search term for recipe name or ingredients

#### GET /api/v1/recipes/{id}
Get a specific recipe by ID.

#### POST /api/v1/recipes
Create a new recipe.

**Request Body:**
```json
{
  "name": "Chicken Stir Fry",
  "description": "Quick and healthy chicken stir fry",
  "prepTime": 15,
  "cookTime": 20,
  "servings": 4,
  "difficulty": "Easy",
  "category": "Main Course",
  "ingredients": [
    {
      "name": "Chicken breast",
      "amount": 1,
      "unit": "lb"
    }
  ],
  "instructions": [
    {
      "step": 1,
      "description": "Cut chicken into strips"
    }
  ],
  "nutritionalInfo": {
    "calories": 350,
    "protein": 25,
    "carbs": 15,
    "fat": 10
  },
  "tags": ["healthy", "quick", "protein"],
  "imageUrl": "recipe-image-url"
}
```

#### PUT /api/v1/recipes/{id}
Update an existing recipe.

#### DELETE /api/v1/recipes/{id}
Delete a recipe.

#### POST /api/v1/recipes/{id}/favorite
Add recipe to favorites.

#### DELETE /api/v1/recipes/{id}/favorite
Remove recipe from favorites.

### AI Meal Planning Endpoints

#### POST /api/v1/ai/meal-suggestions
Get AI-powered meal suggestions based on family preferences.

**Request Body:**
```json
{
  "familyId": "family_id",
  "preferences": {
    "dietaryRestrictions": ["vegetarian", "gluten-free"],
    "cuisinePreferences": ["italian", "asian"],
    "cookingTime": 30,
    "skillLevel": "beginner"
  },
  "plannedDays": 7,
  "mealsPerDay": 3
}
```

#### POST /api/v1/ai/recipe-recommendations
Get personalized recipe recommendations.

#### POST /api/v1/ai/ingredient-substitutions
Get AI suggestions for ingredient substitutions.

### Menu Planning Endpoints

#### GET /api/v1/menus
Get user's meal plans.

#### POST /api/v1/menus
Create a new meal plan.

#### GET /api/v1/menus/{id}
Get a specific meal plan.

#### PUT /api/v1/menus/{id}
Update a meal plan.

#### DELETE /api/v1/menus/{id}
Delete a meal plan.

#### POST /api/v1/menus/{id}/generate-shopping-list
Generate shopping list from meal plan.

### Shopping List Endpoints

#### GET /api/v1/shopping-lists
Get user's shopping lists.

#### POST /api/v1/shopping-lists
Create a new shopping list.

#### PUT /api/v1/shopping-lists/{id}
Update a shopping list.

#### DELETE /api/v1/shopping-lists/{id}
Delete a shopping list.

#### PUT /api/v1/shopping-lists/{id}/items/{itemId}
Update shopping list item status.

### Nutrition Endpoints

#### GET /api/v1/nutrition/daily/{date}
Get daily nutrition summary for a specific date.

#### GET /api/v1/nutrition/weekly/{startDate}
Get weekly nutrition summary.

#### POST /api/v1/nutrition/goals
Set nutrition goals for family members.

#### GET /api/v1/nutrition/goals/{memberId}
Get nutrition goals for a family member.

## Rate Limiting

API requests are rate-limited to prevent abuse:
- **Authenticated users**: 1000 requests per hour
- **AI endpoints**: 100 requests per hour
- **Authentication endpoints**: 10 requests per minute

Rate limit headers are included in responses:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```

## Error Codes

### Authentication Errors
- `AUTH_001`: Invalid credentials
- `AUTH_002`: Token expired
- `AUTH_003`: Invalid token format
- `AUTH_004`: Account locked
- `AUTH_005`: Email not verified

### Validation Errors
- `VAL_001`: Required field missing
- `VAL_002`: Invalid email format
- `VAL_003`: Password too weak
- `VAL_004`: Invalid data format

### Business Logic Errors
- `BUS_001`: Recipe not found
- `BUS_002`: Family member not found
- `BUS_003`: Insufficient permissions
- `BUS_004`: Recipe already favorited
- `BUS_005`: Menu plan conflict

### System Errors
- `SYS_001`: Internal server error
- `SYS_002`: Service unavailable
- `SYS_003`: Database connection error
- `SYS_004`: External service error

## Webhooks

MealMenu supports webhooks for real-time notifications:

### Supported Events
- `recipe.created`
- `menu.planned`
- `shopping_list.completed`
- `ai.suggestion.ready`

### Webhook Format
```json
{
  "event": "recipe.created",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "recipeId": "recipe_id",
    "userId": "user_id"
  }
}
```

## SDKs and Client Libraries

- **JavaScript/TypeScript**: `@mealmenu/api-client`
- **C#**: `MealMenu.ApiClient`
- **Python**: `mealmenu-api-client`

## Testing

Use the provided Postman collection for API testing:
- [Download Postman Collection](api/postman-collection.json)
- Environment variables included for easy setup
- Automated tests for critical endpoints

## Support

For API support and questions:
- **Documentation**: This guide and endpoint-specific docs
- **GitHub Issues**: Report bugs and request features
- **Developer Discord**: Real-time support and discussions