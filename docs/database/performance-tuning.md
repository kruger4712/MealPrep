# Database Performance Tuning Guide

## Overview
This guide provides comprehensive strategies for optimizing MealPrep database performance across different database systems and deployment scenarios.

## SQL Server Performance Optimization

### Index Strategy

#### Primary Indexes
```sql
-- Recipe search optimization
CREATE NONCLUSTERED INDEX IX_Recipe_Search 
ON Recipes (Name, Cuisine, DifficultyLevel) 
INCLUDE (Description, PrepTimeMinutes, CookTimeMinutes, ServingSize);

-- Family member preference queries
CREATE NONCLUSTERED INDEX IX_FamilyMember_Preferences 
ON FamilyMembers (UserId, IsActive) 
INCLUDE (Name, DietaryRestrictions, SpiceToleranceLevel);

-- Menu planning queries
CREATE NONCLUSTERED INDEX IX_MenuPlanning_DateRange 
ON WeeklyMenus (UserId, StartDate, EndDate) 
INCLUDE (IsArchived, TotalCost);

-- AI suggestion caching
CREATE NONCLUSTERED INDEX IX_AISuggestions_Cache 
ON AISuggestionRequests (RequestHash, CreatedDate) 
INCLUDE (ResponseData, FamilyFitScore);
```

#### Composite Indexes for Complex Queries
```sql
-- Recipe filtering by multiple criteria
CREATE NONCLUSTERED INDEX IX_Recipe_Filter_Complex 
ON Recipes (Cuisine, DifficultyLevel, PrepTimeMinutes, IsAiGenerated) 
INCLUDE (Name, Description, EstimatedCost, AverageRating);

-- Ingredient search across recipes
CREATE NONCLUSTERED INDEX IX_RecipeIngredient_Search 
ON RecipeIngredients (IngredientName, IsOptional) 
INCLUDE (RecipeId, Quantity, Unit, PrepNotes);

-- Family dietary restrictions lookup
CREATE NONCLUSTERED INDEX IX_FamilyPreferences_Dietary 
ON FamilyMemberPreferences (FamilyMemberId, PreferenceType, PreferenceValue) 
WHERE PreferenceType IN ('Allergy', 'DietaryRestriction', 'Dislike');
```

### Query Optimization

#### Recipe Search Optimization
```sql
-- Optimized recipe search with full-text indexing
-- Enable full-text search on recipe content
CREATE FULLTEXT CATALOG MealPrepFullTextCatalog;

CREATE FULLTEXT INDEX ON Recipes(Name, Description, Instructions)
KEY INDEX PK_Recipes
ON MealPrepFullTextCatalog;

-- Optimized search query
SELECT TOP 20 
    r.Id, r.Name, r.Description, r.Cuisine, r.DifficultyLevel,
    r.PrepTimeMinutes, r.CookTimeMinutes, r.AverageRating,
    RANK() OVER (ORDER BY r.AverageRating DESC, r.CreatedDate DESC) as SearchRank
FROM Recipes r
WHERE CONTAINS(r.Name, @SearchTerm)
   OR CONTAINS(r.Description, @SearchTerm)
   AND r.IsActive = 1
   AND (@Cuisine IS NULL OR r.Cuisine = @Cuisine)
   AND (@MaxPrepTime IS NULL OR r.PrepTimeMinutes <= @MaxPrepTime)
ORDER BY SearchRank;
```

#### Family Preference Queries
```sql
-- Optimized family preference aggregation
WITH FamilyAllergies AS (
    SELECT DISTINCT fmp.PreferenceValue as AllergyIngredient
    FROM FamilyMemberPreferences fmp
    INNER JOIN FamilyMembers fm ON fmp.FamilyMemberId = fm.Id
    WHERE fm.UserId = @UserId 
      AND fm.IsActive = 1
      AND fmp.PreferenceType = 'Allergy'
),
FamilyDislikes AS (
    SELECT fmp.PreferenceValue as DislikedIngredient,
           COUNT(*) as DislikeCount
    FROM FamilyMemberPreferences fmp
    INNER JOIN FamilyMembers fm ON fmp.FamilyMemberId = fm.Id
    WHERE fm.UserId = @UserId 
      AND fm.IsActive = 1
      AND fmp.PreferenceType = 'Dislike'
    GROUP BY fmp.PreferenceValue
)
SELECT 
    r.Id, r.Name, r.Cuisine, r.FamilyFitScore,
    CASE 
        WHEN EXISTS (
            SELECT 1 FROM RecipeIngredients ri
            INNER JOIN FamilyAllergies fa ON ri.IngredientName = fa.AllergyIngredient
            WHERE ri.RecipeId = r.Id
        ) THEN 0  -- Recipe contains allergens
        ELSE r.FamilyFitScore
    END as AdjustedFitScore
FROM Recipes r
WHERE NOT EXISTS (
    SELECT 1 FROM RecipeIngredients ri
    INNER JOIN FamilyAllergies fa ON ri.IngredientName = fa.AllergyIngredient
    WHERE ri.RecipeId = r.Id
)
ORDER BY AdjustedFitScore DESC;
```

