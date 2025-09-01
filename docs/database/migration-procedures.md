# Database Migration Procedures

## Overview
Comprehensive guide for managing database schema changes in the MealPrep application using Entity Framework Core migrations with proper versioning, rollback procedures, and production deployment strategies.

## Migration Architecture

### Migration Strategy
```
Development ? Testing ? Staging ? Production
     ?           ?         ?          ?
  Generate ?  Validate ? Test ? Deploy with Rollback Plan
```

### Migration Types
1. **Schema Migrations**: Table structure changes, indexes, constraints
2. **Data Migrations**: Data transformations and seeding
3. **Hotfix Migrations**: Critical production fixes
4. **Rollback Migrations**: Reverting problematic changes

## Development Workflow

### Creating Migrations
```bash
# Generate a new migration
dotnet ef migrations add AddRecipeRatingsTable --context MealPrepDbContext

# Generate migration with custom output directory
dotnet ef migrations add AddUserPreferences --context MealPrepDbContext --output-dir Data/Migrations

# Generate migration for specific environment
dotnet ef migrations add AddAnalyticsTables --context MealPrepDbContext --configuration Release
```

### Migration Naming Conventions
```
Format: [YYYYMMDD]_[FeatureName]_[Description]
Examples:
- 20241201_UserManagement_AddEmailVerification
- 20241202_Recipes_AddNutritionalInfo
- 20241203_AI_AddPersonaData
- 20241204_Hotfix_FixUserIndexing
```

### Migration Code Structure
```csharp
// Migrations/20241201_UserManagement_AddEmailVerification.cs
public partial class AddEmailVerification : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Schema changes
        migrationBuilder.AddColumn<bool>(
            name: "EmailVerified",
            table: "Users",
            type: "bit",
            nullable: false,
            defaultValue: false);

        migrationBuilder.AddColumn<string>(
            name: "EmailVerificationToken",
            table: "Users",
            type: "nvarchar(256)",
            maxLength: 256,
            nullable: true);

        migrationBuilder.AddColumn<DateTime>(
            name: "EmailVerificationTokenExpiry",
            table: "Users",
            type: "datetime2",
            nullable: true);

        // Add index for performance
        migrationBuilder.CreateIndex(
            name: "IX_Users_EmailVerificationToken",
            table: "Users",
            column: "EmailVerificationToken",
            unique: true,
            filter: "[EmailVerificationToken] IS NOT NULL");

        // Data migration - mark existing users as verified
        migrationBuilder.Sql(@"
            UPDATE Users 
            SET EmailVerified = 1 
            WHERE CreatedAt < GETUTCDATE()
              AND EmailVerified = 0;
        ");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        // Rollback in reverse order
        migrationBuilder.DropIndex(
            name: "IX_Users_EmailVerificationToken",
            table: "Users");

        migrationBuilder.DropColumn(
            name: "EmailVerificationTokenExpiry",
            table: "Users");

        migrationBuilder.DropColumn(
            name: "EmailVerificationToken",
            table: "Users");

        migrationBuilder.DropColumn(
            name: "EmailVerified",
            table: "Users");
    }
}
```

## Data Migration Patterns

