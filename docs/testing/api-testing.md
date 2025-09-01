# API Testing Guide

## Overview
Comprehensive guide for API testing in the MealPrep application, covering automated testing strategies, manual testing procedures, performance testing, and API documentation validation using Postman, Newman, and custom testing frameworks.

## API Testing Strategy

### Testing Pyramid for APIs
```
    E2E API Tests (10%)
   ?? Contract Tests (15%)
  ?? Integration Tests (25%)
 ?? Unit Tests (50%)
```

### Testing Levels
1. **Unit Tests**: Individual endpoint logic testing
2. **Integration Tests**: API with database and external services
3. **Contract Tests**: API contract compliance and schema validation
4. **End-to-End Tests**: Complete user journey testing through APIs
5. **Performance Tests**: Load, stress, and endurance testing

---

## Postman Testing Framework

### Collection Structure

#### MealPrep API Test Collection
```json
{
  "info": {
    "name": "MealPrep API Tests",
    "description": "Comprehensive API testing suite for MealPrep application",
    "version": "1.0.0",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/"
  },
  "variable": [
    {
      "key": "baseUrl",
      "value": "{{protocol}}://{{host}}:{{port}}/api",
      "type": "string"
    },
    {
      "key": "authToken",
      "value": "",
      "type": "string"
    },
    {
      "key": "testUserId",
      "value": "",
      "type": "string"
    }
  ],
  "auth": {
    "type": "bearer",
    "bearer": [
      {
        "key": "token",
        "value": "{{authToken}}",
        "type": "string"
      }
    ]
  }
}
```

### Environment Configuration

#### Development Environment
```json
{
  "name": "Development",
  "values": [
    {
      "key": "protocol",
      "value": "https",
      "enabled": true
    },
    {
      "key": "host",
      "value": "localhost",
      "enabled": true
    },
    {
      "key": "port",
      "value": "7001",
      "enabled": true
    },
    {
      "key": "testEmail",
      "value": "test@mealprep.dev",
      "enabled": true
    },
    {
      "key": "testPassword",
      "value": "TestPassword123!",
      "enabled": true
    }
  ]
}
```

#### Production Environment
```json
{
  "name": "Production",
  "values": [
    {
      "key": "protocol",
      "value": "https",
      "enabled": true
    },
    {
      "key": "host",
      "value": "api.mealprep.com",
      "enabled": true
    },
    {
      "key": "port",
      "value": "443",
      "enabled": true
    },
    {
      "key": "testEmail",
      "value": "test@mealprep.com",
      "enabled": true
    },
    {
      "key": "testPassword",
      "value": "{{prodTestPassword}}",
      "enabled": true
    }
  ]
}
```

### Authentication Tests

#### Login Test Request
```json
{
  "name": "Authentication - Login",
  "request": {
    "method": "POST",
    "header": [
      {
        "key": "Content-Type",
        "value": "application/json"
      }
    ],
    "body": {
      "mode": "raw",
      "raw": "{\n  \"email\": \"{{testEmail}}\",\n  \"password\": \"{{testPassword}}\"\n}"
    },
    "url": {
      "raw": "{{baseUrl}}/auth/login",
      "host": ["{{baseUrl}}"],
      "path": ["auth", "login"]
    }
  },
  "event": [
    {
      "listen": "test",
      "script": {
        "exec": [
          "// Test response status",
          "pm.test('Login successful', function () {",
          "    pm.response.to.have.status(200);",
          "});",
          "",
          "// Test response structure",
          "pm.test('Response has required fields', function () {",
          "    const responseJson = pm.response.json();",
          "    pm.expect(responseJson).to.have.property('token');",
          "    pm.expect(responseJson).to.have.property('user');",
          "    pm.expect(responseJson).to.have.property('expiresAt');",
          "});",
          "",
          "// Store auth token for subsequent requests",
          "pm.test('Store auth token', function () {",
          "    const responseJson = pm.response.json();",
          "    if (responseJson.token) {",
          "        pm.globals.set('authToken', responseJson.token);",
          "        pm.globals.set('testUserId', responseJson.user.id);",
          "    }",
          "});",
          "",
          "// Validate token format",
          "pm.test('Token is valid JWT format', function () {",
          "    const responseJson = pm.response.json();",
          "    const tokenParts = responseJson.token.split('.');",
          "    pm.expect(tokenParts).to.have.lengthOf(3);",
          "});",
          "",
          "// Test response time",
          "pm.test('Response time is less than 2000ms', function () {",
          "    pm.expect(pm.response.responseTime).to.be.below(2000);",
          "});"
        ],
        "type": "text/javascript"
      }
    }
  ]
}
```