### Database Maintenance

#### Statistics Updates
```sql
-- Automated statistics update job
-- Create maintenance plan for statistics updates
UPDATE STATISTICS Recipes WITH FULLSCAN;
UPDATE STATISTICS FamilyMembers WITH FULLSCAN;
UPDATE STATISTICS RecipeIngredients WITH FULLSCAN;
UPDATE STATISTICS WeeklyMenus WITH FULLSCAN;

-- Query to identify statistics needing updates
SELECT 
    s.name AS StatName,
    sp.last_updated,
    sp.rows AS RowCount,
    sp.rows_sampled AS SampleRows,
    sp.modification_counter AS ModCount
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE s.object_id = OBJECT_ID('Recipes')
  AND sp.modification_counter > 1000;
```

#### Index Maintenance
```sql
-- Index fragmentation analysis and maintenance
SELECT 
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    CASE 
        WHEN ips.avg_fragmentation_in_percent > 30 THEN 'REBUILD'
        WHEN ips.avg_fragmentation_in_percent > 10 THEN 'REORGANIZE'
        ELSE 'NO ACTION'
    END AS RecommendedAction
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 10
  AND ips.page_count > 1000
ORDER BY ips.avg_fragmentation_in_percent DESC;

-- Automated index maintenance script
DECLARE @sql NVARCHAR(1000);
DECLARE @fragmentation FLOAT;
DECLARE @indexname SYSNAME;

DECLARE index_cursor CURSOR FOR
SELECT i.name, ips.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 10;

OPEN index_cursor;
FETCH NEXT FROM index_cursor INTO @indexname, @fragmentation;

WHILE @@FETCH_STATUS = 0
BEGIN
    IF @fragmentation > 30
        SET @sql = 'ALTER INDEX ' + @indexname + ' ON Recipes REBUILD ONLINE = ON;'
    ELSE
        SET @sql = 'ALTER INDEX ' + @indexname + ' ON Recipes REORGANIZE;'
    
    EXEC sp_executesql @sql;
    FETCH NEXT FROM index_cursor INTO @indexname, @fragmentation;
END;

CLOSE index_cursor;
DEALLOCATE index_cursor;
```

## MySQL Performance Optimization

### Index Optimization for MySQL
```sql
-- Recipe search indexes
CREATE INDEX idx_recipe_search ON recipes (name, cuisine, difficulty_level, prep_time_minutes);
CREATE INDEX idx_recipe_fulltext ON recipes (name, description, instructions) USING FULLTEXT;

-- Family member preference indexes
CREATE INDEX idx_family_active ON family_members (user_id, is_active);
CREATE INDEX idx_family_preferences ON family_member_preferences (family_member_id, preference_type);

-- Composite index for complex queries
CREATE INDEX idx_recipe_filter ON recipes (cuisine, difficulty_level, prep_time_minutes, is_ai_generated)
  COVERING (name, description, estimated_cost, average_rating);
```

### Query Cache Configuration
```sql
-- MySQL query cache settings (MySQL 5.7 and earlier)
SET GLOBAL query_cache_type = ON;
SET GLOBAL query_cache_size = 268435456; -- 256MB
SET GLOBAL query_cache_limit = 1048576;  -- 1MB per query

-- For MySQL 8.0+, use prepared statements and optimize
PREPARE recipe_search FROM 
'SELECT id, name, description, cuisine, prep_time_minutes
FROM recipes 
WHERE name LIKE CONCAT("%", ?, "%") 
  AND cuisine = IFNULL(?, cuisine)
  AND prep_time_minutes <= IFNULL(?, prep_time_minutes)
ORDER BY average_rating DESC
LIMIT 20';
```