### Complex Data Transformations
```csharp
public partial class MigrateRecipeInstructionsToJson : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Step 1: Add new JSON column
        migrationBuilder.AddColumn<string>(
            name: "InstructionsJson",
            table: "Recipes",
            type: "nvarchar(max)",
            nullable: true);

        // Step 2: Migrate data using custom SQL
        migrationBuilder.Sql(@"
            UPDATE Recipes 
            SET InstructionsJson = CASE 
                WHEN Instructions IS NOT NULL 
                THEN '[""' + REPLACE(REPLACE(Instructions, CHAR(13)+CHAR(10), '"",""'), '"', '""""') + '""]'
                ELSE '[]'
            END
        ");

        // Step 3: Make new column non-nullable with default
        migrationBuilder.AlterColumn<string>(
            name: "InstructionsJson",
            table: "Recipes",
            type: "nvarchar(max)",
            nullable: false,
            defaultValue: "[]");

        // Step 4: Drop old column after data verification
        // Note: In production, this would be a separate migration after validation
        migrationBuilder.DropColumn(
            name: "Instructions",
            table: "Recipes");

        // Step 5: Rename new column
        migrationBuilder.RenameColumn(
            name: "InstructionsJson",
            table: "Recipes",
            newName: "Instructions");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        // Rollback transformation
        migrationBuilder.RenameColumn(
            name: "Instructions",
            table: "Recipes",
            newName: "InstructionsJson");

        migrationBuilder.AddColumn<string>(
            name: "Instructions",
            table: "Recipes",
            type: "nvarchar(max)",
            nullable: true);

        // Convert JSON back to plain text (data loss expected)
        migrationBuilder.Sql(@"
            UPDATE Recipes 
            SET Instructions = REPLACE(REPLACE(REPLACE(
                SUBSTRING(InstructionsJson, 2, LEN(InstructionsJson) - 2),
                '"",""', CHAR(13)+CHAR(10)), 
                '""""', '"'),
                '""', '')
        ");

        migrationBuilder.DropColumn(
            name: "InstructionsJson",
            table: "Recipes");
    }
}
```

### Large Data Migrations
```csharp
public partial class MigrateUserPreferencesToSeparateTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Create new table first
        migrationBuilder.CreateTable(
            name: "UserPreferences",
            columns: table => new
            {
                Id = table.Column<Guid>(type: "uniqueidentifier", nullable: false),
                UserId = table.Column<Guid>(type: "uniqueidentifier", nullable: false),
                PreferenceKey = table.Column<string>(type: "nvarchar(100)", maxLength: 100, nullable: false),
                PreferenceValue = table.Column<string>(type: "nvarchar(max)", nullable: true),
                CreatedAt = table.Column<DateTime>(type: "datetime2", nullable: false),
                UpdatedAt = table.Column<DateTime>(type: "datetime2", nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_UserPreferences", x => x.Id);
                table.ForeignKey(
                    name: "FK_UserPreferences_Users_UserId",
                    column: x => x.UserId,
                    principalTable: "Users",
                    principalColumn: "Id",
                    onDelete: ReferentialAction.Cascade);
                table.UniqueConstraint("UC_UserPreferences_UserId_Key", x => new { x.UserId, x.PreferenceKey });
            });

        // Create indexes
        migrationBuilder.CreateIndex(
            name: "IX_UserPreferences_UserId",
            table: "UserPreferences",
            column: "UserId");

        // Migrate data in batches to avoid locks
        migrationBuilder.Sql(@"
            INSERT INTO UserPreferences (Id, UserId, PreferenceKey, PreferenceValue, CreatedAt, UpdatedAt)
            SELECT 
                NEWID(),
                Id,
                'DefaultMealBudget',
                CAST(DefaultMealBudget AS NVARCHAR(50)),
                GETUTCDATE(),
                GETUTCDATE()
            FROM Users 
            WHERE DefaultMealBudget IS NOT NULL;
        ");

        // Add more preference migrations...
        migrationBuilder.Sql(@"
            INSERT INTO UserPreferences (Id, UserId, PreferenceKey, PreferenceValue, CreatedAt, UpdatedAt)
            SELECT 
                NEWID(),
                Id,
                'PreferredCookingTime',
                CAST(PreferredCookingTime AS NVARCHAR(50)),
                GETUTCDATE(),
                GETUTCDATE()
            FROM Users 
            WHERE PreferredCookingTime IS NOT NULL;
        ");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        // Restore data to original columns (add them back first)
        migrationBuilder.AddColumn<decimal>(
            name: "DefaultMealBudget",
            table: "Users",
            type: "decimal(8,2)",
            nullable: true);

        migrationBuilder.AddColumn<int>(
            name: "PreferredCookingTime",
            table: "Users",
            type: "int",
            nullable: true);

        // Migrate data back
        migrationBuilder.Sql(@"
            UPDATE u
            SET DefaultMealBudget = CAST(up.PreferenceValue AS DECIMAL(8,2))
            FROM Users u
            INNER JOIN UserPreferences up ON u.Id = up.UserId
            WHERE up.PreferenceKey = 'DefaultMealBudget';
        ");

        migrationBuilder.Sql(@"
            UPDATE u
            SET PreferredCookingTime = CAST(up.PreferenceValue AS INT)
            FROM Users u
            INNER JOIN UserPreferences up ON u.Id = up.UserId
            WHERE up.PreferenceKey = 'PreferredCookingTime';
        ");

        // Drop the new table
        migrationBuilder.DropTable(name: "UserPreferences");
    }
}
```