### Recipe API Tests

#### Create Recipe Test
```json
{
  "name": "Recipes - Create Recipe",
  "request": {
    "method": "POST",
    "header": [
      {
        "key": "Content-Type",
        "value": "application/json"
      },
      {
        "key": "Authorization",
        "value": "Bearer {{authToken}}"
      }
    ],
    "body": {
      "mode": "raw",
      "raw": "{\n  \"name\": \"API Test Recipe {{$timestamp}}\",\n  \"description\": \"A recipe created by API testing\",\n  \"instructions\": \"1. Mix ingredients\\n2. Cook for 30 minutes\\n3. Serve hot\",\n  \"prepTimeMinutes\": 15,\n  \"cookTimeMinutes\": 30,\n  \"servingSize\": 4,\n  \"difficultyLevel\": \"Easy\",\n  \"cuisine\": \"Italian\",\n  \"ingredients\": [\n    {\n      \"ingredientName\": \"Pasta\",\n      \"quantity\": 400,\n      \"unit\": \"grams\"\n    },\n    {\n      \"ingredientName\": \"Tomato Sauce\",\n      \"quantity\": 2,\n      \"unit\": \"cups\"\n    }\n  ]\n}"
    },
    "url": {
      "raw": "{{baseUrl}}/recipes",
      "host": ["{{baseUrl}}"],
      "path": ["recipes"]
    }
  },
  "event": [
    {
      "listen": "test",
      "script": {
        "exec": [
          "pm.test('Recipe created successfully', function () {",
          "    pm.response.to.have.status(201);",
          "});",
          "",
          "pm.test('Response contains recipe data', function () {",
          "    const responseJson = pm.response.json();",
          "    pm.expect(responseJson).to.have.property('id');",
          "    pm.expect(responseJson).to.have.property('name');",
          "    pm.expect(responseJson).to.have.property('ingredients');",
          "    pm.expect(responseJson.ingredients).to.be.an('array');",
          "    pm.expect(responseJson.ingredients).to.have.lengthOf(2);",
          "});",
          "",
          "pm.test('Recipe data matches request', function () {",
          "    const responseJson = pm.response.json();",
          "    pm.expect(responseJson.prepTimeMinutes).to.equal(15);",
          "    pm.expect(responseJson.cookTimeMinutes).to.equal(30);",
          "    pm.expect(responseJson.servingSize).to.equal(4);",
          "    pm.expect(responseJson.difficultyLevel).to.equal('Easy');",
          "});",
          "",
          "// Store recipe ID for cleanup",
          "pm.test('Store recipe ID', function () {",
          "    const responseJson = pm.response.json();",
          "    if (responseJson.id) {",
          "        pm.globals.set('testRecipeId', responseJson.id);",
          "    }",
          "});",
          "",
          "pm.test('Response time is acceptable', function () {",
          "    pm.expect(pm.response.responseTime).to.be.below(3000);",
          "});"
        ],
        "type": "text/javascript"
      }
    }
  ]
}
```