### InnoDB Optimization
```sql
-- InnoDB buffer pool optimization
SET GLOBAL innodb_buffer_pool_size = 2147483648; -- 2GB for 4GB system
SET GLOBAL innodb_buffer_pool_instances = 4;

-- InnoDB log file optimization
SET GLOBAL innodb_log_file_size = 268435456; -- 256MB
SET GLOBAL innodb_log_buffer_size = 16777216; -- 16MB

-- InnoDB thread concurrency
SET GLOBAL innodb_thread_concurrency = 8; -- Match CPU cores
```

## Entity Framework Performance

### DbContext Optimization
```csharp
public class MealPrepDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder
            .UseSqlServer(connectionString, options =>
            {
                options.CommandTimeout(30);
                options.EnableRetryOnFailure(3);
            })
            .EnableSensitiveDataLogging(false)
            .EnableServiceProviderCaching()
            .ConfigureWarnings(warnings =>
                warnings.Ignore(CoreEventId.RowLimitingOperationWithoutOrderByWarning));
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Optimize recipe queries
        modelBuilder.Entity<Recipe>()
            .HasIndex(r => new { r.Cuisine, r.DifficultyLevel, r.PrepTimeMinutes })
            .HasDatabaseName("IX_Recipe_Filter_Performance");

        // Optimize family member lookups
        modelBuilder.Entity<FamilyMember>()
            .HasIndex(fm => new { fm.UserId, fm.IsActive })
            .HasDatabaseName("IX_FamilyMember_Active");
    }
}
```

### Query Optimization Patterns
```csharp
public class RecipeRepository : IRecipeRepository
{
    private readonly MealPrepDbContext _context;

    // Optimized recipe search with projections
    public async Task<IEnumerable<RecipeSearchDto>> SearchRecipesAsync(
        RecipeSearchCriteria criteria)
    {
        return await _context.Recipes
            .Where(r => r.IsActive)
            .Where(r => criteria.Cuisine == null || r.Cuisine == criteria.Cuisine)
            .Where(r => criteria.MaxPrepTime == null || r.PrepTimeMinutes <= criteria.MaxPrepTime)
            .Where(r => criteria.SearchTerm == null || 
                       EF.Functions.Contains(r.Name, criteria.SearchTerm))
            .Select(r => new RecipeSearchDto
            {
                Id = r.Id,
                Name = r.Name,
                Description = r.Description,
                Cuisine = r.Cuisine,
                PrepTimeMinutes = r.PrepTimeMinutes,
                AverageRating = r.AverageRating
            })
            .OrderByDescending(r => r.AverageRating)
            .Take(20)
            .AsNoTracking()
            .ToListAsync();
    }

    // Optimized family preferences with split queries
    public async Task<FamilyPreferencesDto> GetFamilyPreferencesAsync(int userId)
    {
        var family = await _context.FamilyMembers
            .Where(fm => fm.UserId == userId && fm.IsActive)
            .AsSplitQuery()
            .Include(fm => fm.Preferences.Where(p => p.PreferenceType == "Allergy"))
            .Include(fm => fm.Preferences.Where(p => p.PreferenceType == "Dislike"))
            .AsNoTracking()
            .ToListAsync();

        return MapToDto(family);
    }

    // Bulk operations for better performance
    public async Task<int> UpdateRecipeRatingsAsync(List<RecipeRating> ratings)
    {
        _context.RecipeRatings.AddRange(ratings);
        
        // Update average ratings in bulk
        var recipeIds = ratings.Select(r => r.RecipeId).Distinct();
        var recipes = await _context.Recipes
            .Where(r => recipeIds.Contains(r.Id))
            .ToListAsync();

        foreach (var recipe in recipes)
        {
            recipe.AverageRating = await _context.RecipeRatings
                .Where(rr => rr.RecipeId == recipe.Id)
                .AverageAsync(rr => rr.Rating);
        }

        return await _context.SaveChangesAsync();
    }
}
```

## Caching Strategies