## Testing Migrations

### Local Testing
```bash
# Apply migrations to local database
dotnet ef database update --context MealPrepDbContext

# Test specific migration
dotnet ef database update [MigrationName] --context MealPrepDbContext

# Test rollback
dotnet ef database update [PreviousMigrationName] --context MealPrepDbContext

# Generate SQL script for review
dotnet ef migrations script --context MealPrepDbContext --output migration.sql
```

### Automated Testing
```csharp
// Tests/Integration/MigrationTests.cs
[TestClass]
public class MigrationTests
{
    [TestMethod]
    public async Task Migration_AddEmailVerification_ShouldApplySuccessfully()
    {
        // Arrange
        using var context = CreateTestContext();
        await context.Database.EnsureCreatedAsync();
        
        // Add test data before migration
        var user = new User 
        { 
            Email = "test@example.com", 
            PasswordHash = "hash",
            CreatedAt = DateTime.UtcNow.AddDays(-1)
        };
        context.Users.Add(user);
        await context.SaveChangesAsync();

        // Act - Apply migration
        await context.Database.MigrateAsync();

        // Assert
        var migratedUser = await context.Users.FirstAsync(u => u.Email == "test@example.com");
        Assert.IsTrue(migratedUser.EmailVerified); // Should be marked as verified
        Assert.IsNull(migratedUser.EmailVerificationToken);
    }

    [TestMethod]
    public async Task Migration_RollbackEmailVerification_ShouldRevertChanges()
    {
        // Arrange
        using var context = CreateTestContext();
        await context.Database.MigrateAsync(); // Apply all migrations
        
        // Act - Rollback to previous migration
        var targetMigration = "20241130_PreviousMigration";
        await context.Database.MigrateAsync(targetMigration);

        // Assert
        var hasEmailVerifiedColumn = await context.Database
            .ExecuteScalarAsync<int>(@"
                SELECT COUNT(*) 
                FROM INFORMATION_SCHEMA.COLUMNS 
                WHERE TABLE_NAME = 'Users' 
                AND COLUMN_NAME = 'EmailVerified'
            ");
        
        Assert.AreEqual(0, hasEmailVerifiedColumn);
    }

    [TestMethod]
    public async Task Migration_DataTransformation_ShouldPreserveDataIntegrity()
    {
        // Test complex data migrations maintain referential integrity
        using var context = CreateTestContext();
        
        // Setup test data before migration
        var recipe = new Recipe 
        { 
            Name = "Test Recipe",
            Instructions = "Step 1\nStep 2\nStep 3"
        };
        context.Recipes.Add(recipe);
        await context.SaveChangesAsync();

        // Apply migration
        await context.Database.MigrateAsync();

        // Verify data transformation
        var migratedRecipe = await context.Recipes
            .FirstAsync(r => r.Name == "Test Recipe");
        
        var instructions = JsonSerializer.Deserialize<string[]>(migratedRecipe.Instructions);
        Assert.AreEqual(3, instructions.Length);
        Assert.AreEqual("Step 1", instructions[0]);
        Assert.AreEqual("Step 2", instructions[1]);
        Assert.AreEqual("Step 3", instructions[2]);
    }
}
```

## Production Deployment