#### Search Recipes Test
```json
{
  "name": "Recipes - Search Recipes",
  "request": {
    "method": "GET",
    "header": [
      {
        "key": "Authorization",
        "value": "Bearer {{authToken}}"
      }
    ],
    "url": {
      "raw": "{{baseUrl}}/recipes/search?searchTerm=pasta&cuisine=Italian&maxPrepTime=30&page=1&pageSize=10",
      "host": ["{{baseUrl}}"],
      "path": ["recipes", "search"],
      "query": [
        {
          "key": "searchTerm",
          "value": "pasta"
        },
        {
          "key": "cuisine",
          "value": "Italian"
        },
        {
          "key": "maxPrepTime",
          "value": "30"
        },
        {
          "key": "page",
          "value": "1"
        },
        {
          "key": "pageSize",
          "value": "10"
        }
      ]
    }
  },
  "event": [
    {
      "listen": "test",
      "script": {
        "exec": [
          "pm.test('Search returns results', function () {",
          "    pm.response.to.have.status(200);",
          "});",
          "",
          "pm.test('Response is an array', function () {",
          "    const responseJson = pm.response.json();",
          "    pm.expect(responseJson).to.be.an('array');",
          "});",
          "",
          "pm.test('Search filters are applied', function () {",
          "    const responseJson = pm.response.json();",
          "    if (responseJson.length > 0) {",
          "        responseJson.forEach(recipe => {",
          "            pm.expect(recipe.prepTimeMinutes).to.be.at.most(30);",
          "            pm.expect(recipe.cuisine).to.equal('Italian');",
          "        });",
          "    }",
          "});",
          "",
          "pm.test('Recipe objects have required fields', function () {",
          "    const responseJson = pm.response.json();",
          "    if (responseJson.length > 0) {",
          "        const recipe = responseJson[0];",
          "        pm.expect(recipe).to.have.property('id');",
          "        pm.expect(recipe).to.have.property('name');",
          "        pm.expect(recipe).to.have.property('prepTimeMinutes');",
          "        pm.expect(recipe).to.have.property('cookTimeMinutes');",
          "    }",
          "});"
        ],
        "type": "text/javascript"
      }
    }
  ]
}
```

### AI Suggestion Tests

#### Generate AI Suggestions Test
```json
{
  "name": "AI - Generate Meal Suggestions",
  "request": {
    "method": "POST",
    "header": [
      {
        "key": "Content-Type",
        "value": "application/json"
      },
      {
        "key": "Authorization",
        "value": "Bearer {{authToken}}"
      }
    ],
    "body": {
      "mode": "raw",
      "raw": "{\n  \"mealType\": \"Dinner\",\n  \"maxPrepTime\": 45,\n  \"maxCookTime\": 60,\n  \"servingSize\": 4,\n  \"occasion\": \"Weeknight\",\n  \"dietaryRestrictions\": [\"vegetarian\"],\n  \"preferredCuisines\": [\"Italian\", \"Mediterranean\"]\n}"
    },
    "url": {
      "raw": "{{baseUrl}}/ai/suggestions",
      "host": ["{{baseUrl}}"],
      "path": ["ai", "suggestions"]
    }
  },
  "event": [
    {
      "listen": "test",
      "script": {
        "exec": [
          "pm.test('AI suggestions generated successfully', function () {",
          "    pm.response.to.have.status(200);",
          "});",
          "",
          "pm.test('Response contains suggestions', function () {",
          "    const responseJson = pm.response.json();",
          "    pm.expect(responseJson).to.have.property('suggestions');",
          "    pm.expect(responseJson.suggestions).to.be.an('array');",
          "    pm.expect(responseJson.suggestions).to.have.length.above(0);",
          "});",
          "",
          "pm.test('Suggestions have required fields', function () {",
          "    const responseJson = pm.response.json();",
          "    const suggestion = responseJson.suggestions[0];",
          "    pm.expect(suggestion).to.have.property('recipe');",
          "    pm.expect(suggestion).to.have.property('familyFitScore');",
          "    pm.expect(suggestion).to.have.property('reasoning');",
          "    pm.expect(suggestion.familyFitScore).to.be.a('number');",
          "    pm.expect(suggestion.familyFitScore).to.be.within(1, 10);",
          "});",
          "",
          "pm.test('AI respects dietary restrictions', function () {",
          "    const responseJson = pm.response.json();",
          "    // Verify suggestions respect vegetarian constraint",
          "    responseJson.suggestions.forEach(suggestion => {",
          "        const ingredients = suggestion.recipe.ingredients || [];",
          "        const hasNonVegetarian = ingredients.some(ing => ",
          "            /\\b(chicken|beef|pork|fish|meat)\\b/i.test(ing.ingredientName)",
          "        );",
          "        pm.expect(hasNonVegetarian).to.be.false;",
          "    });",
          "});",
          "",
          "pm.test('Response time is reasonable for AI processing', function () {",
          "    pm.expect(pm.response.responseTime).to.be.below(10000);",
          "});"
        ],
        "type": "text/javascript"
      }
    }
  ]
}
```

