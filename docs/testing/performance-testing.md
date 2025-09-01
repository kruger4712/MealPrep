# Performance Testing Guide

## Overview
Comprehensive guide for performance testing in the MealPrep application, covering load testing, stress testing, endurance testing, spike testing, and scalability testing for both frontend and backend components.

## Performance Testing Strategy

### Performance Testing Philosophy
- **User Experience First**: Test from the user's perspective with real-world scenarios
- **Early and Continuous**: Integrate performance testing throughout the development cycle
- **Realistic Workloads**: Use production-like data volumes and user patterns
- **Baseline and Trend**: Establish baselines and monitor performance trends over time
- **Scalability Planning**: Test beyond current requirements to plan for growth

### Performance Testing Types
1. **Load Testing**: Normal expected load
2. **Stress Testing**: Beyond normal capacity
3. **Spike Testing**: Sudden load increases
4. **Endurance Testing**: Extended periods
5. **Volume Testing**: Large amounts of data
6. **Scalability Testing**: System growth capacity

### Performance Requirements
```yaml
Performance Targets:
  API Response Times:
    - Average: < 500ms
    - 95th Percentile: < 1000ms
    - 99th Percentile: < 2000ms
  
  Frontend Performance:
    - First Contentful Paint: < 1.5s
    - Largest Contentful Paint: < 2.5s
    - Cumulative Layout Shift: < 0.1
    - First Input Delay: < 100ms
  
  Database Performance:
    - Query Response: < 100ms (average)
    - Complex Queries: < 500ms
    - Connection Pool: 95% utilization
  
  Throughput:
    - API Requests: 1000 req/sec
    - Concurrent Users: 500 users
    - Database Connections: 100 concurrent
```

---

## Load Testing with k6

### k6 Installation and Setup
```bash
# Install k6
curl https://github.com/grafana/k6/releases/download/v0.47.0/k6-v0.47.0-linux-amd64.tar.gz -L | tar xvz --strip-components 1

# Or using package manager
sudo apt-get install k6

# Verify installation
k6 version
```

### Basic Load Testing Script
```javascript
// scripts/load-tests/basic-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '2m', target: 10 },   // Ramp up to 10 users
    { duration: '5m', target: 10 },   // Stay at 10 users
    { duration: '2m', target: 50 },   // Ramp up to 50 users
    { duration: '5m', target: 50 },   // Stay at 50 users
    { duration: '2m', target: 0 },    // Ramp down to 0 users
  ],
  thresholds: {
    http_req_duration: ['p(95)<1000', 'p(99)<2000'], // 95% of requests under 1s, 99% under 2s
    http_req_failed: ['rate<0.1'],    // Less than 10% failure rate
    errors: ['rate<0.1'],             // Less than 10% custom error rate
  },
};

// Test configuration
const BASE_URL = __ENV.BASE_URL || 'https://api.mealprep.com';
const API_TOKEN = __ENV.API_TOKEN || '';

export function setup() {
  // Login to get authentication token
  const loginResponse = http.post(`${BASE_URL}/auth/login`, {
    email: 'loadtest@mealprep.com',
    password: 'LoadTest123!'
  });
  
  check(loginResponse, {
    'login successful': (r) => r.status === 200,
  });
  
  const token = loginResponse.json('token');
  return { token };
}

export default function(data) {
  const headers = {
    'Authorization': `Bearer ${data.token}`,
    'Content-Type': 'application/json',
  };

  // Test scenario: Browse recipes
  const recipesResponse = http.get(`${BASE_URL}/api/recipes`, { headers });
  
  const recipesCheck = check(recipesResponse, {
    'recipes loaded': (r) => r.status === 200,
    'recipes response time OK': (r) => r.timings.duration < 1000,
    'recipes returned': (r) => r.json().length > 0,
  });
  
  errorRate.add(!recipesCheck);
  
  sleep(1);

  // Test scenario: Search recipes
  const searchResponse = http.get(
    `${BASE_URL}/api/recipes/search?searchTerm=chicken&cuisine=Italian`, 
    { headers }
  );
  
  const searchCheck = check(searchResponse, {
    'search successful': (r) => r.status === 200,
    'search response time OK': (r) => r.timings.duration < 1500,
  });
  
  errorRate.add(!searchCheck);
  
  sleep(2);

  // Test scenario: Get AI suggestions
  const aiResponse = http.post(`${BASE_URL}/api/ai/suggestions`, {
    mealType: 'Dinner',
    maxPrepTime: 30,
    servingSize: 4,
    occasion: 'Weeknight'
  }, { headers });
  
  const aiCheck = check(aiResponse, {
    'AI suggestions generated': (r) => r.status === 200,
    'AI response time acceptable': (r) => r.timings.duration < 5000,
    'AI suggestions returned': (r) => r.json('suggestions').length > 0,
  });
  
  errorRate.add(!aiCheck);
  
  sleep(3);
}

export function teardown(data) {
  // Cleanup if needed
  console.log('Load test completed');
}
```

