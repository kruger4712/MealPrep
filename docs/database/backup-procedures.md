# Database Backup Procedures

## Overview
Comprehensive database backup and recovery procedures for the MealPrep SQL Server database, covering automated backups, point-in-time recovery, disaster recovery scenarios, and best practices for data protection across development, staging, and production environments.

## Backup Strategy Architecture

### Backup Types and Schedule
```
???????????????????????????????????????????????????????????????
?                    Production Backup Strategy               ?
???????????????????????????????????????????????????????????????
? Full Backup:     Weekly (Sunday 2:00 AM)                   ?
? Differential:    Daily (2:00 AM, except Sunday)            ?
? Transaction Log: Every 15 minutes                          ?
? Retention:       Full (6 months), Diff (30 days), Log (7 days) ?
???????????????????????????????????????????????????????????????

???????????????????????????????????????????????????????????????
?                    Staging Backup Strategy                  ?
???????????????????????????????????????????????????????????????
? Full Backup:     Daily (3:00 AM)                          ?
? Transaction Log: Every 30 minutes                          ?
? Retention:       Full (30 days), Log (3 days)             ?
???????????????????????????????????????????????????????????????

???????????????????????????????????????????????????????????????
?                  Development Backup Strategy                ?
???????????????????????????????????????????????????????????????
? Full Backup:     Daily (4:00 AM)                          ?
? Retention:       7 days                                    ?
???????????????????????????????????????????????????????????????
```

## SQL Server Backup Implementation

### Automated Backup Scripts
```sql
-- Production Full Backup Script
-- File: backup-scripts/full-backup-production.sql
USE [master]
GO

DECLARE @BackupPath NVARCHAR(500)
DECLARE @BackupFileName NVARCHAR(500)
DECLARE @DatabaseName NVARCHAR(100) = 'MealPrepProd'
DECLARE @BackupDate NVARCHAR(20) = FORMAT(GETDATE(), 'yyyyMMdd_HHmmss')

-- Set backup path based on environment
SET @BackupPath = N'\\backup-server\MealPrep\Production\Full\'
SET @BackupFileName = @BackupPath + @DatabaseName + '_Full_' + @BackupDate + '.bak'

-- Perform full backup with compression and verification
BACKUP DATABASE [@DatabaseName] 
TO DISK = @BackupFileName
WITH 
    COMPRESSION,
    CHECKSUM,
    VERIFY_ONLY = 0,
    FORMAT,
    INIT,
    NAME = N'MealPrep Production Full Backup',
    DESCRIPTION = N'Full backup of MealPrep production database',
    STATS = 10,
    COPY_ONLY = 0;

-- Verify backup integrity
RESTORE VERIFYONLY 
FROM DISK = @BackupFileName
WITH CHECKSUM;

-- Log backup completion
DECLARE @Message NVARCHAR(1000)
SET @Message = 'Full backup completed successfully: ' + @BackupFileName
RAISERROR(@Message, 0, 1) WITH NOWAIT;

-- Cleanup old backups (older than retention period)
EXEC [dbo].[sp_CleanupOldBackups] 
    @BackupType = 'FULL',
    @RetentionDays = 180; -- 6 months
```

```sql
-- Differential Backup Script
-- File: backup-scripts/differential-backup-production.sql
USE [master]
GO

DECLARE @BackupPath NVARCHAR(500)
DECLARE @BackupFileName NVARCHAR(500)
DECLARE @DatabaseName NVARCHAR(100) = 'MealPrepProd'
DECLARE @BackupDate NVARCHAR(20) = FORMAT(GETDATE(), 'yyyyMMdd_HHmmss')

SET @BackupPath = N'\\backup-server\MealPrep\Production\Differential\'
SET @BackupFileName = @BackupPath + @DatabaseName + '_Diff_' + @BackupDate + '.bak'

-- Check if full backup exists (required for differential)
IF NOT EXISTS (
    SELECT 1 FROM msdb.dbo.backupset 
    WHERE database_name = @DatabaseName 
    AND type = 'D' 
    AND backup_finish_date > DATEADD(DAY, -7, GETDATE())
)
BEGIN
    RAISERROR('No recent full backup found. Differential backup requires a full backup.', 16, 1);
    RETURN;
END

-- Perform differential backup
BACKUP DATABASE [@DatabaseName] 
TO DISK = @BackupFileName
WITH 
    DIFFERENTIAL,
    COMPRESSION,
    CHECKSUM,
    FORMAT,
    INIT,
    NAME = N'MealPrep Production Differential Backup',
    DESCRIPTION = N'Differential backup of MealPrep production database',
    STATS = 10;

-- Verify backup
RESTORE VERIFYONLY 
FROM DISK = @BackupFileName
WITH CHECKSUM;

-- Log completion
DECLARE @Message NVARCHAR(1000)
SET @Message = 'Differential backup completed: ' + @BackupFileName
RAISERROR(@Message, 0, 1) WITH NOWAIT;

-- Cleanup old differential backups
EXEC [dbo].[sp_CleanupOldBackups] 
    @BackupType = 'DIFF',
    @RetentionDays = 30;
```