### Data Validation Tests

#### Schema Validation Helper
```javascript
// Pre-request script for schema validation
const ajv = require('ajv');
const schemaValidator = new ajv();

// Recipe schema
const recipeSchema = {
  type: "object",
  required: ["id", "name", "instructions", "prepTimeMinutes", "cookTimeMinutes", "servingSize"],
  properties: {
    id: { type: "integer", minimum: 1 },
    name: { type: "string", minLength: 1, maxLength: 100 },
    description: { type: "string", maxLength: 500 },
    instructions: { type: "string", minLength: 1 },
    prepTimeMinutes: { type: "integer", minimum: 0, maximum: 480 },
    cookTimeMinutes: { type: "integer", minimum: 0, maximum: 480 },
    servingSize: { type: "integer", minimum: 1, maximum: 20 },
    difficultyLevel: { type: "string", enum: ["Easy", "Medium", "Hard"] },
    cuisine: { type: "string", maxLength: 50 },
    averageRating: { type: "number", minimum: 0, maximum: 5 },
    ingredients: {
      type: "array",
      items: {
        type: "object",
        required: ["ingredientName", "quantity", "unit"],
        properties: {
          ingredientName: { type: "string", minLength: 1 },
          quantity: { type: "number", minimum: 0 },
          unit: { type: "string", minLength: 1 }
        }
      }
    }
  }
};

pm.globals.set("recipeSchema", JSON.stringify(recipeSchema));
```

#### Schema Validation Test
```javascript
// Test script for schema validation
pm.test('Response matches recipe schema', function () {
    const responseJson = pm.response.json();
    const schema = JSON.parse(pm.globals.get("recipeSchema"));
    const ajv = require('ajv');
    const validate = ajv.compile(schema);
    
    if (Array.isArray(responseJson)) {
        // Validate array of recipes
        responseJson.forEach((recipe, index) => {
            const valid = validate(recipe);
            pm.expect(valid, `Recipe at index ${index} should match schema. Errors: ${JSON.stringify(validate.errors)}`).to.be.true;
        });
    } else {
        // Validate single recipe
        const valid = validate(responseJson);
        pm.expect(valid, `Recipe should match schema. Errors: ${JSON.stringify(validate.errors)}`).to.be.true;
    }
});
```

---

## Newman CLI Testing

### Newman Configuration

#### Newman Test Runner Script
```bash
#!/bin/bash
# run-api-tests.sh

echo "Starting MealPrep API Tests..."

# Set environment variables
export NEWMAN_REPORTER_CLI_NO_SUMMARY=false
export NEWMAN_REPORTER_CLI_NO_FAILURES=false
export NEWMAN_REPORTER_CLI_NO_ASSERTIONS=false

# Test environments
ENVIRONMENTS=("development" "staging" "production")
COLLECTIONS=("auth" "recipes" "family" "ai" "menu")

# Run tests for each environment
for env in "${ENVIRONMENTS[@]}"; do
    echo "Testing environment: $env"
    
    # Run full test suite
    newman run "collections/MealPrep-API-Tests.postman_collection.json" \
        --environment "environments/$env.postman_environment.json" \
        --reporters cli,json,html \
        --reporter-json-export "reports/$env-test-results.json" \
        --reporter-html-export "reports/$env-test-report.html" \
        --timeout-request 10000 \
        --delay-request 100 \
        --bail \
        --color on
    
    # Check exit code
    if [ $? -ne 0 ]; then
        echo "Tests failed for $env environment"
        exit 1
    fi
done

echo "All API tests completed successfully!"
```

#### Newman with Docker
```dockerfile
# Dockerfile.newman
FROM postman/newman:5-alpine

# Copy test files
COPY collections/ /etc/newman/collections/
COPY environments/ /etc/newman/environments/
COPY data/ /etc/newman/data/

# Create reports directory
RUN mkdir -p /etc/newman/reports

# Set working directory
WORKDIR /etc/newman

# Default command
CMD ["run", "collections/MealPrep-API-Tests.postman_collection.json", \
     "--environment", "environments/ci.postman_environment.json", \
     "--reporters", "cli,json", \
     "--reporter-json-export", "reports/test-results.json"]
```

