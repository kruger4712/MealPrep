# AI Suggestion Endpoints

## Overview
API endpoints for AI-powered meal suggestions, recipe personalization, and menu planning using Google Gemini AI integration.

## Base URL
```
https://api.mealprep.com/v1/ai
```

## Endpoints

### Generate Meal Suggestions

**Endpoint:** `POST /meal-suggestions`

**Description:** Generate AI-powered meal suggestions based on family preferences and constraints.

**Headers:**
```
Authorization: Bearer [JWT_TOKEN]
Content-Type: application/json
```

**Request Body:**
```json
{
  "familyMembers": [1, 2, 3],
  "mealType": "dinner",
  "targetDate": "2024-01-25",
  "budget": 25.00,
  "maxPrepTime": 30,
  "maxCookTime": 45,
  "occasion": "weeknight",
  "availableIngredients": [
    {
      "name": "chicken breast",
      "quantity": 2,
      "unit": "lbs",
      "expirationDate": "2024-01-27"
    },
    {
      "name": "pasta",
      "quantity": 1,
      "unit": "box"
    }
  ],
  "recentMeals": [
    "Spaghetti Carbonara",
    "Chicken Stir Fry", 
    "Caesar Salad"
  ],
  "preferences": {
    "cuisineTypes": ["Italian", "American"],
    "avoidIngredients": ["mushrooms", "seafood"],
    "prioritizeHealthy": true,
    "vegetarianOptions": true
  },
  "nutritionalGoals": {
    "maxCalories": 600,
    "minProtein": 25,
    "maxSodium": 800
  }
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "requestId": "ai_req_12345",
    "suggestions": [
      {
        "id": "suggestion_1",
        "name": "Herb Crusted Chicken with Garlic Pasta",
        "description": "Tender chicken breast with herb crust served over garlic-infused pasta",
        "familyFitScore": 9,
        "reasoning": "High family fit due to loved ingredients (chicken, pasta, garlic) and accommodates vegetarian alternative",
        "prepTime": 20,
        "cookTime": 25,
        "totalTime": 45,
        "difficulty": "Medium",
        "estimatedCost": 18.50,
        "servings": 4,
        "cuisine": "Italian-American",
        "ingredients": [
          {
            "name": "Chicken breast",
            "amount": 1.5,
            "unit": "lbs",
            "cost": 8.00,
            "isAvailable": true,
            "substitutions": ["Tofu for vegetarians"]
          },
          {
            "name": "Pasta",
            "amount": 12,
            "unit": "oz",
            "cost": 2.00,
            "isAvailable": true
          },
          {
            "name": "Fresh herbs",
            "amount": 0.25,
            "unit": "cup",
            "cost": 3.00,
            "isAvailable": false
          }
        ],
        "instructions": [
          {
            "step": 1,
            "instruction": "Preheat oven to 375°F and prepare herb crust mixture",
            "timeEstimate": 5,
            "difficulty": "Easy"
          },
          {
            "step": 2, 
            "instruction": "Season and crust chicken breasts, then bake for 20 minutes",
            "timeEstimate": 25,
            "difficulty": "Medium"
          }
        ],
        "nutritionalInfo": {
          "calories": 485,
          "protein": 35,
          "carbohydrates": 45,
          "fat": 18,
          "fiber": 3,
          "sodium": 650,
          "sugar": 4
        },
        "tags": ["family-friendly", "protein-rich", "uses-available-ingredients"],
        "allergenWarnings": ["Contains gluten"],
        "dietaryCompatibility": [
          "Can be made vegetarian with tofu substitution",
          "Gluten-free with pasta substitution"
        ],
        "aiConfidence": 0.92,
        "personalizations": {
          "forJohn": "Perfect match for preferences",
          "forJane": "Vegetarian version with tofu",
          "forEmma": "Mild herbs, smaller portion"
        }
      },
      {
        "id": "suggestion_2",
        "name": "One-Pan Chicken and Vegetable Skillet",
        "description": "Complete meal with chicken, seasonal vegetables, and herbs",
        "familyFitScore": 8,
        "reasoning": "Uses available chicken, quick cooking method, kid-friendly vegetables",
        "prepTime": 15,
        "cookTime": 20,
        "totalTime": 35,
        "difficulty": "Easy",
        "estimatedCost": 16.75,
        "servings": 4,
        "cuisine": "American",
        "ingredients": [
          {
            "name": "Chicken breast",
            "amount": 1,
            "unit": "lb", 
            "isAvailable": true
          }
        ],
        "nutritionalInfo": {
          "calories": 420,
          "protein": 28,
          "carbohydrates": 25,
          "fat": 22
        },
        "aiConfidence": 0.87
      },
      {
        "id": "suggestion_3",
        "name": "Creamy Chicken Alfredo Pasta",
        "description": "Rich and creamy pasta dish with tender chicken pieces",
        "familyFitScore": 7,
        "reasoning": "Uses both available ingredients, popular with families, but higher in calories",
        "prepTime": 10,
        "cookTime": 20,
        "totalTime": 30,
        "difficulty": "Easy",
        "estimatedCost": 14.25,
        "aiConfidence": 0.83
      }
    ],
    "metadata": {
      "totalSuggestions": 3,
      "processingTime": "4.2s",
      "aiModel": "gemini-pro",
      "tokensUsed": 1247,
      "cost": 0.0062,
      "promptVersion": "v2.1"
    }
  },
  "metadata": {
    "timestamp": "2024-01-20T00:00:00Z",
    "requestId": "req_12345"
  }
}
```