### Redis Caching Implementation
```csharp
public class CachedRecipeService : IRecipeService
{
    private readonly IRecipeService _recipeService;
    private readonly IDistributedCache _cache;
    private readonly ILogger<CachedRecipeService> _logger;

    public async Task<IEnumerable<Recipe>> SearchRecipesAsync(RecipeSearchCriteria criteria)
    {
        var cacheKey = $"recipes:search:{criteria.GetHashCode()}";
        var cachedResult = await _cache.GetStringAsync(cacheKey);

        if (cachedResult != null)
        {
            _logger.LogInformation("Cache hit for recipe search: {CacheKey}", cacheKey);
            return JsonSerializer.Deserialize<IEnumerable<Recipe>>(cachedResult);
        }

        var recipes = await _recipeService.SearchRecipesAsync(criteria);
        
        var cacheOptions = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15),
            SlidingExpiration = TimeSpan.FromMinutes(5)
        };

        var serializedRecipes = JsonSerializer.Serialize(recipes);
        await _cache.SetStringAsync(cacheKey, serializedRecipes, cacheOptions);

        _logger.LogInformation("Cached recipe search results: {CacheKey}", cacheKey);
        return recipes;
    }
}

// Redis configuration
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = configuration.GetConnectionString("Redis");
    options.InstanceName = "MealPrep";
});
```

### Memory Caching for Hot Data
```csharp
public class MemoryCachedFamilyService : IFamilyService
{
    private readonly IFamilyService _familyService;
    private readonly IMemoryCache _memoryCache;
    private readonly MemoryCacheEntryOptions _defaultOptions;

    public MemoryCachedFamilyService(IFamilyService familyService, IMemoryCache memoryCache)
    {
        _familyService = familyService;
        _memoryCache = memoryCache;
        _defaultOptions = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
            SlidingExpiration = TimeSpan.FromMinutes(2),
            Priority = CacheItemPriority.High
        };
    }

    public async Task<IEnumerable<FamilyMember>> GetActiveFamilyMembersAsync(int userId)
    {
        var cacheKey = $"family:active:{userId}";
        
        if (_memoryCache.TryGetValue(cacheKey, out IEnumerable<FamilyMember> cachedMembers))
        {
            return cachedMembers;
        }

        var members = await _familyService.GetActiveFamilyMembersAsync(userId);
        _memoryCache.Set(cacheKey, members, _defaultOptions);
        
        return members;
    }
}
```

## Connection Pooling

### SQL Server Connection Pooling
```csharp
// Connection string optimization
services.AddDbContext<MealPrepDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.CommandTimeout(30);
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorNumbersToAdd: null);
    }));

// Connection string configuration
"DefaultConnection": "Server=localhost;Database=MealPrepDB;Trusted_Connection=true;MultipleActiveResultSets=true;Connection Timeout=30;Min Pool Size=5;Max Pool Size=100;Pooling=true;"
```

### Custom Connection Management
```csharp
public class DatabaseConnectionManager
{
    private readonly string _connectionString;
    private readonly SemaphoreSlim _connectionSemaphore;

    public DatabaseConnectionManager(string connectionString, int maxConnections = 50)
    {
        _connectionString = connectionString;
        _connectionSemaphore = new SemaphoreSlim(maxConnections, maxConnections);
    }

    public async Task<T> ExecuteWithConnectionAsync<T>(Func<IDbConnection, Task<T>> operation)
    {
        await _connectionSemaphore.WaitAsync();
        try
        {
            using var connection = new SqlConnection(_connectionString);
            await connection.OpenAsync();
            return await operation(connection);
        }
        finally
        {
            _connectionSemaphore.Release();
        }
    }
}
```

## Monitoring and Diagnostics

### SQL Server Performance Monitoring
```sql
-- Query to identify slow queries
SELECT TOP 10
    qs.sql_handle,
    qs.execution_count,
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time,
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.total_physical_reads / qs.execution_count AS avg_physical_reads,
    SUBSTRING(qt.text, qs.statement_start_offset/2+1,
        (CASE WHEN qs.statement_end_offset = -1
        THEN LEN(CONVERT(nvarchar(max), qt.text)) * 2
        ELSE qs.statement_end_offset end - qs.statement_start_offset)/2) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
WHERE qt.text LIKE '%Recipe%' OR qt.text LIKE '%FamilyMember%'
ORDER BY avg_elapsed_time DESC;

-- Index usage statistics
SELECT 
    i.name AS IndexName,
    ius.user_seeks,
    ius.user_scans,
    ius.user_lookups,
    ius.user_updates,
    ius.last_user_seek,
    ius.last_user_scan
FROM sys.dm_db_index_usage_stats ius
INNER JOIN sys.indexes i ON ius.object_id = i.object_id AND ius.index_id = i.index_id
WHERE ius.database_id = DB_ID('MealPrepDB')
  AND i.name IS NOT NULL
ORDER BY ius.user_seeks + ius.user_scans + ius.user_lookups DESC;
```