```sql
-- Transaction Log Backup Script
-- File: backup-scripts/log-backup-production.sql
USE [master]
GO

DECLARE @BackupPath NVARCHAR(500)
DECLARE @BackupFileName NVARCHAR(500)
DECLARE @DatabaseName NVARCHAR(100) = 'MealPrepProd'
DECLARE @BackupDate NVARCHAR(20) = FORMAT(GETDATE(), 'yyyyMMdd_HHmmss')

SET @BackupPath = N'\\backup-server\MealPrep\Production\Log\'
SET @BackupFileName = @BackupPath + @DatabaseName + '_Log_' + @BackupDate + '.trn'

-- Verify database is in FULL recovery model
IF (SELECT recovery_model FROM sys.databases WHERE name = @DatabaseName) != 1
BEGIN
    RAISERROR('Database must be in FULL recovery model for transaction log backups.', 16, 1);
    RETURN;
END

-- Perform transaction log backup
BACKUP LOG [@DatabaseName] 
TO DISK = @BackupFileName
WITH 
    COMPRESSION,
    CHECKSUM,
    FORMAT,
    INIT,
    NAME = N'MealPrep Production Transaction Log Backup',
    DESCRIPTION = N'Transaction log backup of MealPrep production database',
    STATS = 10;

-- Verify backup
RESTORE VERIFYONLY 
FROM DISK = @BackupFileName
WITH CHECKSUM;

-- Cleanup old log backups
EXEC [dbo].[sp_CleanupOldBackups] 
    @BackupType = 'LOG',
    @RetentionDays = 7;
```

### Backup Cleanup Stored Procedure
```sql
-- File: backup-scripts/sp_CleanupOldBackups.sql
CREATE OR ALTER PROCEDURE [dbo].[sp_CleanupOldBackups]
    @BackupType NVARCHAR(10), -- 'FULL', 'DIFF', 'LOG'
    @RetentionDays INT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX)
    DECLARE @BackupPath NVARCHAR(500)
    DECLARE @CutoffDate DATETIME = DATEADD(DAY, -@RetentionDays, GETDATE())
    
    -- Set backup path based on type
    SET @BackupPath = CASE @BackupType
        WHEN 'FULL' THEN '\\backup-server\MealPrep\Production\Full\'
        WHEN 'DIFF' THEN '\\backup-server\MealPrep\Production\Differential\'
        WHEN 'LOG' THEN '\\backup-server\MealPrep\Production\Log\'
        ELSE ''
    END
    
    IF @BackupPath = ''
    BEGIN
        RAISERROR('Invalid backup type specified', 16, 1);
        RETURN;
    END
    
    -- Create cursor for old backup files
    DECLARE @FileName NVARCHAR(500)
    DECLARE @FileDate DATETIME
    
    DECLARE backup_cursor CURSOR FOR
    SELECT 
        bf.physical_device_name,
        bs.backup_finish_date
    FROM msdb.dbo.backupset bs
    INNER JOIN msdb.dbo.backupmediafamily bf 
        ON bs.media_set_id = bf.media_set_id
    WHERE bs.database_name = 'MealPrepProd'
        AND bs.backup_finish_date < @CutoffDate
        AND bf.physical_device_name LIKE @BackupPath + '%'
        AND bs.type = CASE @BackupType 
            WHEN 'FULL' THEN 'D'
            WHEN 'DIFF' THEN 'I' 
            WHEN 'LOG' THEN 'L'
        END
    
    OPEN backup_cursor
    FETCH NEXT FROM backup_cursor INTO @FileName, @FileDate
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Delete physical backup file
        SET @SQL = 'EXEC xp_cmdshell ''DEL "' + @FileName + '"'''
        
        BEGIN TRY
            EXEC sp_executesql @SQL
            PRINT 'Deleted old backup file: ' + @FileName
        END TRY
        BEGIN CATCH
            PRINT 'Failed to delete backup file: ' + @FileName + ' - ' + ERROR_MESSAGE()
        END CATCH
        
        FETCH NEXT FROM backup_cursor INTO @FileName, @FileDate
    END
    
    CLOSE backup_cursor
    DEALLOCATE backup_cursor
    
    -- Clean up backup history from msdb
    EXEC msdb.dbo.sp_delete_backuphistory @oldest_date = @CutoffDate
    
    PRINT 'Cleanup completed for ' + @BackupType + ' backups older than ' + 
          CAST(@RetentionDays AS NVARCHAR(10)) + ' days'
END
```

