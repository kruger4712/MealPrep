# Troubleshooting Guide

This guide helps you solve common issues when developing, deploying, or using the MealPrep application.

## ?? Quick Reference

### Common Issues
- [Development Environment Setup](#development-environment)
- [API Integration Problems](#api-integration)
- [AI Service Issues](#ai-service-issues)
- [Database Connection Problems](#database-issues)
- [Frontend Build Issues](#frontend-issues)
- [Authentication Problems](#authentication-issues)
- [Performance Issues](#performance-issues)

---

## Development Environment

### .NET SDK Issues

**Problem**: `dotnet` command not found
```bash
# Solution: Install .NET 8 SDK
# Download from: https://dotnet.microsoft.com/download/dotnet/8.0

# Verify installation
dotnet --version
# Should show 8.0.x
```

**Problem**: Package restore fails
```bash
# Solution: Clear NuGet cache and restore
dotnet nuget locals all --clear
dotnet restore --force
```

### Node.js and npm Issues

**Problem**: `npm install` fails with permission errors
```bash
# Solution: Use correct Node.js version and fix permissions
# Install Node.js 18+ from nodejs.org

# Clear npm cache
npm cache clean --force

# Install with verbose logging
npm install --verbose
```

**Problem**: React app won't start
```bash
# Solution: Check port conflicts and dependencies
# Ensure port 3000 is available
netstat -an | findstr :3000

# Delete node_modules and reinstall
rm -rf node_modules package-lock.json
npm install
npm start
```

### Docker Issues

**Problem**: Docker commands fail
```bash
# Solution: Ensure Docker Desktop is running
# Check Docker status
docker version
docker-compose version

# Restart Docker Desktop if needed
```

**Problem**: Container build fails
```bash
# Solution: Clear Docker cache and rebuild
docker system prune -f
docker-compose build --no-cache
docker-compose up -d
```

---

## API Integration

### Connection Issues

**Problem**: API returns 500 Internal Server Error

Check `appsettings.json` configuration:
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MealPrepDb;..."
  },
  "GoogleCloud": {
    "ProjectId": "your-project-id",
    "ServiceAccountKeyPath": "path/to/service-account.json"
  }
}
```

**Solution**:
1. Verify database connection string
2. Check Google Cloud service account file exists
3. Review application logs for detailed error information

**Problem**: CORS errors in browser
```
// Error: Access to fetch at 'https://localhost:7000/api/recipes' 
// from origin 'http://localhost:3000' has been blocked by CORS policy
```

**Solution**: Update CORS configuration in `Program.cs`
```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend",
        policy =>
        {
            policy.WithOrigins("http://localhost:3000", "https://localhost:3001")
                  .AllowAnyHeader()
                  .AllowAnyMethod()
                  .AllowCredentials();
        });
});

app.UseCors("AllowFrontend");
```

### Authentication Issues

**Problem**: JWT token validation fails
```json
{
  "error": "Invalid token",
  "message": "The token is expired or malformed"
}
```

**Solutions**:
1. Check token expiration time
2. Verify JWT secret key configuration
3. Ensure token is properly formatted in Authorization header

```
// Correct header format
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## AI Service Issues

### Google Gemini AI Connection

**Problem**: AI service returns authentication errors
```json
{
  "error": "Unauthenticated", 
  "message": "Request had invalid authentication credentials"
}
```

**Solutions**:
1. Verify Google Cloud project ID is correct
2. Check service account JSON file exists and is valid
3. Ensure service account has necessary permissions

```bash
# Test service account authentication
gcloud auth activate-service-account --key-file=path/to/service-account.json
gcloud projects list
```

**Problem**: AI suggestions are poor quality or irrelevant

**Solutions**:
1. Review prompt engineering templates
2. Check family persona data completeness
3. Verify request parameters are properly formatted
4. Test with different prompt variations

**Problem**: AI service rate limiting
```json
{
  "error": "Rate limit exceeded",
  "message": "Too many requests to AI service"
}
```

**Solutions**:
1. Implement request throttling
2. Add caching for similar requests
3. Use fallback strategies when rate limited
4. Review Google Cloud quotas and billing

---

## Database Issues

### Connection Problems

**Problem**: Database connection timeout
```
Microsoft.Data.SqlClient.SqlException: Timeout expired
```

**Solutions**:
1. Check database server is running
2. Verify connection string is correct
3. Test network connectivity to database
4. Review database server performance

**Problem**: Migration errors
```bash
# Error: Unable to create an object of type 'MealPrepDbContext'
dotnet ef migrations add InitialCreate
```

**Solutions**:
```bash
# Ensure design-time factory is configured
# Or specify startup project
dotnet ef migrations add InitialCreate --startup-project MealPrep.API
dotnet ef database update --startup-project MealPrep.API
```

### Data Issues

**Problem**: Entity Framework queries are slow

**Solutions**:
1. Add appropriate database indexes
2. Use `Include()` for eager loading
3. Consider query optimization
4. Review database execution plans

```csharp
// Optimized query example
var recipes = await context.Recipes
    .Include(r => r.Ingredients)
    .Where(r => r.IsActive)
    .OrderBy(r => r.Name)
    .ToListAsync();
```

---

## Frontend Issues

### Build and Runtime Errors

**Problem**: TypeScript compilation errors
```
TS2307: Cannot find module '@mui/material' or its corresponding type declarations
```

**Solutions**:
```bash
# Install missing dependencies
npm install @mui/material @emotion/react @emotion/styled

# Install type definitions
npm install --save-dev @types/react @types/react-dom
```

**Problem**: React component not updating

**Solutions**:
1. Check React Query cache invalidation
2. Verify state management with `useState` hooks
3. Ensure proper dependency arrays in `useEffect`
4. Check for immutability issues in state updates

**Problem**: API calls failing from frontend
```
// Error: Network Error or Request failed
```

**Solutions**:
1. Verify API base URL in environment variables
2. Check CORS configuration
3. Test API endpoints directly with Postman
4. Review browser network tab for detailed errors

---

## Performance Issues

### API Performance

**Problem**: Slow API response times

**Solutions**:
1. Add database query optimization
2. Implement response caching
3. Use async/await properly
4. Add performance logging

```csharp
// Add response caching
[ResponseCache(Duration = 300)] // 5 minutes
public async Task<ActionResult<List<Recipe>>> GetRecipes()
{
    return await _recipeService.GetRecipesAsync();
}
```

### Frontend Performance

**Problem**: Slow page loads

**Solutions**:
1. Implement code splitting
2. Optimize bundle size
3. Use React.memo for expensive components
4. Implement virtual scrolling for large lists

```typescript
// React.memo example
export const RecipeCard = React.memo<RecipeCardProps>(({ recipe }) => {
  return (
    <Card>
      {/* Component content */}
    </Card>
  );
});
```

---

## User Issues

### Login and Account Problems

**Problem**: User cannot log in

**Solutions**:
1. Verify email and password are correct
2. Check account is activated
3. Test password reset functionality
4. Review authentication logs

**Problem**: Family member setup fails

**Solutions**:
1. Validate all required fields are filled
2. Check dietary restrictions format
3. Verify age ranges are appropriate
4. Test with simplified preferences first

### AI Meal Planning Issues

**Problem**: No meal suggestions generated

**Solutions**:
1. Ensure family members have preferences set
2. Check budget and time constraints are reasonable
3. Verify available ingredients list is not empty
4. Try broadening dietary restrictions temporarily

**Problem**: Suggestions don't match preferences

**Solutions**:
1. Review family member preference settings
2. Check for conflicting dietary restrictions
3. Verify spice tolerance levels are set correctly
4. Test with individual family members first

---

## Getting More Help

### Log Files
- **Backend Logs**: `logs/` directory or Application Insights
- **Frontend Logs**: Browser Developer Console
- **Database Logs**: Database server error logs

### Support Channels
1. **GitHub Issues**: Report bugs and request features
2. **GitHub Discussions**: Ask questions and get community help
3. **Documentation**: Check relevant guides in `docs/` folder
4. **Stack Overflow**: Tag questions with `mealprep-app`

### Escalation Process
1. Search existing issues and documentation
2. Try troubleshooting steps in this guide
3. Gather relevant log files and error messages
4. Create detailed issue report with reproduction steps
5. Include environment details (OS, versions, etc.)

---

**Note**: Keep this guide updated as new issues are discovered and resolved. Add solutions that work for multiple users to benefit the entire community.