### Application Performance Monitoring
```csharp
public class DatabasePerformanceMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<DatabasePerformanceMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            
            if (stopwatch.ElapsedMilliseconds > 1000) // Log slow requests
            {
                _logger.LogWarning("Slow database operation detected: {Path} took {ElapsedMs}ms",
                    context.Request.Path, stopwatch.ElapsedMilliseconds);
            }
        }
    }
}

// Database query logging
public class QueryLogger : IDbCommandInterceptor
{
    private readonly ILogger<QueryLogger> _logger;

    public override ValueTask<InterceptionResult<int>> NonQueryExecutingAsync(
        DbCommand command, CommandEventData eventData, 
        InterceptionResult<int> result, CancellationToken cancellationToken = default)
    {
        LogCommand(command, "NonQuery");
        return base.NonQueryExecutingAsync(command, eventData, result, cancellationToken);
    }

    private void LogCommand(DbCommand command, string operationType)
    {
        if (command.CommandTimeout > 5000) // Log slow queries
        {
            _logger.LogWarning("Slow {OperationType} detected: {CommandText} (Timeout: {Timeout}ms)",
                operationType, command.CommandText, command.CommandTimeout);
        }
    }
}
```

## Performance Testing

### Load Testing Scenarios
```csharp
public class DatabaseLoadTests
{
    [Test]
    public async Task Recipe_Search_Load_Test()
    {
        var tasks = new List<Task>();
        var concurrent_users = 100;
        var requests_per_user = 10;

        for (int i = 0; i < concurrent_users; i++)
        {
            tasks.Add(Task.Run(async () =>
            {
                for (int j = 0; j < requests_per_user; j++)
                {
                    var criteria = new RecipeSearchCriteria
                    {
                        SearchTerm = $"test{j}",
                        Cuisine = j % 2 == 0 ? "Italian" : null,
                        MaxPrepTime = j % 3 == 0 ? 30 : null
                    };
                    
                    var stopwatch = Stopwatch.StartNew();
                    await _recipeService.SearchRecipesAsync(criteria);
                    stopwatch.Stop();
                    
                    Assert.That(stopwatch.ElapsedMilliseconds, Is.LessThan(1000));
                }
            }));
        }

        await Task.WhenAll(tasks);
    }
}
```

## Performance Baselines and SLAs

### Target Performance Metrics
| Operation | Target Response Time | Acceptable Response Time | Error Rate |
|-----------|---------------------|-------------------------|------------|
| Recipe Search | < 200ms | < 500ms | < 0.1% |
| Family Member Lookup | < 100ms | < 300ms | < 0.01% |
| AI Suggestion Generation | < 10s | < 30s | < 2% |
| Menu Creation | < 500ms | < 1s | < 0.5% |
| Shopping List Generation | < 300ms | < 800ms | < 0.2% |
| User Authentication | < 150ms | < 400ms | < 0.01% |

### Database Performance Benchmarks
```sql
-- Baseline performance test queries
-- Recipe search benchmark
SELECT COUNT(*) FROM (
    SELECT TOP 1000 r.Id, r.Name, r.Description
    FROM Recipes r 
    WHERE r.Name LIKE '%chicken%' 
      AND r.Cuisine = 'Italian'
      AND r.PrepTimeMinutes <= 30
    ORDER BY r.AverageRating DESC
) AS benchmark_result;

-- Family preference lookup benchmark
SELECT COUNT(*) FROM (
    SELECT fm.Id, fm.Name, STRING_AGG(fmp.PreferenceValue, ', ') as Preferences
    FROM FamilyMembers fm
    LEFT JOIN FamilyMemberPreferences fmp ON fm.Id = fmp.FamilyMemberId
    WHERE fm.UserId IN (SELECT TOP 100 Id FROM Users ORDER BY CreatedDate DESC)
    GROUP BY fm.Id, fm.Name
) AS family_benchmark;

-- Menu planning benchmark
SELECT COUNT(*) FROM (
    SELECT wm.Id, wm.Name, COUNT(mm.Id) as MealCount
    FROM WeeklyMenus wm
    LEFT JOIN MenuMeals mm ON wm.Id = mm.WeeklyMenuId
    WHERE wm.StartDate >= DATEADD(month, -3, GETDATE())
    GROUP BY wm.Id, wm.Name
) AS menu_benchmark;
```