## Recovery Procedures

### Point-in-Time Recovery
```sql
-- File: recovery-scripts/point-in-time-recovery.sql
-- Point-in-time recovery procedure for MealPrep database
USE [master]
GO

-- STEP 1: Prepare for recovery
-- Variables for recovery
DECLARE @DatabaseName NVARCHAR(100) = 'MealPrepProd'
DECLARE @RestoreDatabaseName NVARCHAR(100) = 'MealPrepProd_Recovery'
DECLARE @PointInTime DATETIME = '2024-12-15 14:30:00' -- Specify recovery point
DECLARE @DataFilePath NVARCHAR(500) = 'C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\Data\'
DECLARE @LogFilePath NVARCHAR(500) = 'C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\Data\'

-- STEP 2: Take tail-log backup (if database is still accessible)
DECLARE @TailLogBackup NVARCHAR(500) = @DataFilePath + 'TailLog_' + FORMAT(GETDATE(), 'yyyyMMdd_HHmmss') + '.trn'

-- Only if database is still online and accessible
IF DATABASEPROPERTYEX(@DatabaseName, 'Status') = 'ONLINE'
BEGIN
    BACKUP LOG [@DatabaseName] 
    TO DISK = @TailLogBackup
    WITH NO_RECOVERY, 
         NOFORMAT, 
         INIT,
         NAME = N'Tail Log Backup for Point-in-Time Recovery',
         DESCRIPTION = N'Tail log backup before point-in-time recovery';
END

-- STEP 3: Find the appropriate backup chain
WITH BackupChain AS (
    SELECT 
        bs.backup_set_id,
        bs.database_name,
        bs.backup_start_date,
        bs.backup_finish_date,
        bs.type,
        bf.physical_device_name,
        ROW_NUMBER() OVER (PARTITION BY bs.type ORDER BY bs.backup_finish_date DESC) as rn
    FROM msdb.dbo.backupset bs
    INNER JOIN msdb.dbo.backupmediafamily bf ON bs.media_set_id = bf.media_set_id
    WHERE bs.database_name = @DatabaseName
        AND bs.backup_finish_date <= @PointInTime
)
SELECT 
    type as BackupType,
    physical_device_name as BackupFile,
    backup_start_date,
    backup_finish_date
FROM BackupChain 
WHERE rn = 1
ORDER BY 
    CASE type 
        WHEN 'D' THEN 1  -- Full backup first
        WHEN 'I' THEN 2  -- Differential second
        WHEN 'L' THEN 3  -- Log backups last
    END;

-- STEP 4: Restore Full Backup (REPLACE WITH ACTUAL BACKUP FILE PATHS)
RESTORE DATABASE [@RestoreDatabaseName] 
FROM DISK = N'\\backup-server\MealPrep\Production\Full\MealPrepProd_Full_20241215_020000.bak'
WITH 
    REPLACE,
    NORECOVERY,
    MOVE 'MealPrepProd' TO @DataFilePath + @RestoreDatabaseName + '.mdf',
    MOVE 'MealPrepProd_Log' TO @LogFilePath + @RestoreDatabaseName + '_Log.ldf',
    CHECKSUM,
    STATS = 10;

-- STEP 5: Restore Most Recent Differential (if exists and before point in time)
RESTORE DATABASE [@RestoreDatabaseName] 
FROM DISK = N'\\backup-server\MealPrep\Production\Differential\MealPrepProd_Diff_20241215_020000.bak'
WITH 
    NORECOVERY,
    CHECKSUM,
    STATS = 10;

-- STEP 6: Restore Transaction Log Backups up to point in time
-- (This would need to be done for each log backup in sequence)
RESTORE LOG [@RestoreDatabaseName] 
FROM DISK = N'\\backup-server\MealPrep\Production\Log\MealPrepProd_Log_20241215_143000.trn'
WITH 
    NORECOVERY,
    STOPAT = @PointInTime,
    CHECKSUM,
    STATS = 10;

-- STEP 7: Restore tail log backup (if taken)
IF @TailLogBackup IS NOT NULL
BEGIN
    RESTORE LOG [@RestoreDatabaseName] 
    FROM DISK = @TailLogBackup
    WITH 
        NORECOVERY,
        STOPAT = @PointInTime,
        CHECKSUM;
END

-- STEP 8: Bring database online
RESTORE DATABASE [@RestoreDatabaseName] WITH RECOVERY;

-- STEP 9: Verify recovery
SELECT 
    name,
    state_desc,
    create_date,
    collation_name
FROM sys.databases 
WHERE name = @RestoreDatabaseName;

PRINT 'Point-in-time recovery completed successfully to: ' + CONVERT(NVARCHAR(25), @PointInTime, 121);
```