### Advanced Recipe Management Load Test
```javascript
// scripts/load-tests/recipe-management-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { SharedArray } from 'k6/data';

// Load test data
const recipes = new SharedArray('recipes', function () {
  return JSON.parse(open('./test-data/sample-recipes.json'));
});

export const options = {
  scenarios: {
    browse_recipes: {
      executor: 'ramping-vus',
      stages: [
        { duration: '2m', target: 20 },
        { duration: '5m', target: 20 },
        { duration: '2m', target: 0 },
      ],
      gracefulRampDown: '30s',
    },
    create_recipes: {
      executor: 'constant-vus',
      vus: 5,
      duration: '10m',
    },
    search_recipes: {
      executor: 'ramping-arrival-rate',
      stages: [
        { duration: '2m', target: 10 },
        { duration: '5m', target: 30 },
        { duration: '2m', target: 0 },
      ],
      preAllocatedVUs: 10,
      maxVUs: 50,
    },
  },
  thresholds: {
    'http_req_duration{scenario:browse_recipes}': ['p(95)<800'],
    'http_req_duration{scenario:create_recipes}': ['p(95)<1500'],
    'http_req_duration{scenario:search_recipes}': ['p(95)<600'],
    'http_req_failed': ['rate<0.05'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'https://api.mealprep.com';

export function setup() {
  // Authenticate and return token
  const response = http.post(`${BASE_URL}/auth/login`, {
    email: 'perftest@mealprep.com',
    password: 'PerfTest123!'
  });
  
  return { token: response.json('token') };
}

export default function(data) {
  const headers = {
    'Authorization': `Bearer ${data.token}`,
    'Content-Type': 'application/json',
  };

  // Scenario-specific behavior
  const scenario = __ENV.K6_SCENARIO_NAME;
  
  switch (scenario) {
    case 'browse_recipes':
      browseRecipes(headers);
      break;
    case 'create_recipes':
      createRecipe(headers);
      break;
    case 'search_recipes':
      searchRecipes(headers);
      break;
    default:
      browseRecipes(headers);
  }
}

function browseRecipes(headers) {
  // Get recipes list
  const response = http.get(`${BASE_URL}/api/recipes?page=1&pageSize=20`, { headers });
  
  check(response, {
    'recipes loaded': (r) => r.status === 200,
    'recipes count correct': (r) => r.json().length <= 20,
  });
  
  sleep(1);
  
  // Get random recipe details
  if (response.status === 200) {
    const recipes = response.json();
    if (recipes.length > 0) {
      const randomRecipe = recipes[Math.floor(Math.random() * recipes.length)];
      
      const detailResponse = http.get(`${BASE_URL}/api/recipes/${randomRecipe.id}`, { headers });
      
      check(detailResponse, {
        'recipe detail loaded': (r) => r.status === 200,
        'recipe has ingredients': (r) => r.json('ingredients').length > 0,
      });
    }
  }
  
  sleep(2);
}

function createRecipe(headers) {
  const recipeData = recipes[Math.floor(Math.random() * recipes.length)];
  
  const response = http.post(`${BASE_URL}/api/recipes`, JSON.stringify({
    ...recipeData,
    name: `${recipeData.name} - Load Test ${Date.now()}`,
  }), { headers });
  
  const recipeId = response.json('id');
  
  check(response, {
    'recipe created': (r) => r.status === 201,
    'recipe has ID': (r) => recipeId > 0,
  });
  
  // Clean up - delete the test recipe
  if (recipeId) {
    sleep(1);
    const deleteResponse = http.del(`${BASE_URL}/api/recipes/${recipeId}`, null, { headers });
    
    check(deleteResponse, {
      'recipe deleted': (r) => r.status === 204,
    });
  }
  
  sleep(5);
}

function searchRecipes(headers) {
  const searchTerms = ['chicken', 'pasta', 'vegetarian', 'quick', 'healthy'];
  const cuisines = ['Italian', 'Mexican', 'Chinese', 'American', 'Indian'];
  
  const searchTerm = searchTerms[Math.floor(Math.random() * searchTerms.length)];
  const cuisine = Math.random() > 0.5 ? cuisines[Math.floor(Math.random() * cuisines.length)] : '';
  
  let url = `${BASE_URL}/api/recipes/search?searchTerm=${searchTerm}`;
  if (cuisine) {
    url += `&cuisine=${cuisine}`;
  }
  
  const response = http.get(url, { headers });
  
  check(response, {
    'search completed': (r) => r.status === 200,
    'search results returned': (r) => Array.isArray(r.json()),
  });
  
  sleep(1);
}
```