### CI/CD Integration

#### GitHub Actions Workflow
```yaml
name: API Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 */6 * * *'  # Run every 6 hours

jobs:
  api-tests:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    
    - name: Install Newman
      run: npm install -g newman newman-reporter-html
    
    - name: Wait for API to be ready
      run: |
        timeout 300 bash -c 'until curl -f ${{ secrets.API_BASE_URL }}/health; do sleep 5; done'
    
    - name: Run API Tests
      run: |
        newman run tests/collections/MealPrep-API-Tests.postman_collection.json \
          --environment tests/environments/ci.postman_environment.json \
          --env-var "baseUrl=${{ secrets.API_BASE_URL }}" \
          --env-var "testEmail=${{ secrets.TEST_EMAIL }}" \
          --env-var "testPassword=${{ secrets.TEST_PASSWORD }}" \
          --reporters cli,html \
          --reporter-html-export reports/api-test-report.html \
          --bail
    
    - name: Upload Test Reports
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: api-test-reports
        path: reports/
    
    - name: Notify on Failure
      if: failure()
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Performance Testing

### Load Testing with Newman

#### Performance Test Collection
```json
{
  "name": "MealPrep API Performance Tests",
  "item": [
    {
      "name": "Recipe Search Load Test",
      "request": {
        "method": "GET",
        "header": [
          {
            "key": "Authorization",
            "value": "Bearer {{authToken}}"
          }
        ],
        "url": {
          "raw": "{{baseUrl}}/recipes/search?searchTerm={{randomSearchTerm}}&page={{randomPage}}",
          "host": ["{{baseUrl}}"],
          "path": ["recipes", "search"],
          "query": [
            {
              "key": "searchTerm",
              "value": "{{randomSearchTerm}}"
            },
            {
              "key": "page",
              "value": "{{randomPage}}"
            }
          ]
        }
      },
      "event": [
        {
          "listen": "test",
          "script": {
            "exec": [
              "pm.test('Performance - Response time under 500ms', function () {",
              "    pm.expect(pm.response.responseTime).to.be.below(500);",
              "});",
              "",
              "pm.test('Performance - Successful response', function () {",
              "    pm.response.to.have.status(200);",
              "});",
              "",
              "// Collect performance metrics",
              "pm.globals.set('responseTime_' + pm.info.iteration, pm.response.responseTime);",
              "",
              "// Calculate running average",
              "if (pm.info.iteration > 0) {",
              "    let totalTime = 0;",
              "    for (let i = 0; i <= pm.info.iteration; i++) {",
              "        totalTime += parseInt(pm.globals.get('responseTime_' + i) || 0);",
              "    }",
              "    const avgTime = totalTime / (pm.info.iteration + 1);",
              "    pm.globals.set('averageResponseTime', avgTime);",
              "    ",
              "    pm.test('Performance - Average response time acceptable', function () {",
              "        pm.expect(avgTime).to.be.below(1000);",
              "    });",
              "}"
            ],
            "type": "text/javascript"
          }
        }
      ]
    }
  ]
}
```

#### Load Testing Script
```bash
#!/bin/bash
# load-test.sh

echo "Starting Load Test..."

# Run load test with 100 iterations and 10 concurrent users
newman run collections/Performance-Tests.postman_collection.json \
    --environment environments/production.postman_environment.json \
    --iteration-count 100 \
    --delay-request 100 \
    --reporters cli,json \
    --reporter-json-export reports/load-test-results.json

# Parse results and generate report
node scripts/parse-performance-results.js reports/load-test-results.json
```

#### Performance Results Parser
```javascript
// scripts/parse-performance-results.js
const fs = require('fs');
const path = require('path');