### Disaster Recovery Procedures
```sql
-- File: recovery-scripts/disaster-recovery.sql
-- Complete disaster recovery procedure for MealPrep
USE [master]
GO

-- Disaster Recovery Checklist Script
PRINT '========================================='
PRINT 'MEALPREP DISASTER RECOVERY PROCEDURE'
PRINT '========================================='
PRINT ''

-- Step 1: Assess the situation
PRINT 'STEP 1: DAMAGE ASSESSMENT'
PRINT '- Check database accessibility'
PRINT '- Verify backup availability'
PRINT '- Determine recovery requirements'
PRINT ''

-- Check if primary database exists and is accessible
IF EXISTS (SELECT 1 FROM sys.databases WHERE name = 'MealPrepProd')
BEGIN
    DECLARE @State NVARCHAR(60) = (SELECT state_desc FROM sys.databases WHERE name = 'MealPrepProd')
    PRINT 'Primary database status: ' + @State
    
    IF @State = 'ONLINE'
    BEGIN
        PRINT 'WARNING: Primary database is still online. Verify this is a disaster recovery scenario.'
    END
END
ELSE
BEGIN
    PRINT 'Primary database not found - proceeding with full recovery'
END

-- Step 2: Locate most recent backups
PRINT 'STEP 2: BACKUP VERIFICATION'

-- Find most recent full backup
DECLARE @MostRecentFull NVARCHAR(500)
DECLARE @FullBackupDate DATETIME

SELECT TOP 1 
    @MostRecentFull = bf.physical_device_name,
    @FullBackupDate = bs.backup_finish_date
FROM msdb.dbo.backupset bs
INNER JOIN msdb.dbo.backupmediafamily bf ON bs.media_set_id = bf.media_set_id
WHERE bs.database_name = 'MealPrepProd'
    AND bs.type = 'D'  -- Full backup
ORDER BY bs.backup_finish_date DESC

IF @MostRecentFull IS NOT NULL
BEGIN
    PRINT 'Most recent full backup: ' + @MostRecentFull
    PRINT 'Backup date: ' + CONVERT(NVARCHAR(25), @FullBackupDate, 121)
    
    -- Verify backup file exists and is readable
    RESTORE VERIFYONLY FROM DISK = @MostRecentFull
    PRINT 'Full backup verified successfully'
END
ELSE
BEGIN
    PRINT 'ERROR: No full backup found!'
    RETURN
END

-- Find most recent differential backup
DECLARE @MostRecentDiff NVARCHAR(500)
DECLARE @DiffBackupDate DATETIME

SELECT TOP 1 
    @MostRecentDiff = bf.physical_device_name,
    @DiffBackupDate = bs.backup_finish_date
FROM msdb.dbo.backupset bs
INNER JOIN msdb.dbo.backupmediafamily bf ON bs.media_set_id = bf.media_set_id
WHERE bs.database_name = 'MealPrepProd'
    AND bs.type = 'I'  -- Differential backup
    AND bs.backup_finish_date > @FullBackupDate
ORDER BY bs.backup_finish_date DESC

IF @MostRecentDiff IS NOT NULL
BEGIN
    PRINT 'Most recent differential backup: ' + @MostRecentDiff
    PRINT 'Differential backup date: ' + CONVERT(NVARCHAR(25), @DiffBackupDate, 121)
    
    RESTORE VERIFYONLY FROM DISK = @MostRecentDiff
    PRINT 'Differential backup verified successfully'
END

-- List recent transaction log backups
PRINT 'Recent transaction log backups:'
SELECT TOP 10
    bf.physical_device_name,
    bs.backup_finish_date,
    DATEDIFF(MINUTE, bs.backup_start_date, bs.backup_finish_date) as DurationMinutes
FROM msdb.dbo.backupset bs
INNER JOIN msdb.dbo.backupmediafamily bf ON bs.media_set_id = bf.media_set_id
WHERE bs.database_name = 'MealPrepProd'
    AND bs.type = 'L'  -- Log backup
    AND bs.backup_finish_date > ISNULL(@DiffBackupDate, @FullBackupDate)
ORDER BY bs.backup_finish_date DESC

PRINT ''
PRINT 'STEP 3: RECOVERY EXECUTION'
PRINT 'Ready to proceed with recovery. Execute the following steps:'
PRINT '1. Restore full backup with NORECOVERY'
PRINT '2. Restore differential backup with NORECOVERY (if available)'
PRINT '3. Restore transaction log backups in sequence with NORECOVERY'
PRINT '4. Restore final log backup with RECOVERY to bring database online'
PRINT '5. Verify database integrity and application connectivity'
```