### Personalize Recipe

**Endpoint:** `POST /personalize-recipe`

**Description:** Personalize an existing recipe for specific family members.

**Request Body:**
```json
{
  "recipeId": 123,
  "familyMembers": [1, 2],
  "personalizations": {
    "adjustServings": 6,
    "dietaryModifications": ["vegetarian", "low-sodium"],
    "spiceLevel": "mild",
    "substituteIngredients": {
      "chicken": "tofu",
      "heavy cream": "coconut milk"
    },
    "allergenAvoidance": ["nuts", "dairy"]
  },
  "preferences": {
    "reducePrep": true,
    "healthierVersion": true,
    "kidFriendly": true
  }
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "originalRecipe": {
      "id": 123,
      "name": "Chicken Alfredo",
      "servings": 4
    },
    "personalizedRecipe": {
      "name": "Mild Tofu Alfredo (Family Version)",
      "description": "Family-friendly version with tofu and dairy-free cream sauce",
      "servings": 6,
      "modifications": [
        "Substituted tofu for chicken",
        "Used coconut milk instead of heavy cream",
        "Reduced sodium by 40%",
        "Mild seasonings for children"
      ],
      "ingredients": [
        {
          "original": "1 lb chicken breast",
          "modified": "14 oz extra-firm tofu",
          "reason": "Vegetarian substitution"
        },
        {
          "original": "1 cup heavy cream",
          "modified": "1 cup coconut milk",
          "reason": "Dairy-free and lower saturated fat"
        }
      ],
      "instructions": [
        {
          "step": 1,
          "instruction": "Press tofu to remove moisture, then cube",
          "modification": "New step for tofu preparation"
        }
      ],
      "nutritionalChanges": {
        "calories": {
          "original": 520,
          "modified": 380,
          "change": -140
        },
        "protein": {
          "original": 35,
          "modified": 18,
          "change": -17
        },
        "sodium": {
          "original": 950,
          "modified": 570,
          "change": -380
        }
      },
      "familyNotes": {
        "forJane": "Perfect vegetarian version with familiar flavors",
        "forEmma": "Mild and creamy, no overwhelming spices"
      }
    },
    "confidence": 0.89,
    "processingTime": "3.1s"
  }
}
```

### Generate Weekly Menu

**Endpoint:** `POST /weekly-menu`

**Description:** Generate a complete weekly meal plan with AI suggestions.

**Request Body:**
```json
{
  "familyMembers": [1, 2, 3],
  "weekStartDate": "2024-01-22",
  "mealTypes": ["breakfast", "lunch", "dinner"],
  "weeklyBudget": 150.00,
  "preferences": {
    "varietyImportance": 8,
    "healthImportance": 7,
    "convenienceImportance": 6,
    "maxPrepTimePerMeal": 45,
    "allowLeftovers": true,
    "shoppingFrequency": "once"
  },
  "constraints": {
    "busyDays": ["Monday", "Wednesday"],
    "specialOccasions": [
      {
        "date": "2024-01-24",
        "occasion": "Date Night",
        "mealType": "dinner",
        "budget": 40.00
      }
    ],
    "groceryStores": ["Whole Foods", "Kroger"],
    "cookingSchedule": {
      "mealPrepDay": "Sunday",
      "maxCookingTime": 3
    }
  }
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "weeklyMenu": {
      "weekOf": "2024-01-22",
      "totalEstimatedCost": 142.75,
      "totalPrepTime": 380,
      "mealsPlanned": 21,
      "dailyMenus": [
        {
          "date": "2024-01-22",
          "dayOfWeek": "Monday",
          "meals": {
            "breakfast": {
              "recipeName": "Quick Overnight Oats",
              "prepTime": 5,
              "cost": 3.25,
              "familyFitScore": 8
            },
            "lunch": {
              "recipeName": "Turkey and Avocado Wrap",
              "prepTime": 10,
              "cost": 6.50,
              "familyFitScore": 7
            },
            "dinner": {
              "recipeName": "Sheet Pan Chicken and Vegetables",
              "prepTime": 15,
              "cookTime": 25,
              "cost": 14.25,
              "familyFitScore": 9,
              "notes": "Busy day - easy cleanup"
            }
          },
          "dailyCost": 24.00,
          "dailyPrepTime": 30
        }
      ],
      "shoppingList": {
        "byStore": {
          "Whole Foods": [
            {
              "item": "Organic chicken breast",
              "quantity": "3 lbs",
              "estimatedCost": 18.00
            }
          ],
          "Kroger": [
            {
              "item": "Pasta",
              "quantity": "2 boxes", 
              "estimatedCost": 4.00
            }
          ]
        },
        "byCategory": {
          "Proteins": [
            {
              "item": "Chicken breast",
              "totalQuantity": "3 lbs",
              "estimatedCost": 18.00
            }
          ]
        },
        "totalEstimatedCost": 138.50
      },
      "mealPrepPlan": {
        "sunday": [
          {
            "task": "Prep overnight oats for the week",
            "timeEstimate": 20,
            "difficulty": "Easy"
          },
          {
            "task": "Marinate chicken for Monday dinner",
            "timeEstimate": 10,
            "difficulty": "Easy"
          }
        ]
      },
      "nutritionalSummary": {
        "dailyAverages": {
          "calories": 1850,
          "protein": 95,
          "carbohydrates": 210,
          "fat": 70
        },
        "weeklyGoalAlignment": 0.87
      }
    },
    "alternatives": [
      {
        "day": "Wednesday", 
        "meal": "dinner",
        "suggestion": "Slow Cooker Beef Stew",
        "reason": "Busy day alternative - set and forget"
      }
    ],
    "aiInsights": [
      "Your family prefers Italian cuisine - included 3 Italian-inspired meals",
      "Scheduled quick meals on busy days (Monday, Wednesday)",
      "Included leftover plan for Thursday lunch"
    ]
  }
}
```