---

## Backend Performance Testing

### Database Performance Testing
```javascript
// scripts/load-tests/database-performance-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    light_load: {
      executor: 'constant-vus',
      vus: 10,
      duration: '5m',
    },
    medium_load: {
      executor: 'ramping-vus',
      stages: [
        { duration: '2m', target: 50 },
        { duration: '5m', target: 50 },
        { duration: '2m', target: 0 },
      ],
      startTime: '5m',
    },
    heavy_load: {
      executor: 'ramping-vus',
      stages: [
        { duration: '1m', target: 100 },
        { duration: '3m', target: 100 },
        { duration: '1m', target: 0 },
      ],
      startTime: '12m',
    },
  },
  thresholds: {
    'http_req_duration{test_type:simple_query}': ['p(95)<100'],
    'http_req_duration{test_type:complex_query}': ['p(95)<500'],
    'http_req_duration{test_type:aggregation}': ['p(95)<1000'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'https://api.mealprep.com';

export function setup() {
  const response = http.post(`${BASE_URL}/auth/login`, {
    email: 'dbtest@mealprep.com',
    password: 'DbTest123!'
  });
  
  return { token: response.json('token') };
}

export default function(data) {
  const headers = {
    'Authorization': `Bearer ${data.token}`,
    'Content-Type': 'application/json',
  };

  // Test simple database queries
  testSimpleQueries(headers);
  sleep(1);
  
  // Test complex queries with joins
  testComplexQueries(headers);
  sleep(1);
  
  // Test aggregation queries
  testAggregationQueries(headers);
  sleep(2);
}

function testSimpleQueries(headers) {
  // Simple recipe lookup by ID
  const recipeId = Math.floor(Math.random() * 1000) + 1;
  const response = http.get(`${BASE_URL}/api/recipes/${recipeId}`, { 
    headers,
    tags: { test_type: 'simple_query' }
  });
  
  check(response, {
    'simple query successful': (r) => r.status === 200 || r.status === 404,
    'simple query fast': (r) => r.timings.duration < 100,
  });
}

function testComplexQueries(headers) {
  // Complex search with multiple filters
  const response = http.get(
    `${BASE_URL}/api/recipes/search?searchTerm=chicken&cuisine=Italian&maxPrepTime=30&minRating=4&includeTags=true`,
    { 
      headers,
      tags: { test_type: 'complex_query' }
    }
  );
  
  check(response, {
    'complex query successful': (r) => r.status === 200,
    'complex query acceptable': (r) => r.timings.duration < 500,
    'results include tags': (r) => {
      const results = r.json();
      return results.length === 0 || results[0].tags !== undefined;
    },
  });
}

function testAggregationQueries(headers) {
  // Recipe statistics and aggregations
  const response = http.get(`${BASE_URL}/api/recipes/statistics`, { 
    headers,
    tags: { test_type: 'aggregation' }
  });
  
  check(response, {
    'aggregation query successful': (r) => r.status === 200,
    'aggregation query reasonable': (r) => r.timings.duration < 1000,
    'statistics returned': (r) => {
      const stats = r.json();
      return stats.totalRecipes !== undefined && stats.averageRating !== undefined;
    },
  });
}
```