## Backup Monitoring and Alerts

### Backup Health Check
```sql
-- File: monitoring/backup-health-check.sql
-- Comprehensive backup health monitoring for MealPrep
USE [master]
GO

CREATE OR ALTER PROCEDURE [dbo].[sp_BackupHealthCheck]
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @DatabaseName NVARCHAR(100) = 'MealPrepProd'
    DECLARE @AlertThreshold_Hours INT = 26  -- Alert if no backup in 26 hours
    DECLARE @CriticalThreshold_Hours INT = 50  -- Critical if no backup in 50 hours
    
    -- Check last backup times
    SELECT 
        'Full Backup' as BackupType,
        MAX(backup_finish_date) as LastBackupDate,
        DATEDIFF(HOUR, MAX(backup_finish_date), GETDATE()) as HoursSinceBackup,
        CASE 
            WHEN DATEDIFF(HOUR, MAX(backup_finish_date), GETDATE()) > @CriticalThreshold_Hours THEN 'CRITICAL'
            WHEN DATEDIFF(HOUR, MAX(backup_finish_date), GETDATE()) > @AlertThreshold_Hours THEN 'WARNING'
            ELSE 'OK'
        END as Status
    FROM msdb.dbo.backupset 
    WHERE database_name = @DatabaseName AND type = 'D'
    
    UNION ALL
    
    SELECT 
        'Differential Backup' as BackupType,
        MAX(backup_finish_date) as LastBackupDate,
        DATEDIFF(HOUR, MAX(backup_finish_date), GETDATE()) as HoursSinceBackup,
        CASE 
            WHEN DATEDIFF(HOUR, MAX(backup_finish_date), GETDATE()) > @AlertThreshold_Hours THEN 'WARNING'
            ELSE 'OK'
        END as Status
    FROM msdb.dbo.backupset 
    WHERE database_name = @DatabaseName AND type = 'I'
    
    UNION ALL
    
    SELECT 
        'Transaction Log Backup' as BackupType,
        MAX(backup_finish_date) as LastBackupDate,
        DATEDIFF(MINUTE, MAX(backup_finish_date), GETDATE()) as HoursSinceBackup,
        CASE 
            WHEN DATEDIFF(MINUTE, MAX(backup_finish_date), GETDATE()) > 30 THEN 'WARNING'
            WHEN DATEDIFF(MINUTE, MAX(backup_finish_date), GETDATE()) > 60 THEN 'CRITICAL'
            ELSE 'OK'
        END as Status
    FROM msdb.dbo.backupset 
    WHERE database_name = @DatabaseName AND type = 'L'
    
    -- Check for backup failures
    PRINT ''
    PRINT 'Recent Backup Failures:'
    
    SELECT TOP 10
        backup_start_date,
        type,
        CASE type 
            WHEN 'D' THEN 'Full'
            WHEN 'I' THEN 'Differential' 
            WHEN 'L' THEN 'Transaction Log'
        END as BackupType,
        'FAILED' as Status
    FROM msdb.dbo.backupset 
    WHERE database_name = @DatabaseName 
        AND backup_finish_date IS NULL
        AND backup_start_date > DATEADD(DAY, -7, GETDATE())
    ORDER BY backup_start_date DESC
    
    -- Check backup file sizes and growth
    PRINT ''
    PRINT 'Backup Size Trend (Last 7 Days):'
    
    SELECT 
        CONVERT(DATE, backup_finish_date) as BackupDate,
        CASE type 
            WHEN 'D' THEN 'Full'
            WHEN 'I' THEN 'Differential' 
            WHEN 'L' THEN 'Transaction Log'
        END as BackupType,
        AVG(backup_size / 1024 / 1024) as AvgSizeMB,
        MAX(backup_size / 1024 / 1024) as MaxSizeMB,
        COUNT(*) as BackupCount
    FROM msdb.dbo.backupset 
    WHERE database_name = @DatabaseName 
        AND backup_finish_date > DATEADD(DAY, -7, GETDATE())
    GROUP BY CONVERT(DATE, backup_finish_date), type
    ORDER BY BackupDate DESC, type
END
```