### Pre-Deployment Checklist
```markdown
## Migration Deployment Checklist

### Pre-Deployment (1-2 days before)
- [ ] Review all pending migrations
- [ ] Generate and review SQL scripts
- [ ] Test migrations on staging environment
- [ ] Verify rollback procedures
- [ ] Estimate migration time and potential downtime
- [ ] Prepare communication for users if downtime expected
- [ ] Backup production database
- [ ] Coordinate with DevOps team

### Deployment Day
- [ ] Confirm backup is recent and verified
- [ ] Put application in maintenance mode if required
- [ ] Apply migrations in order
- [ ] Verify data integrity after each migration
- [ ] Run post-migration validation tests
- [ ] Monitor application performance
- [ ] Remove maintenance mode
- [ ] Monitor for issues for 2-4 hours

### Post-Deployment
- [ ] Document any issues encountered
- [ ] Update runbooks if procedures changed
- [ ] Clean up old backup files (after retention period)
- [ ] Update team on deployment status
```

### Production Migration Scripts
```bash
#!/bin/bash
# deploy-migrations.sh

set -e

echo "Starting MealPrep Database Migration Deployment"
echo "================================================"

# Configuration
DB_CONNECTION_STRING=$1
BACKUP_LOCATION="/backups/mealprep/$(date +%Y%m%d_%H%M%S)"
LOG_FILE="/logs/migration_$(date +%Y%m%d_%H%M%S).log"

# Functions
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

backup_database() {
    log "Creating database backup..."
    
    # SQL Server backup
    sqlcmd -S $DB_SERVER -U $DB_USER -P $DB_PASSWORD -Q "
        BACKUP DATABASE [MealPrepProd] 
        TO DISK = N'$BACKUP_LOCATION.bak' 
        WITH FORMAT, 
        INIT, 
        NAME = N'MealPrepProd-Full Database Backup', 
        SKIP, 
        NOREWIND, 
        NOUNLOAD, 
        STATS = 10"
    
    log "Backup completed: $BACKUP_LOCATION.bak"
}

apply_migrations() {
    log "Applying database migrations..."
    
    # Generate migration script
    dotnet ef migrations script --idempotent --context MealPrepDbContext --output migration_script.sql
    
    # Apply migrations
    dotnet ef database update --context MealPrepDbContext --connection "$DB_CONNECTION_STRING"
    
    log "Migrations applied successfully"
}

verify_migration() {
    log "Verifying migration success..."
    
    # Run verification queries
    sqlcmd -S $DB_SERVER -U $DB_USER -P $DB_PASSWORD -d MealPrepProd -Q "
        SELECT 
            'Migration verification - User count: ' + CAST(COUNT(*) AS VARCHAR(10))
        FROM Users;
        
        SELECT 
            'Migration verification - Recipe count: ' + CAST(COUNT(*) AS VARCHAR(10))
        FROM Recipes;
    " | tee -a $LOG_FILE
    
    log "Migration verification completed"
}

# Main execution
log "=== Migration Deployment Started ==="

backup_database
apply_migrations
verify_migration

log "=== Migration Deployment Completed Successfully ==="
```

### Rollback Procedures
```csharp
// Services/MigrationRollbackService.cs
public class MigrationRollbackService
{
    private readonly MealPrepDbContext _context;
    private readonly ILogger<MigrationRollbackService> _logger;

    public async Task<bool> RollbackToMigrationAsync(string targetMigration)
    {
        try
        {
            _logger.LogWarning("Starting rollback to migration: {TargetMigration}", targetMigration);
            
            // Verify target migration exists
            var appliedMigrations = await _context.Database.GetAppliedMigrationsAsync();
            if (!appliedMigrations.Contains(targetMigration))
            {
                throw new InvalidOperationException($"Target migration {targetMigration} was not found in applied migrations");
            }

            // Create backup before rollback
            await CreateRollbackBackupAsync();

            // Perform rollback
            await _context.Database.MigrateAsync(targetMigration);

            // Verify rollback success
            var currentMigrations = await _context.Database.GetAppliedMigrationsAsync();
            var lastApplied = currentMigrations.LastOrDefault();
            
            if (lastApplied == targetMigration)
            {
                _logger.LogInformation("Rollback completed successfully to {TargetMigration}", targetMigration);
                return true;
            }
            else
            {
                _logger.LogError("Rollback verification failed. Expected {Expected}, got {Actual}", 
                    targetMigration, lastApplied);
                return false;
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Rollback failed for migration {TargetMigration}", targetMigration);
            throw;
        }
    }

    private async Task CreateRollbackBackupAsync()
    {
        var timestamp = DateTime.UtcNow.ToString("yyyyMMdd_HHmmss");
        var backupName = $"MealPrepProd_PreRollback_{timestamp}";
        
        await _context.Database.ExecuteSqlRawAsync($@"
            BACKUP DATABASE [MealPrepProd] 
            TO DISK = N'/backups/rollback/{backupName}.bak' 
            WITH FORMAT, INIT, NAME = N'{backupName}', SKIP, NOREWIND, NOUNLOAD, STATS = 10
        ");
        
        _logger.LogInformation("Rollback backup created: {BackupName}", backupName);
    }
}
```