### Automated Performance Monitoring
```csharp
public class PerformanceMonitoringService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<PerformanceMonitoringService> _logger;
    private readonly PerformanceCounterTimer _timer;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await MonitorDatabasePerformance();
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }

    private async Task MonitorDatabasePerformance()
    {
        using var scope = _serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<MealPrepDbContext>();

        // Monitor slow queries
        var slowQueries = await dbContext.Database.SqlQueryRaw<SlowQueryResult>(@"
            SELECT TOP 5
                qt.text as QueryText,
                qs.total_elapsed_time / qs.execution_count as AvgElapsedTime,
                qs.execution_count as ExecutionCount
            FROM sys.dm_exec_query_stats qs
            CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
            WHERE qs.total_elapsed_time / qs.execution_count > 1000000  -- > 1 second avg
            ORDER BY qs.total_elapsed_time / qs.execution_count DESC
        ").ToListAsync();

        foreach (var query in slowQueries)
        {
            _logger.LogWarning("Slow query detected: {QueryText} (Avg: {AvgTime}ms, Count: {Count})",
                query.QueryText.Substring(0, Math.Min(100, query.QueryText.Length)),
                query.AvgElapsedTime / 1000, query.ExecutionCount);
        }

        // Monitor connection pool health
        var connectionPoolMetrics = await GetConnectionPoolMetrics(dbContext);
        if (connectionPoolMetrics.ActiveConnections > connectionPoolMetrics.MaxPoolSize * 0.8)
        {
            _logger.LogWarning("High connection pool usage: {Active}/{Max}",
                connectionPoolMetrics.ActiveConnections, connectionPoolMetrics.MaxPoolSize);
        }
    }
}

public class SlowQueryResult
{
    public string QueryText { get; set; }
    public long AvgElapsedTime { get; set; }
    public int ExecutionCount { get; set; }
}
```

### Performance Optimization Recommendations

#### Immediate Optimizations
1. **Enable Query Store** for SQL Server performance tracking
2. **Implement database connection pooling** with appropriate limits
3. **Add covering indexes** for most frequent query patterns
4. **Configure Redis caching** for frequently accessed data
5. **Enable compression** for large text fields

#### Progressive Enhancements
1. **Implement read replicas** for reporting and analytics queries
2. **Consider database sharding** for multi-tenant scenarios
3. **Implement intelligent query caching** based on user patterns
4. **Add database health monitoring** with automated alerting
5. **Optimize AI request batching** to reduce database round trips

#### Advanced Optimizations
1. **Implement CQRS pattern** for complex read scenarios
2. **Consider event sourcing** for audit and temporal queries
3. **Implement materialized views** for complex aggregations
4. **Add database partitioning** for historical data management
5. **Implement smart data archiving** strategies

## Best Practices Summary

### Index Design
1. **Create indexes for frequently queried columns**
2. **Use composite indexes for multi-column WHERE clauses**
3. **Include covering indexes for SELECT columns**
4. **Monitor and remove unused indexes**
5. **Regular index maintenance and statistics updates**

### Query Optimization
1. **Use parameterized queries to prevent SQL injection**
2. **Avoid SELECT * in production queries**
3. **Use appropriate JOIN types and order**
4. **Implement proper pagination for large result sets**
5. **Use EXISTS instead of IN for subqueries**

### Caching Strategy
1. **Cache frequently accessed, slowly changing data**
2. **Implement cache invalidation strategies**
3. **Use appropriate cache expiration times**
4. **Monitor cache hit ratios and adjust accordingly**
5. **Consider distributed caching for scale-out scenarios**

### Connection Management
1. **Use connection pooling appropriately**
2. **Implement connection timeouts**
3. **Monitor connection pool usage**
4. **Use async/await for database operations**
5. **Implement proper error handling and retry logic**

### Monitoring and Alerting
1. **Track key performance indicators continuously**
2. **Set up automated alerts for performance degradation**
3. **Monitor resource utilization patterns**
4. **Implement trending analysis for capacity planning**
5. **Regular performance baseline reviews and updates**

---

*Last Updated: December 2024*  
*Performance baselines and optimization strategies are continuously monitored and updated*