function parsePerformanceResults(resultsFile) {
    const results = JSON.parse(fs.readFileSync(resultsFile, 'utf8'));
    
    const metrics = {
        totalRequests: results.run.stats.requests.total,
        failedRequests: results.run.stats.requests.failed,
        averageResponseTime: 0,
        minResponseTime: Number.MAX_VALUE,
        maxResponseTime: 0,
        p95ResponseTime: 0,
        p99ResponseTime: 0
    };
    
    const responseTimes = [];
    
    results.run.executions.forEach(execution => {
        const responseTime = execution.response?.responseTime || 0;
        responseTimes.push(responseTime);
        
        metrics.minResponseTime = Math.min(metrics.minResponseTime, responseTime);
        metrics.maxResponseTime = Math.max(metrics.maxResponseTime, responseTime);
    });
    
    // Calculate percentiles
    responseTimes.sort((a, b) => a - b);
    metrics.averageResponseTime = responseTimes.reduce((a, b) => a + b, 0) / responseTimes.length;
    metrics.p95ResponseTime = responseTimes[Math.floor(responseTimes.length * 0.95)];
    metrics.p99ResponseTime = responseTimes[Math.floor(responseTimes.length * 0.99)];
    
    // Generate performance report
    const report = `
# API Performance Test Report

## Summary
- **Total Requests**: ${metrics.totalRequests}
- **Failed Requests**: ${metrics.failedRequests}
- **Success Rate**: ${((metrics.totalRequests - metrics.failedRequests) / metrics.totalRequests * 100).toFixed(2)}%

## Response Times
- **Average**: ${metrics.averageResponseTime.toFixed(2)}ms
- **Minimum**: ${metrics.minResponseTime}ms
- **Maximum**: ${metrics.maxResponseTime}ms
- **95th Percentile**: ${metrics.p95ResponseTime}ms
- **99th Percentile**: ${metrics.p99ResponseTime}ms

## Performance Targets
- ? Average response time < 1000ms: ${metrics.averageResponseTime < 1000 ? 'PASS' : 'FAIL'}
- ? 95th percentile < 2000ms: ${metrics.p95ResponseTime < 2000 ? 'PASS' : 'FAIL'}
- ? Success rate > 99%: ${((metrics.totalRequests - metrics.failedRequests) / metrics.totalRequests) > 0.99 ? 'PASS' : 'FAIL'}

Generated at: ${new Date().toISOString()}
    `;
    
    fs.writeFileSync('reports/performance-report.md', report);
    console.log('Performance report generated: reports/performance-report.md');
    
    // Exit with error code if performance targets not met
    if (metrics.averageResponseTime >= 1000 || metrics.p95ResponseTime >= 2000) {
        console.error('Performance targets not met!');
        process.exit(1);
    }
}

const resultsFile = process.argv[2];
if (!resultsFile) {
    console.error('Usage: node parse-performance-results.js <results-file>');
    process.exit(1);
}

parsePerformanceResults(resultsFile);
```

---

## Contract Testing

### API Contract Validation

#### OpenAPI Specification Testing
```javascript
// Contract testing with OpenAPI spec
pm.test('Response matches OpenAPI specification', function () {
    const OpenAPISnippet = require('openapi-snippet');
    const spec = pm.globals.get('openApiSpec');
    const endpoint = pm.request.url.getPath();
    const method = pm.request.method.toLowerCase();
    
    // Validate response against OpenAPI schema
    const responseSchema = spec.paths[endpoint][method].responses[pm.response.code.toString()];
    
    if (responseSchema && responseSchema.content && responseSchema.content['application/json']) {
        const schema = responseSchema.content['application/json'].schema;
        const responseBody = pm.response.json();
        
        // Use AJV for schema validation
        const Ajv = require('ajv');
        const ajv = new Ajv();
        const validate = ajv.compile(schema);
        const valid = validate(responseBody);
        
        pm.expect(valid, 'Response should match OpenAPI schema: ' + JSON.stringify(validate.errors)).to.be.true;
    }
});
```

### Consumer-Driven Contract Testing

#### Pact Testing Setup
```javascript
// pact-consumer-test.js
const { Pact } = require('@pact-foundation/pact');
const { MealPrepApiClient } = require('../src/api-client');