### PowerShell Backup Monitoring
```powershell
# File: monitoring/Backup-HealthMonitor.ps1
# PowerShell script for backup monitoring and alerting

param(
    [string]$ServerInstance = "localhost",
    [string]$DatabaseName = "MealPrepProd",
    [string]$EmailTo = "admin@mealprep.com",
    [string]$EmailFrom = "alerts@mealprep.com",
    [string]$SmtpServer = "smtp.company.com"
)

# Import SQL Server module
Import-Module SqlServer -ErrorAction SilentlyContinue

function Send-BackupAlert {
    param(
        [string]$Subject,
        [string]$Body,
        [string]$Priority = "Normal"
    )
    
    try {
        Send-MailMessage -To $EmailTo -From $EmailFrom -Subject $Subject -Body $Body -SmtpServer $SmtpServer -Priority $Priority
        Write-Host "Alert sent: $Subject"
    }
    catch {
        Write-Error "Failed to send email alert: $($_.Exception.Message)"
    }
}

function Test-BackupHealth {
    param(
        [string]$Server,
        [string]$Database
    )
    
    $query = @"
    SELECT 
        'Full' as BackupType,
        MAX(backup_finish_date) as LastBackupDate,
        DATEDIFF(HOUR, MAX(backup_finish_date), GETDATE()) as HoursSinceBackup
    FROM msdb.dbo.backupset 
    WHERE database_name = '$Database' AND type = 'D'
    
    UNION ALL
    
    SELECT 
        'Log' as BackupType,
        MAX(backup_finish_date) as LastBackupDate,
        DATEDIFF(MINUTE, MAX(backup_finish_date), GETDATE()) as HoursSinceBackup
    FROM msdb.dbo.backupset 
    WHERE database_name = '$Database' AND type = 'L'
"@
    
    try {
        $results = Invoke-Sqlcmd -ServerInstance $Server -Query $query
        return $results
    }
    catch {
        Write-Error "Failed to check backup health: $($_.Exception.Message)"
        return $null
    }
}

function Test-BackupFiles {
    param(
        [string]$BackupPath
    )
    
    $alerts = @()
    
    # Check if backup directory exists and is accessible
    if (-not (Test-Path $BackupPath)) {
        $alerts += "Backup directory not accessible: $BackupPath"
        return $alerts
    }
    
    # Check disk space on backup drive
    $drive = Split-Path $BackupPath -Qualifier
    $disk = Get-WmiObject -Class Win32_LogicalDisk -Filter "DeviceID='$drive'"
    
    if ($disk) {
        $freeSpaceGB = [math]::Round($disk.FreeSpace / 1GB, 2)
        $totalSpaceGB = [math]::Round($disk.Size / 1GB, 2)
        $freeSpacePercent = [math]::Round(($disk.FreeSpace / $disk.Size) * 100, 2)
        
        if ($freeSpacePercent -lt 20) {
            $alerts += "Low disk space on backup drive $drive : $freeSpaceGB GB free ($freeSpacePercent%)"
        }
        
        Write-Host "Backup drive $drive : $freeSpaceGB GB free of $totalSpaceGB GB ($freeSpacePercent%)"
    }
    
    return $alerts
}

# Main execution
Write-Host "Starting backup health check for $DatabaseName on $ServerInstance"

# Check backup health
$backupHealth = Test-BackupHealth -Server $ServerInstance -Database $DatabaseName

if ($backupHealth) {
    foreach ($backup in $backupHealth) {
        $backupType = $backup.BackupType
        $lastBackup = $backup.LastBackupDate
        $timeSince = $backup.HoursSinceBackup
        
        Write-Host "$backupType backup: Last completed $lastBackup ($timeSince hours ago)"
        
        # Check thresholds and send alerts
        switch ($backupType) {
            "Full" {
                if ($timeSince -gt 48) {
                    Send-BackupAlert -Subject "CRITICAL: Full backup overdue for $DatabaseName" -Body "Last full backup was $timeSince hours ago at $lastBackup" -Priority "High"
                }
                elseif ($timeSince -gt 26) {
                    Send-BackupAlert -Subject "WARNING: Full backup overdue for $DatabaseName" -Body "Last full backup was $timeSince hours ago at $lastBackup"
                }
            }
            "Log" {
                if ($timeSince -gt 60) {
                    Send-BackupAlert -Subject "CRITICAL: Transaction log backup overdue for $DatabaseName" -Body "Last log backup was $timeSince minutes ago at $lastBackup" -Priority "High"
                }
                elseif ($timeSince -gt 30) {
                    Send-BackupAlert -Subject "WARNING: Transaction log backup overdue for $DatabaseName" -Body "Last log backup was $timeSince minutes ago at $lastBackup"
                }
            }
        }
    }
}

# Check backup file system health
$backupPaths = @(
    "\\backup-server\MealPrep\Production\Full\",
    "\\backup-server\MealPrep\Production\Differential\",
    "\\backup-server\MealPrep\Production\Log\"
)

foreach ($path in $backupPaths) {
    $alerts = Test-BackupFiles -BackupPath $path
    foreach ($alert in $alerts) {
        Send-BackupAlert -Subject "Backup System Alert" -Body $alert
        Write-Warning $alert
    }
}

Write-Host "Backup health check completed"
```