## Monitoring and Maintenance

### Migration Monitoring
```csharp
// Services/MigrationHealthService.cs
public class MigrationHealthService
{
    public async Task<MigrationHealthStatus> CheckMigrationHealthAsync()
    {
        var status = new MigrationHealthStatus();
        
        using var context = new MealPrepDbContext(_options);
        
        // Check for pending migrations
        var pendingMigrations = await context.Database.GetPendingMigrationsAsync();
        status.HasPendingMigrations = pendingMigrations.Any();
        status.PendingMigrations = pendingMigrations.ToList();
        
        // Check migration history
        var appliedMigrations = await context.Database.GetAppliedMigrationsAsync();
        status.AppliedMigrations = appliedMigrations.ToList();
        status.LastMigration = appliedMigrations.LastOrDefault();
        
        // Check database connectivity
        status.DatabaseConnected = await context.Database.CanConnectAsync();
        
        // Check for migration locks
        status.HasMigrationLocks = await CheckForMigrationLocksAsync(context);
        
        return status;
    }

    private async Task<bool> CheckForMigrationLocksAsync(MealPrepDbContext context)
    {
        var lockCount = await context.Database.ExecuteScalarAsync<int>(@"
            SELECT COUNT(*) 
            FROM sys.dm_exec_requests 
            WHERE blocking_session_id > 0 
            AND command LIKE '%MIGRATION%'
        ");
        
        return lockCount > 0;
    }
}

public class MigrationHealthStatus
{
    public bool DatabaseConnected { get; set; }
    public bool HasPendingMigrations { get; set; }
    public List<string> PendingMigrations { get; set; } = new();
    public List<string> AppliedMigrations { get; set; } = new();
    public string LastMigration { get; set; }
    public bool HasMigrationLocks { get; set; }
    public DateTime CheckedAt { get; set; } = DateTime.UtcNow;
}
```

### Performance Monitoring
```sql
-- Migration performance queries
-- Check migration execution times
SELECT 
    MigrationId,
    ProductVersion,
    -- Estimate execution time from logs if available
    'Check migration logs for timing' as ExecutionTime
FROM __EFMigrationsHistory 
ORDER BY MigrationId DESC;

-- Check for long-running queries during migrations
SELECT 
    r.session_id,
    r.start_time,
    r.command,
    r.sql_handle,
    r.status,
    r.blocking_session_id,
    DATEDIFF(SECOND, r.start_time, GETDATE()) as duration_seconds
FROM sys.dm_exec_requests r
WHERE r.command LIKE '%ALTER%' 
   OR r.command LIKE '%CREATE%'
   OR r.command LIKE '%DROP%'
ORDER BY r.start_time DESC;

-- Check index fragmentation after migrations
SELECT 
    OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    s.avg_fragmentation_in_percent,
    s.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.avg_fragmentation_in_percent > 10
AND s.page_count > 1000
ORDER BY s.avg_fragmentation_in_percent DESC;
```

This comprehensive migration procedures guide ensures safe, reliable database schema changes with proper testing, deployment, and rollback capabilities for the MealPrep application.

*This guide should be updated as migration patterns evolve and new database technologies are adopted.*