describe('MealPrep API Consumer Tests', () => {
    const provider = new Pact({
        consumer: 'MealPrep-Frontend',
        provider: 'MealPrep-API',
        port: 1234,
        log: path.resolve(process.cwd(), 'logs', 'pact.log'),
        dir: path.resolve(process.cwd(), 'pacts'),
        logLevel: 'INFO'
    });

    beforeAll(() => provider.setup());
    afterAll(() => provider.finalize());

    describe('Recipe API', () => {
        it('should return recipes when searching', async () => {
            // Arrange
            await provider.addInteraction({
                state: 'recipes exist',
                uponReceiving: 'a request for recipe search',
                withRequest: {
                    method: 'GET',
                    path: '/api/recipes/search',
                    query: 'searchTerm=chicken&page=1&pageSize=10',
                    headers: {
                        'Authorization': 'Bearer validToken',
                        'Accept': 'application/json'
                    }
                },
                willRespondWith: {
                    status: 200,
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: [
                        {
                            id: 1,
                            name: 'Chicken Parmesan',
                            description: 'Delicious chicken dish',
                            prepTimeMinutes: 20,
                            cookTimeMinutes: 25,
                            servingSize: 4,
                            difficultyLevel: 'Medium',
                            cuisine: 'Italian'
                        }
                    ]
                }
            });

            // Act
            const client = new MealPrepApiClient('http://localhost:1234');
            const recipes = await client.searchRecipes({
                searchTerm: 'chicken',
                page: 1,
                pageSize: 10
            });

            // Assert
            expect(recipes).toHaveLength(1);
            expect(recipes[0].name).toBe('Chicken Parmesan');
        });
    });
});
```

---

## Security Testing

### Authentication Security Tests

#### JWT Token Security Validation
```javascript
// JWT security tests
pm.test('JWT token has proper security', function () {
    const token = pm.globals.get('authToken');
    const tokenParts = token.split('.');
    
    // Decode header and payload
    const header = JSON.parse(atob(tokenParts[0]));
    const payload = JSON.parse(atob(tokenParts[1]));
    
    // Validate security properties
    pm.expect(header.alg).to.equal('HS256'); // Ensure secure algorithm
    pm.expect(payload.exp).to.be.a('number'); // Has expiration
    pm.expect(payload.iat).to.be.a('number'); // Has issued at
    
    // Check expiration is reasonable (not too long)
    const expirationTime = payload.exp * 1000;
    const issuedTime = payload.iat * 1000;
    const tokenLifetime = expirationTime - issuedTime;
    pm.expect(tokenLifetime).to.be.below(24 * 60 * 60 * 1000); // Less than 24 hours
});
```

#### Authorization Tests
```json
{
  "name": "Security - Unauthorized Access",
  "request": {
    "method": "GET",
    "header": [],
    "url": {
      "raw": "{{baseUrl}}/recipes",
      "host": ["{{baseUrl}}"],
      "path": ["recipes"]
    }
  },
  "event": [
    {
      "listen": "test",
      "script": {
        "exec": [
          "pm.test('Unauthorized request returns 401', function () {",
          "    pm.response.to.have.status(401);",
          "});",
          "",
          "pm.test('Error message indicates authentication required', function () {",
          "    const responseJson = pm.response.json();",
          "    pm.expect(responseJson.message).to.include('Authentication');",
          "});"
        ],
        "type": "text/javascript"
      }
    }
  ]
}
```

### Input Validation Security Tests

#### SQL Injection Protection Test
```json
{
  "name": "Security - SQL Injection Protection",
  "request": {
    "method": "GET",
    "header": [
      {
        "key": "Authorization",
        "value": "Bearer {{authToken}}"
      }
    ],
    "url": {
      "raw": "{{baseUrl}}/recipes/search?searchTerm=' OR '1'='1'; DROP TABLE recipes; --",
      "host": ["{{baseUrl}}"],
      "path": ["recipes", "search"],
      "query": [
        {
          "key": "searchTerm",
          "value": "' OR '1'='1'; DROP TABLE recipes; --"
        }
      ]
    }
  },
  "event": [
    {
      "listen": "test",
      "script": {
        "exec": [
          "pm.test('SQL injection attempt handled safely', function () {",
          "    // Should return 400 (bad request) or 200 with no results",
          "    pm.expect([200, 400]).to.include(pm.response.code);",
          "    ",
          "    if (pm.response.code === 200) {",
          "        const responseJson = pm.response.json();",
          "        pm.expect(responseJson).to.be.an('array');",
          "        // Verify no actual SQL injection occurred",
          "    }",
          "});",
          "",
          "pm.test('No error stack traces in response', function () {",
          "    const responseText = pm.response.text();",
          "    pm.expect(responseText).to.not.include('System.Data.SqlClient');",
          "    pm.expect(responseText).to.not.include('at Microsoft');",
          "    pm.expect(responseText).to.not.include('StackTrace');",
          "});"
        ],
        "type": "text/javascript"
      }
    }
  ]
}
```

---

## Test Data Management

### Dynamic Test Data Generation

#### Data Faker Helper
```javascript
// Pre-request script for generating test data
const faker = require('faker');