## Environment-Specific Configurations

### Production Configuration
```yaml
# File: config/backup-config-production.yml
production:
  database:
    name: "MealPrepProd"
    recovery_model: "FULL"
    
  backup_schedule:
    full_backup:
      frequency: "Weekly"
      day: "Sunday"
      time: "02:00"
      retention_days: 180
      
    differential_backup:
      frequency: "Daily"
      exclude_days: ["Sunday"]
      time: "02:00"
      retention_days: 30
      
    log_backup:
      frequency_minutes: 15
      retention_days: 7
      
  backup_locations:
    primary: "\\backup-server\MealPrep\Production\"
    secondary: "\\offsite-backup\MealPrep\Production\"
    
  compression: true
  encryption: true
  verification: true
  
  monitoring:
    email_alerts: true
    alert_recipients: ["admin@mealprep.com", "dba@mealprep.com"]
    
  disaster_recovery:
    rpo_target_minutes: 15  # Recovery Point Objective
    rto_target_hours: 4     # Recovery Time Objective
```

### Development Configuration
```yaml
# File: config/backup-config-development.yml
development:
  database:
    name: "MealPrepDev"
    recovery_model: "SIMPLE"
    
  backup_schedule:
    full_backup:
      frequency: "Daily"
      time: "04:00"
      retention_days: 7
      
  backup_locations:
    primary: "C:\Backups\MealPrep\Development\"
    
  compression: true
  encryption: false
  verification: true
  
  monitoring:
    email_alerts: false
```

This comprehensive database backup procedures guide provides enterprise-level backup and recovery strategies for the MealPrep application database across all environments.

*This backup procedures guide should be tested regularly and updated as infrastructure changes occur.*