### AI Service Performance Testing
```javascript
// scripts/load-tests/ai-performance-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    ai_suggestions: {
      executor: 'ramping-vus',
      stages: [
        { duration: '1m', target: 5 },    // AI services are expensive
        { duration: '3m', target: 5 },
        { duration: '1m', target: 10 },
        { duration: '2m', target: 10 },
        { duration: '1m', target: 0 },
      ],
    },
  },
  thresholds: {
    'http_req_duration{endpoint:ai_suggestions}': ['p(95)<10000'], // AI can be slow
    'http_req_failed{endpoint:ai_suggestions}': ['rate<0.05'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'https://api.mealprep.com';

export function setup() {
  const response = http.post(`${BASE_URL}/auth/login`, {
    email: 'aitest@mealprep.com',
    password: 'AiTest123!'
  });
  
  return { token: response.json('token') };
}

export default function(data) {
  const headers = {
    'Authorization': `Bearer ${data.token}`,
    'Content-Type': 'application/json',
  };

  // Test AI suggestion generation
  const aiRequest = {
    mealType: ['Breakfast', 'Lunch', 'Dinner'][Math.floor(Math.random() * 3)],
    maxPrepTime: [15, 30, 45, 60][Math.floor(Math.random() * 4)],
    maxCookTime: [20, 40, 60, 90][Math.floor(Math.random() * 4)],
    servingSize: Math.floor(Math.random() * 6) + 2,
    occasion: ['Weeknight', 'Weekend', 'Special'][Math.floor(Math.random() * 3)],
    dietaryRestrictions: Math.random() > 0.7 ? ['vegetarian'] : [],
    preferredCuisines: Math.random() > 0.5 ? ['Italian', 'Mexican'] : [],
  };

  const response = http.post(
    `${BASE_URL}/api/ai/suggestions`,
    JSON.stringify(aiRequest),
    { 
      headers,
      tags: { endpoint: 'ai_suggestions' }
    }
  );
  
  check(response, {
    'AI suggestions generated': (r) => r.status === 200,
    'AI response has suggestions': (r) => r.json('suggestions').length > 0,
    'AI suggestions have scores': (r) => {
      const suggestions = r.json('suggestions');
      return suggestions.every(s => s.familyFitScore >= 1 && s.familyFitScore <= 10);
    },
    'AI response time acceptable': (r) => r.timings.duration < 15000,
  });
  
  // AI requests are expensive, so space them out
  sleep(10);
}
```

---

## Frontend Performance Testing