// Generate random test data
pm.globals.set('randomRecipeName', `Test Recipe ${faker.lorem.words(2)} ${Date.now()}`);
pm.globals.set('randomEmail', faker.internet.email());
pm.globals.set('randomSearchTerm', faker.random.arrayElement(['chicken', 'pasta', 'salad', 'soup']));
pm.globals.set('randomPage', faker.random.number({ min: 1, max: 5 }));
pm.globals.set('randomIngredient', faker.random.arrayElement(['tomatoes', 'onions', 'garlic', 'basil']));

// Generate consistent test user data
const testUser = {
    firstName: 'API',
    lastName: 'TestUser',
    email: 'apitest@mealprep.com',
    password: 'TestPassword123!'
};

pm.globals.set('testUserData', JSON.stringify(testUser));
```

### Test Cleanup

#### Cleanup Test Data
```json
{
  "name": "Cleanup - Delete Test Recipe",
  "request": {
    "method": "DELETE",
    "header": [
      {
        "key": "Authorization",
        "value": "Bearer {{authToken}}"
      }
    ],
    "url": {
      "raw": "{{baseUrl}}/recipes/{{testRecipeId}}",
      "host": ["{{baseUrl}}"],
      "path": ["recipes", "{{testRecipeId}}"]
    }
  },
  "event": [
    {
      "listen": "test",
      "script": {
        "exec": [
          "pm.test('Test recipe deleted successfully', function () {",
          "    pm.expect([200, 204, 404]).to.include(pm.response.code);",
          "});",
          "",
          "// Clear stored test data",
          "pm.globals.unset('testRecipeId');"
        ],
        "type": "text/javascript"
      }
    }
  ]
}
```

---

## Monitoring and Reporting

### Test Results Dashboard

#### Test Metrics Collection
```javascript
// Collect test metrics
pm.test('Collect API metrics', function () {
    const metrics = {
        endpoint: pm.request.url.toString(),
        method: pm.request.method,
        status: pm.response.code,
        responseTime: pm.response.responseTime,
        timestamp: new Date().toISOString(),
        success: pm.response.code >= 200 && pm.response.code < 300
    };
    
    // Store metrics for reporting
    const existingMetrics = JSON.parse(pm.globals.get('apiMetrics') || '[]');
    existingMetrics.push(metrics);
    pm.globals.set('apiMetrics', JSON.stringify(existingMetrics));
});
```

### Continuous Monitoring

#### Health Check Collection
```json
{
  "name": "Health Check - API Status",
  "request": {
    "method": "GET",
    "header": [],
    "url": {
      "raw": "{{baseUrl}}/health",
      "host": ["{{baseUrl}}"],
      "path": ["health"]
    }
  },
  "event": [
    {
      "listen": "test",
      "script": {
        "exec": [
          "pm.test('API is healthy', function () {",
          "    pm.response.to.have.status(200);",
          "});",
          "",
          "pm.test('Health check includes required services', function () {",
          "    const responseJson = pm.response.json();",
          "    pm.expect(responseJson).to.have.property('database');",
          "    pm.expect(responseJson).to.have.property('ai_service');",
          "    pm.expect(responseJson).to.have.property('email_service');",
          "    ",
          "    pm.expect(responseJson.database).to.equal('healthy');",
          "    pm.expect(responseJson.ai_service).to.equal('healthy');",
          "});"
        ],
        "type": "text/javascript"
      }
    }
  ]
}
```

---

*Last Updated: December 2024*  
*API testing guide continuously updated with new testing patterns and best practices*