### Get AI Usage Statistics

**Endpoint:** `GET /usage-stats`

**Description:** Get AI usage statistics and cost tracking for the user.

**Query Parameters:**
- `period` (string) - "day", "week", "month" (default: "month")
- `startDate` (date) - Start date for custom period
- `endDate` (date) - End date for custom period

**Success Response (200 OK):**
```json
{
  "data": {
    "period": "month",
    "startDate": "2024-01-01",
    "endDate": "2024-01-31",
    "usage": {
      "totalRequests": 47,
      "requestsByType": {
        "mealSuggestions": 32,
        "recipePersonalization": 12,
        "weeklyMenus": 3
      },
      "totalTokensUsed": 58420,
      "totalCost": 0.292,
      "averageResponseTime": 3.8,
      "successRate": 0.96
    },
    "limits": {
      "monthlyRequestLimit": 200,
      "remainingRequests": 153,
      "resetDate": "2024-02-01"
    },
    "costBreakdown": {
      "inputTokens": 0.147,
      "outputTokens": 0.145,
      "processingFees": 0.000
    }
  }
}
```

### Provide AI Feedback

**Endpoint:** `POST /feedback`

**Description:** Provide feedback on AI suggestions to improve future recommendations.

**Request Body:**
```json
{
  "suggestionId": "suggestion_1",
  "requestId": "ai_req_12345",
  "feedback": {
    "overall": 4,
    "accuracy": 5,
    "usefulness": 4,
    "familyFit": 5
  },
  "comments": "Great suggestion! Family loved it, but prep time was longer than estimated.",
  "actualResults": {
    "madeRecipe": true,
    "actualPrepTime": 35,
    "actualCost": 20.00,
    "familyRatings": {
      "John": 5,
      "Jane": 4,
      "Emma": 5
    }
  },
  "improvements": [
    "More accurate time estimates",
    "Better cost estimation"
  ]
}
```

**Success Response (200 OK):**
```json
{
  "data": {
    "feedbackId": "feedback_789",
    "message": "Thank you for your feedback! This helps improve our AI suggestions.",
    "impactOnFuture": "Your feedback will be used to improve time and cost estimates for similar recipes."
  }
}
```

## AI Model Configuration

### Supported AI Models
- **gemini-pro**: Default model for complex meal planning
- **gemini-pro-vision**: For image-based recipe analysis
- **gemini-flash**: Faster responses for simple suggestions

### Request Optimization
- **Prompt Caching**: Common family profiles cached for faster responses
- **Batch Processing**: Multiple suggestions generated in single request
- **Response Streaming**: Large responses streamed for better UX

## Rate Limiting

### AI Endpoint Limits
- **Meal Suggestions**: 50 requests per hour per user
- **Recipe Personalization**: 30 requests per hour per user
- **Weekly Menu Generation**: 10 requests per hour per user
- **Feedback**: 100 requests per hour per user

### Subscription Tiers
- **Free**: 20 AI requests per month
- **Premium**: 200 AI requests per month
- **Family**: 500 AI requests per month
- **Enterprise**: Custom limits

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| AI_001 | 400 | Invalid AI request parameters |
| AI_002 | 429 | AI rate limit exceeded |
| AI_003 | 503 | AI service temporarily unavailable |
| AI_004 | 402 | AI quota exceeded |
| AI_005 | 422 | Insufficient family data for suggestions |
| AI_006 | 400 | Recipe not found for personalization |
| AI_007 | 500 | AI processing error |

## Security Considerations

### Data Privacy
- Family data used only for AI processing
- No data stored on AI provider servers
- Option to opt-out of AI training data
- Anonymized usage statistics only

### Request Validation
- All inputs sanitized before AI processing
- Content filtering for inappropriate responses
- Family-safe content enforcement
- Nutritional accuracy validation

---

These AI endpoints provide powerful meal planning capabilities while maintaining user privacy and data security through the Google Gemini AI integration.

*Documentation updated for API version 1.0*