### Lighthouse Performance Testing
```javascript
// scripts/performance/lighthouse-test.js
const lighthouse = require('lighthouse');
const chromeLauncher = require('chrome-launcher');
const fs = require('fs');

async function runLighthouseTest(url, options = {}) {
  const chrome = await chromeLauncher.launch({
    chromeFlags: ['--headless', '--disable-gpu', '--no-sandbox']
  });
  
  const opts = {
    logLevel: 'info',
    output: 'json',
    onlyCategories: ['performance'],
    port: chrome.port,
    ...options
  };
  
  const runnerResult = await lighthouse(url, opts);
  
  await chrome.kill();
  
  return runnerResult;
}

async function performanceTest() {
  const urls = [
    'https://mealprep.com',
    'https://mealprep.com/recipes',
    'https://mealprep.com/recipes/search',
    'https://mealprep.com/meal-planning',
    'https://mealprep.com/profile'
  ];
  
  const results = {};
  
  for (const url of urls) {
    console.log(`Testing ${url}...`);
    
    const result = await runLighthouseTest(url);
    const performance = result.report.categories.performance.score * 100;
    
    results[url] = {
      performanceScore: performance,
      firstContentfulPaint: result.report.audits['first-contentful-paint'].displayValue,
      largestContentfulPaint: result.report.audits['largest-contentful-paint'].displayValue,
      speedIndex: result.report.audits['speed-index'].displayValue,
      cumulativeLayoutShift: result.report.audits['cumulative-layout-shift'].displayValue,
      firstInputDelay: result.report.audits['max-potential-fid'].displayValue,
    };
    
    console.log(`Performance Score: ${performance}`);
  }
  
  // Save results
  fs.writeFileSync(
    `reports/lighthouse-${Date.now()}.json`,
    JSON.stringify(results, null, 2)
  );
  
  // Check performance thresholds
  const failures = [];
  Object.entries(results).forEach(([url, metrics]) => {
    if (metrics.performanceScore < 90) {
      failures.push(`${url}: Performance score ${metrics.performanceScore} below threshold (90)`);
    }
  });
  
  if (failures.length > 0) {
    console.error('Performance tests failed:');
    failures.forEach(failure => console.error(`  ${failure}`));
    process.exit(1);
  }
  
  console.log('All performance tests passed!');
}

performanceTest().catch(console.error);
```

---

## Continuous Performance Testing

### CI/CD Pipeline Integration
```yaml
# .github/workflows/performance-tests.yml
name: Performance Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1-5'  # Daily at 2 AM, weekdays only

jobs:
  performance-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    
    - name: Install k6
      run: |
        curl https://github.com/grafana/k6/releases/download/v0.47.0/k6-v0.47.0-linux-amd64.tar.gz -L | tar xvz --strip-components 1
        sudo mv k6 /usr/local/bin/
    
    - name: Build application
      run: |
        dotnet build --configuration Release
        cd ClientApp && npm ci && npm run build
    
    - name: Start application
      run: |
        dotnet run --configuration Release &
        sleep 30  # Wait for app to start
    
    - name: Run load tests
      run: |
        k6 run scripts/load-tests/basic-load-test.js \
          --env BASE_URL=http://localhost:5000 \
          --out influxdb=http://localhost:8086/k6
    
    - name: Run frontend performance tests
      run: |
        npm install -g lighthouse
        node scripts/performance/lighthouse-test.js
    
    - name: Upload performance reports
      uses: actions/upload-artifact@v3
      with:
        name: performance-reports
        path: |
          reports/
          traces/
    
    - name: Performance regression check
      run: |
        node scripts/performance/check-regression.js \
          --baseline reports/baseline-performance.json \
          --current reports/lighthouse-*.json
```

---

## Performance Testing Best Practices

### Test Data Management
1. **Realistic Data Volumes**: Use production-like data sizes
2. **Data Variety**: Include edge cases and different data patterns
3. **Data Cleanup**: Clean up test data after tests complete
4. **Data Privacy**: Use anonymized data for testing

### Test Environment
1. **Production-Like**: Mirror production environment as closely as possible
2. **Isolated**: Dedicated performance testing environment
3. **Monitored**: Comprehensive monitoring during tests
4. **Repeatable**: Consistent test conditions

### Result Analysis
1. **Baseline Comparison**: Compare against established baselines
2. **Trend Analysis**: Monitor performance trends over time
3. **Root Cause Analysis**: Investigate performance degradations
4. **Actionable Insights**: Provide clear recommendations

### Continuous Improvement
1. **Regular Testing**: Integrate into CI/CD pipeline
2. **Performance Budgets**: Set and enforce performance limits
3. **Optimization Cycles**: Regular performance optimization
4. **Team Education**: Train team on performance best practices

---

*Last Updated: December 2024*  
*Performance testing guide continuously updated with new tools and best practices*