# Week 09: Transaction Log Management

## Overview

This week focuses on mastering SQL Server transaction log management, which is crucial for database integrity, performance, and recovery operations. You'll learn about log file architecture, checkpoint processes, recovery models, and maintenance procedures essential for professional database administration.

## Learning Objectives

By the end of this week, you will be able to:
- Understand SQL Server transaction log architecture and internal mechanisms
- Configure and manage transaction log files for optimal performance
- Implement proper checkpoint processes and recovery strategies
- Choose appropriate recovery models for different business requirements
- Monitor transaction log space usage and growth
- Troubleshoot transaction log issues and perform log shrink operations
- Implement transaction log backup strategies and disaster recovery

## Monday - Transaction Log Architecture and Fundamentals

### Understanding Transaction Log Structure

SQL Server's transaction log is a critical component that ensures data durability and supports recovery operations. Unlike other database components, the log is a sequential write-ahead logging system.

#### Log Architecture Components

**Virtual Log Files (VLFs):**
```sql
-- Analyze transaction log virtual log files
SELECT 
    DB_NAME() AS DatabaseName,
    file_id,
    name AS LogFileName,
    physical_name AS PhysicalPath,
    CASE 
        WHEN status & 0x40 = 0x40 THEN 'Autogrow'
        WHEN status & 0x20 = 0x20 THEN 'Default'
        WHEN status & 0x80 = 0x80 THEN 'InitialSize'
        ELSE 'Regular'
    END AS LogFileStatus,
    size * 8 / 1024 AS SizeMB,
    max_size * 8 / 1024 AS MaxSizeMB,
    growth AS GrowthSetting,
    is_percent_growth AS IsPercentGrowth
FROM sys.database_files
WHERE type = 1; -- Log files only

-- Query VLF information using DBCC LOGINFO
-- This shows the internal VLF structure
DECLARE @LogInfo TABLE (
    FileID INT,
    FileSize BIGINT,
    StartOffset BIGINT,
    FSeqNo INT,
    Status INT,
    Parity INT,
    CreateLSN NUMERIC
);

INSERT INTO @LogInfo
EXEC ('DBCC LOGINFO');

SELECT 
    FileID,
    FileSize * 8 / 1024 AS FileSizeMB,
    StartOffset * 8 / 1024 / 1024 AS StartOffsetMB,
    FSeqNo AS FileSequenceNumber,
    Status,
    CASE 
        WHEN Status = 2 THEN 'Active'
        WHEN Status = 0 THEN 'Reusable'
        ELSE 'Unknown'
    END AS StatusDescription,
    Parity,
    CreateLSN
FROM @LogInfo
ORDER BY FSeqNo;
```

**Log Sequence Number (LSN):**
```sql
-- Understand LSN structure and importance
SELECT 
    'Current LSN Information' AS InfoType,
    CAST(DATABASEPROPERTYEX(DB_NAME(), 'LastLogBackup') AS DATETIME) AS LastLogBackup,
    DATABASEPROPERTYEX(DB_NAME(), 'RecoveryModel') AS RecoveryModel,
    (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID())) AS ActiveVLFs,
    -- Get the current active LSN range
    MIN(fi.[Current LSN]) AS MinActiveLSN,
    MAX(fi.[Current LSN]) AS MaxActiveLSN,
    -- Calculate log usage
    SUM(fi.FileSize * 8.0 / 1024) AS TotalLogSizeMB,
    SUM(CASE WHEN fi.Status = 2 THEN fi.FileSize * 8.0 / 1024 ELSE 0 END) AS ActiveLogSizeMB,
    SUM(CASE WHEN fi.Status = 0 THEN fi.FileSize * 8.0 / 1024 ELSE 0 END) AS ReusableLogSizeMB
FROM (
    DBCC LOGINFO
) AS fi
GROUP BY CAST(DATABASEPROPERTYEX(DB_NAME(), 'LastLogBackup') AS DATETIME), 
         DATABASEPROPERTYEX(DB_NAME(), 'RecoveryModel');

-- Analyze LSN chains for recovery planning
SELECT 
    bs.database_name,
    bs.backup_set_id,
    bs.backup_start_date,
    bs.backup_finish_date,
    bs.type,
    bs.first_lsn,
    bs.last_lsn,
    bs.checkpoint_lsn,
    CASE bs.type
        WHEN 'D' THEN 'Full Backup'
        WHEN 'I' THEN 'Differential Backup'
        WHEN 'L' THEN 'Log Backup'
        ELSE 'Unknown'
    END AS BackupType,
    bs.is_copy_only,
    -- Analyze LSN continuity
    LAG(bs.last_lsn) OVER (ORDER BY bs.backup_start_date) AS PreviousLastLSN,
    CASE 
        WHEN LAG(bs.last_lsn) OVER (ORDER BY bs.backup_start_date) IS NULL THEN 'Chain Start'
        WHEN LAG(bs.last_lsn) OVER (ORDER BY bs.backup_start_date) + 1 = bs.first_lsn THEN 'Continuous'
        ELSE 'Broken Chain - Verify Recovery'
    END AS LSNChainStatus
FROM msdb.dbo.backupset bs
WHERE bs.database_name = DB_NAME()
AND bs.backup_start_date >= DATEADD(DAY, -7, GETDATE())
ORDER BY bs.backup_start_date DESC;
```

#### Write-Ahead Logging (WAL) Mechanism

**Transaction Log Record Structure:**
```sql
-- Analyze transaction log records (requires elevated privileges)
-- This shows the internal log record structure
SELECT 
    [Current LSN],
    [Operation],
    [Context],
    [Transaction ID],
    [Log Record Fixed Length],
    [Log Record Fixed Length],
    [Log Block],
    [Log Data]
FROM fn_dblog(NULL, NULL)
WHERE [Transaction ID] IN (
    SELECT TOP 10 [Transaction ID]
    FROM fn_dblog(NULL, NULL)
    WHERE [Transaction ID] != '0000:00000000'
    GROUP BY [Transaction ID]
    ORDER BY COUNT(*) DESC
)
ORDER BY [Current LSN];

-- Analyze transaction log operations by type
SELECT 
    [Operation],
    COUNT(*) AS OperationCount,
    SUM([Log Record Fixed Length]) AS TotalBytes,
    MIN([Current LSN]) AS FirstOccurrence,
    MAX([Current LSN]) AS LastOccurrence,
    -- Analyze log patterns
    CASE [Operation]
        WHEN 'LOP_BEGIN_XACT' THEN 'Transaction start'
        WHEN 'LOP_COMMIT_XACT' THEN 'Transaction commit'
        WHEN 'LOP_ABORT_XACT' THEN 'Transaction rollback'
        WHEN 'LOP_MODIFY_ROW' THEN 'Row modification'
        WHEN 'LOP_DELETE_ROWS' THEN 'Row deletion'
        WHEN 'LOP_INSERT_ROWS' THEN 'Row insertion'
        WHEN 'LOP_MODIFY_COLUMNS' THEN 'Column modification'
        WHEN 'LOP_HOBT_DELTA' THEN 'Index page management'
        ELSE 'Other operation'
    END AS OperationDescription
FROM fn_dblog(NULL, NULL)
WHERE [Current LSN] > '00000000:00000000:0000'
GROUP BY [Operation]
ORDER BY OperationCount DESC;
```

### Checkpoint Processes

#### Automatic Checkpoints
```sql
-- Monitor automatic checkpoint activity
SELECT 
    database_id,
    DB_NAME(database_id) AS DatabaseName,
    CASE 
        WHEN database_id = 32767 THEN 'Resource Database'
        ELSE DB_NAME(database_id)
    END AS DBName,
    last_checkpoint_time,
    recovery_interval_sec AS RecoveryTargetSec,
    checkpoint_pages/sec AS CheckpointRate,
    backed_up_pages/sec AS BackupRate
FROM sys.dm_db_checkpoint_stats
WHERE database_id = DB_ID()
ORDER BY last_checkpoint_time DESC;

-- Get detailed checkpoint history
SELECT 
    checkpoint_start_date,
    checkpoint_end_date,
    DATEDIFF(SECOND, checkpoint_start_date, checkpoint_end_date) AS CheckpointDurationSec,
    checkpoint_pages_written / 1000.0 AS PagesWrittenM,
    duration_ms / 1000.0 AS DurationSec,
    -- Analyze checkpoint patterns
    CASE 
        WHEN duration_ms > 30000 THEN 'Long checkpoint - investigate'
        WHEN duration_ms > 5000 THEN 'Moderate checkpoint duration'
        ELSE 'Quick checkpoint'
    END AS CheckpointHealth,
    -- Performance impact
    CASE 
        WHEN checkpoint_pages_written > 1000000 THEN 'High I/O impact'
        WHEN checkpoint_pages_written > 100000 THEN 'Moderate I/O impact'
        ELSE 'Low I/O impact'
    END AS IOImpact
FROM msdb.dbo.dm_db_checkpoint_stats_history
WHERE database_id = DB_ID()
AND checkpoint_start_date >= DATEADD(HOUR, -24, GETDATE())
ORDER BY checkpoint_start_date DESC;
```

#### Manual and Indirect Checkpoints
```sql
-- Perform manual checkpoint and analyze impact
-- Run this in a test environment
DECLARE @StartTime DATETIME = GETDATE();

-- Manual checkpoint
CHECKPOINT;

DECLARE @EndTime DATETIME = GETDATE();
DECLARE @Duration INT = DATEDIFF(MILLISECOND, @StartTime, @EndTime);

-- Analyze checkpoint results
SELECT 
    'Manual Checkpoint Completed' AS Operation,
    @Duration AS DurationMs,
    @StartTime AS StartTime,
    @EndTime AS EndTime,
    (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID()) WHERE status = 2) AS ActiveVLFsBefore,
    (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) AS LogFileSizeMB;

-- Create procedure for controlled checkpoint operations
CREATE OR ALTER PROCEDURE sp_PerformControlledCheckpoint
    @DatabaseName NVARCHAR(128) = NULL,
    @TimeoutSeconds INT = 30,
    @MaxConcurrentCheckpoints INT = 1
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDB NVARCHAR(128) = DB_NAME();
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @StartTime DATETIME;
    DECLARE @EndTime DATETIME;
    DECLARE @Duration INT;
    
    SET @StartTime = GETDATE();
    
    -- Check if another checkpoint is in progress
    IF (SELECT COUNT(*) FROM sys.dm_exec_requests WHERE command = 'CHECKPOINT') >= @MaxConcurrentCheckpoints
    BEGIN
        RAISERROR('Another checkpoint is currently in progress', 16, 1);
        RETURN;
    END
    
    BEGIN TRY
        SET @SQL = 'USE ' + QUOTENAME(ISNULL(@DatabaseName, @CurrentDB)) + '; CHECKPOINT;';
        
        PRINT 'Starting checkpoint for: ' + ISNULL(@DatabaseName, @CurrentDB);
        
        EXEC sp_executesql @SQL;
        
        SET @EndTime = GETDATE();
        SET @Duration = DATEDIFF(MILLISECOND, @StartTime, @EndTime);
        
        -- Log checkpoint performance
        INSERT INTO CheckpointPerformanceLog (DatabaseName, OperationType, StartTime, EndTime, DurationMs, Status)
        VALUES (ISNULL(@DatabaseName, @CurrentDB), 'Manual', @StartTime, @EndTime, @Duration, 'Success');
        
        PRINT 'Checkpoint completed in ' + CAST(@Duration AS VARCHAR) + ' milliseconds';
        
        -- Return checkpoint summary
        SELECT 
            DatabaseName = ISNULL(@DatabaseName, @CurrentDB),
            Duration = @Duration,
            DurationCategory = CASE 
                WHEN @Duration > @TimeoutSeconds * 1000 THEN 'Timeout exceeded'
                WHEN @Duration > 30000 THEN 'Long'
                WHEN @Duration > 5000 THEN 'Moderate'
                ELSE 'Quick'
            END,
            ActiveVLFs = (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID()) WHERE status = 2)
            
    END TRY
    BEGIN CATCH
        SET @EndTime = GETDATE();
        SET @Duration = DATEDIFF(MILLISECOND, @StartTime, @EndTime);
        
        INSERT INTO CheckpointPerformanceLog (DatabaseName, OperationType, StartTime, EndTime, DurationMs, Status, ErrorMessage)
        VALUES (ISNULL(@DatabaseName, @CurrentDB), 'Manual', @StartTime, @EndTime, @Duration, 'Failed', ERROR_MESSAGE());
        
        RAISERROR('Checkpoint failed: %s', 16, 1, ERROR_MESSAGE());
    END CATCH
END;
GO

-- Create table to track checkpoint performance
CREATE TABLE CheckpointPerformanceLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(128),
    OperationType NVARCHAR(50), -- Manual, Automatic, Indirect
    StartTime DATETIME,
    EndTime DATETIME,
    DurationMs INT,
    Status NVARCHAR(20),
    ErrorMessage NVARCHAR(MAX),
    CreatedDate DATETIME DEFAULT GETDATE()
);

-- Grant execute permission
GRANT EXECUTE ON sp_PerformControlledCheckpoint TO [YourDbaUser];
```

#### Recovery Interval Configuration
```sql
-- Monitor and configure recovery interval
-- Current recovery interval setting
SELECT 
    'Current Recovery Configuration' AS ConfigType,
    value_in_use AS RecoveryIntervalSeconds,
    CASE 
        WHEN value_in_use <= 1 THEN 'Aggressive - More frequent checkpoints'
        WHEN value_in_use <= 3 THEN 'Balanced - Default recommended'
        WHEN value_in_use <= 10 THEN 'Conservative - Less frequent checkpoints'
        ELSE 'Very Conservative - Rare checkpoints'
    END AS ConfigDescription
FROM sys.configurations
WHERE name = 'recovery interval (max)';

-- Calculate actual recovery time for comparison
SELECT 
    'Recovery Performance Analysis' AS AnalysisType,
    -- Historical checkpoint data
    COUNT(*) AS TotalCheckpoints,
    AVG(duration_ms) AS AvgCheckpointDurationMs,
    MAX(duration_ms) AS MaxCheckpointDurationMs,
    MIN(duration_ms) AS MinCheckpointDurationMs,
    -- Recovery time estimation
    AVG(duration_ms) * (SELECT value_in_use FROM sys.configurations WHERE name = 'recovery interval (max)') AS EstimatedRecoveryTimeSec,
    -- Recommend recovery interval
    CASE 
        WHEN AVG(duration_ms) * (SELECT value_in_use FROM sys.configurations WHERE name = 'recovery interval (max)') > 600 THEN
            CAST(CEILING(600.0 / AVG(duration_ms) * 1000) AS INT)
        ELSE (SELECT value_in_use FROM sys.configurations WHERE name = 'recovery interval (max)')
    END AS RecommendedRecoveryIntervalSec,
    -- Performance categories
    CASE 
        WHEN AVG(duration_ms) > 30000 THEN 'SLOW_CHECKPOINTS'
        WHEN AVG(duration_ms) > 5000 THEN 'MODERATE_CHECKPOINTS'
        ELSE 'FAST_CHECKPOINTS'
    END AS CheckpointPerformanceCategory
FROM msdb.dbo.dm_db_checkpoint_stats_history
WHERE database_id = DB_ID()
AND checkpoint_start_date >= DATEADD(HOUR, -24, GETDATE());

-- Adjust recovery interval based on analysis
-- Execute this carefully in production
-- EXEC sp_configure 'show advanced options', 1;
-- RECONFIGURE;
-- EXEC sp_configure 'recovery interval (max)', 3; -- Set to 3 seconds
-- RECONFIGURE;
```

## Tuesday - Recovery Models and Configuration

### Full Recovery Model

The Full Recovery Model provides maximum data protection by logging all transactions and requiring regular transaction log backups.

**Configuration and Monitoring:**
```sql
-- Set database to full recovery model
ALTER DATABASE SalesDB
SET RECOVERY FULL;

-- Verify recovery model setting
SELECT 
    name AS DatabaseName,
    recovery_model_desc AS RecoveryModel,
    recovery_model AS RecoveryModelCode,
    log_reuse_wait_desc AS LogReuseWait,
    last_log_backup_lsn AS LastLogBackupLSN,
    -- Log backup chain analysis
    CASE 
        WHEN last_log_backup_lsn IS NOT NULL THEN 'Log backup chain intact'
        ELSE 'No log backup taken - check backup schedule'
    END AS LogBackupStatus,
    -- Space usage analysis
    (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) AS TotalLogSizeMB,
    (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
     FROM sys.dm_db_log_info(DB_ID())) AS ActiveLogSpaceMB
FROM sys.databases
WHERE name = DB_NAME();

-- Monitor log space usage in full recovery model
SELECT 
    'Log Space Usage Analysis' AS AnalysisType,
    DB_NAME() AS DatabaseName,
    -- Current log size
    (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) AS TotalLogSizeMB,
    -- Active log space (can't be reused)
    (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
     FROM sys.dm_db_log_info(DB_ID())) AS ActiveLogSpaceMB,
    -- Reusable log space
    (SELECT SUM(CASE WHEN status = 0 THEN size * 8.0 / 1024 ELSE 0 END) 
     FROM sys.dm_db_log_info(DB_ID())) AS ReusableLogSpaceMB,
    -- Log space percentage
    CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
          FROM sys.dm_db_log_info(DB_ID())) / 
         (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) * 100 AS DECIMAL(5,2)) AS ActiveLogPercent,
    -- Growth recommendations
    CASE 
        WHEN (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID())) / 
             (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) > 0.8 
        THEN 'LOG FILE FULL - Schedule log backup immediately'
        WHEN (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID())) / 
             (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) > 0.6 
        THEN 'High log usage - Schedule log backup soon'
        ELSE 'Log space healthy'
    END AS LogSpaceStatus;
```

**Transaction Log Backup Strategy:**
```sql
-- Create comprehensive transaction log backup procedure
CREATE OR ALTER PROCEDURE sp_TransactionLogBackup
    @DatabaseName NVARCHAR(128) = NULL,
    @BackupPath NVARCHAR(500) = 'C:\SQLBackups\LogBackups\',
    @BackupType VARCHAR(10) = 'LOG', -- LOG, DIFFERENTIAL, FULL
    @Compress BIT = 1,
    @Verify BIT = 1,
    @MaxRetainedBackups INT = 24, -- Keep 24 hours of log backups
    @AlertThresholdPercent DECIMAL(5,2) = 80.0
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDB NVARCHAR(128) = DB_NAME();
    DECLARE @TargetDB NVARCHAR(128) = ISNULL(@DatabaseName, @CurrentDB);
    DECLARE @BackupFile NVARCHAR(500);
    @BackupSQL NVARCHAR(MAX);
    DECLARE @StartTime DATETIME;
    DECLARE @EndTime DATETIME;
    DECLARE @BackupSizeMB DECIMAL(18,2);
    DECLARE @ActiveLogPercent DECIMAL(5,2);
    DECLARE @LogFileFull BIT = 0;
    
    SET @StartTime = GETDATE();
    
    -- Check if database is in full or bulk-logged recovery model
    IF DB_ID(@TargetDB) IS NULL
        RAISERROR('Database %s does not exist', 16, 1, @TargetDB);
    
    DECLARE @RecoveryModel NVARCHAR(60);
    SELECT @RecoveryModel = recovery_model_desc 
    FROM sys.databases 
    WHERE name = @TargetDB;
    
    IF @RecoveryModel NOT IN ('FULL', 'BULK_LOGGED')
        RAISERROR('Database %s is not in FULL or BULK_LOGGED recovery model. Current: %s', 16, 1, @TargetDB, @RecoveryModel);
    
    -- Check log space usage
    SELECT @ActiveLogPercent = 
        CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID(@TargetDB))) / 
             (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1) * 100 AS DECIMAL(5,2));
    
    IF @ActiveLogPercent > @AlertThresholdPercent
    BEGIN
        PRINT 'WARNING: Log space usage is ' + CAST(@ActiveLogPercent AS VARCHAR) + '% - Emergency log backup recommended';
        SET @LogFileFull = 1;
    END
    
    -- Generate backup file name
    DECLARE @DateString NVARCHAR(20) = CONVERT(NVARCHAR(20), GETDATE(), 112) + '_' + 
                                     REPLACE(CONVERT(NVARCHAR(20), GETDATE(), 108), ':', '');
    
    SET @BackupFile = @BackupPath + @TargetDB + '_' + @BackupType + '_' + @DateString + 
                     CASE @BackupType
                         WHEN 'LOG' THEN '.trn'
                         WHEN 'DIFFERENTIAL' THEN '.diff'
                         ELSE '.bak'
                     END;
    
    -- Ensure backup directory exists
    EXEC sp_configure 'show advanced options', 1;
    RECONFIGURE;
    EXEC sp_configure 'xp_cmdshell', 1;
    RECONFIGURE;
    
    DECLARE @DirCmd NVARCHAR(500) = 'mkdir "' + @BackupPath + '"';
    EXEC xp_cmdshell @DirCmd;
    
    -- Build backup command
    SET @BackupSQL = 'BACKUP ' + @BackupType + ' [' + @TargetDB + '] ' +
                    'TO DISK = ''' + @BackupFile + '''';
    
    IF @Compress = 1
        SET @BackupSQL = @BackupSQL + ' WITH COMPRESSION';
    
    -- Add specific options for log backups
    IF @BackupType = 'LOG'
    BEGIN
        SET @BackupSQL = @BackupSQL + ', NO_TRUNCATE, STATS = 10';
        
        -- Check log backup chain
        DECLARE @LastLogBackupLSN NUMERIC(25,0);
        SELECT @LastLogBackupLSN = last_log_backup_lsn 
        FROM sys.databases WHERE name = @TargetDB;
        
        IF @LastLogBackupLSN IS NULL
            PRINT 'WARNING: First log backup for this database - ensure full backup exists';
    END
    
    SET @BackupSQL = @BackupSQL + ', DESCRIPTION = ''' + 
                    @BackupType + ' backup of ' + @TargetDB + ' - ' + 
                    CONVERT(NVARCHAR(20), GETDATE(), 120) + '''';
    
    BEGIN TRY
        PRINT 'Starting ' + @BackupType + ' backup of ' + @TargetDB;
        PRINT 'Backup file: ' + @BackupFile;
        PRINT 'Log space before backup: ' + CAST(@ActiveLogPercent AS VARCHAR) + '%';
        
        -- Execute backup
        EXEC sp_executesql @BackupSQL;
        
        -- Get backup size
        DECLARE @BackupSizeQuery NVARCHAR(500) = 
            'SELECT CAST(SUM(backup_size / 1024.0 / 1024) AS DECIMAL(18,2)) ' +
            'FROM msdb.dbo.backupset ' +
            'WHERE database_name = ''' + @TargetDB + ''' ' +
            'AND type = ''' + CASE WHEN @BackupType = 'LOG' THEN 'L' ELSE CASE WHEN @BackupType = 'DIFFERENTIAL' THEN 'I' ELSE 'D' END END + ''' ' +
            'AND backup_start_date = ''' + CONVERT(VARCHAR(30), @StartTime, 120) + '''';
        
        EXEC sp_executesql @BackupSizeQuery;
        
        SET @EndTime = GETDATE();
        
        -- Verify backup if requested
        IF @Verify = 1
        BEGIN
            PRINT 'Verifying backup...';
            DECLARE @VerifySQL NVARCHAR(500) = 'RESTORE VERIFYONLY FROM DISK = ''' + @BackupFile + '''';
            EXEC sp_executesql @VerifySQL;
        END
        
        -- Log backup success
        INSERT INTO BackupHistory (DatabaseName, BackupType, BackupFile, StartTime, EndTime, BackupSizeMB, Status, LogSpaceBeforePercent, LogSpaceAfterPercent)
        VALUES (@TargetDB, @BackupType, @BackupFile, @StartTime, @EndTime, @BackupSizeMB, 'Success', @ActiveLogPercent, NULL);
        
        -- Clean up old backups
        EXEC sp_CleanupOldBackups @TargetDB, @BackupType, @MaxRetainedBackups;
        
        PRINT 'Backup completed successfully in ' + CAST(DATEDIFF(SECOND, @StartTime, @EndTime) AS VARCHAR) + ' seconds';
        
    END TRY
    BEGIN CATCH
        SET @EndTime = GETDATE();
        
        -- Log backup failure
        INSERT INTO BackupHistory (DatabaseName, BackupType, BackupFile, StartTime, EndTime, Status, ErrorMessage)
        VALUES (@TargetDB, @BackupType, @BackupFile, @StartTime, @EndTime, 'Failed', ERROR_MESSAGE());
        
        RAISERROR('Backup failed: %s', 16, 1, ERROR_MESSAGE());
    END CATCH
    
    -- Calculate post-backup log space
    DECLARE @LogSpaceAfter DECIMAL(5,2);
    
    -- Create temp table for post-backup log analysis
    SELECT @LogSpaceAfter = 
        CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID(@TargetDB))) / 
             (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1) * 100 AS DECIMAL(5,2));
    
    -- Update backup history with log space after backup
    UPDATE BackupHistory 
    SET LogSpaceAfterPercent = @LogSpaceAfter 
    WHERE DatabaseName = @TargetDB 
    AND BackupType = @BackupType 
    AND StartTime = @StartTime 
    AND EndTime = @EndTime;
    
    PRINT 'Log space after backup: ' + CAST(@LogSpaceAfter AS VARCHAR) + '%';
    
    RETURN @LogSpaceAfter;
END;
GO

-- Create backup history table
CREATE TABLE BackupHistory (
    HistoryID INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(128),
    BackupType NVARCHAR(50),
    BackupFile NVARCHAR(500),
    StartTime DATETIME,
    EndTime DATETIME,
    BackupSizeMB DECIMAL(18,2),
    Status NVARCHAR(20),
    LogSpaceBeforePercent DECIMAL(5,2),
    LogSpaceAfterPercent DECIMAL(5,2),
    ErrorMessage NVARCHAR(MAX),
    CreatedDate DATETIME DEFAULT GETDATE()
);

-- Create cleanup procedure
CREATE OR ALTER PROCEDURE sp_CleanupOldBackups
    @DatabaseName NVARCHAR(128),
    @BackupType NVARCHAR(50),
    @MaxRetainedBackups INT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @DeleteCount INT;
    
    WITH BackupToDelete AS (
        SELECT HistoryID, BackupFile,
               ROW_NUMBER() OVER (ORDER BY StartTime DESC) AS RowNum
        FROM BackupHistory
        WHERE DatabaseName = @DatabaseName 
        AND BackupType = @BackupType 
        AND Status = 'Success'
    )
    DELETE FROM BackupHistory
    WHERE HistoryID IN (
        SELECT HistoryID FROM BackupToDelete
        WHERE RowNum > @MaxRetainedBackups
    );
    
    SET @DeleteCount = @@ROWCOUNT;
    
    IF @DeleteCount > 0
        PRINT 'Cleaned up ' + CAST(@DeleteCount AS VARCHAR) + ' old backup records';
END;
```

### Simple Recovery Model

In the Simple Recovery Model, the transaction log is automatically truncated after each checkpoint, but point-in-time recovery is not possible.

**Configuration and Optimization:**
```sql
-- Set database to simple recovery model
ALTER DATABASE SalesDB
SET RECOVERY SIMPLE;

-- Monitor simple recovery model performance
SELECT 
    'Simple Recovery Model Analysis' AS AnalysisType,
    DB_NAME() AS DatabaseName,
    recovery_model_desc AS RecoveryModel,
    -- Log space analysis
    (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) AS TotalLogSizeMB,
    -- Active log space
    (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
     FROM sys.dm_db_log_info(DB_ID())) AS ActiveLogSpaceMB,
    -- Reusable space (should be most of log in simple model)
    (SELECT SUM(CASE WHEN status = 0 THEN size * 8.0 / 1024 ELSE 0 END) 
     FROM sys.dm_db_log_info(DB_ID())) AS ReusableLogSpaceMB,
    -- VLF analysis
    (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID())) AS TotalVLFs,
    (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID()) WHERE status = 2) AS ActiveVLFs,
    (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID()) WHERE status = 0) AS ReusableVLFs,
    -- Recovery considerations
    CASE 
        WHEN (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID())) / 
             (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) > 0.2 
        THEN 'High active log space - investigate transaction activity'
        ELSE 'Log space usage is normal for simple recovery model'
    END AS LogSpaceAssessment;

-- Analyze transaction activity that affects log space
SELECT 
    'Transaction Activity Analysis' AS AnalysisType,
    -- Long-running transactions
    session_id,
    start_time,
    DATEDIFF(SECOND, start_time, GETDATE()) AS DurationSeconds,
    cpu_time,
    reads,
    writes,
    logical_reads,
    -- Transaction details
    CASE 
        WHEN transaction_begin_time IS NOT NULL THEN
            DATEDIFF(SECOND, transaction_begin_time, GETDATE())
        ELSE 0
    END AS TransactionDurationSec,
    -- Log impact assessment
    CASE 
        WHEN DATEDIFF(SECOND, start_time, GETDATE()) > 300 THEN 'Long-running transaction - monitor log space'
        WHEN logical_reads > 1000000 THEN 'High I/O transaction - may impact log'
        ELSE 'Normal transaction'
    END AS TransactionImpact
FROM sys.dm_exec_requests
WHERE database_id = DB_ID()
AND (session_id IN (
    SELECT session_id FROM sys.dm_tran_active_snapshot_database_transactions
    UNION
    SELECT session_id FROM sys.dm_tran_locks WHERE request_status = 'WAIT'
    UNION
    SELECT session_id FROM sys.dm_exec_sessions WHERE is_user_process = 1
));

-- Create procedure for simple recovery model monitoring
CREATE OR ALTER PROCEDURE sp_MonitorSimpleRecoveryLog
    @DatabaseName NVARCHAR(128) = NULL,
    @AlertThresholdPercent DECIMAL(5,2) = 20.0,
    @CheckIntervalMinutes INT = 5
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDB NVARCHAR(128) = DB_NAME();
    DECLARE @TargetDB NVARCHAR(128) = ISNULL(@DatabaseName, @CurrentDB);
    
    -- Check if database is in simple recovery model
    DECLARE @RecoveryModel NVARCHAR(60);
    SELECT @RecoveryModel = recovery_model_desc 
    FROM sys.databases 
    WHERE name = @TargetDB;
    
    IF @RecoveryModel != 'SIMPLE'
    BEGIN
        PRINT 'Database ' + @TargetDB + ' is not in SIMPLE recovery model. Current: ' + @RecoveryModel;
        RETURN;
    END
    
    -- Analyze log space usage
    DECLARE @ActiveLogPercent DECIMAL(5,2);
    DECLARE @ActiveLogMB DECIMAL(18,2);
    DECLARE @TotalLogMB DECIMAL(18,2);
    DECLARE @LongTransactions INT;
    DECLARE @Recommendation NVARCHAR(500);
    
    -- Get log space metrics
    SELECT 
        @ActiveLogPercent = 
        CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID(@TargetDB))) / 
             (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1) * 100 AS DECIMAL(5,2)),
        @ActiveLogMB = (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
                        FROM sys.dm_db_log_info(DB_ID(@TargetDB))),
        @TotalLogMB = (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1);
    
    -- Count long-running transactions
    SELECT @LongTransactions = COUNT(*)
    FROM sys.dm_exec_requests
    WHERE database_id = DB_ID(@TargetDB)
    AND DATEDIFF(SECOND, start_time, GETDATE()) > 300; -- 5 minutes
    
    -- Generate recommendations
    SET @Recommendation = 
        CASE 
            WHEN @ActiveLogPercent > @AlertThresholdPercent AND @LongTransactions > 0 
            THEN 'High log usage with long transactions - consider transaction optimization and checkpoint frequency'
            WHEN @ActiveLogPercent > @AlertThresholdPercent 
            THEN 'High log usage - investigate transaction patterns and consider reducing transaction sizes'
            WHEN @LongTransactions > 5 
            THEN 'Multiple long-running transactions detected - optimize queries or increase checkpoint frequency'
            ELSE 'Log space usage is normal for simple recovery model'
        END;
    
    -- Log monitoring results
    INSERT INTO SimpleRecoveryLogMonitoring (DatabaseName, ActiveLogPercent, ActiveLogMB, TotalLogMB, LongTransactionCount, Recommendation, AlertTriggered)
    VALUES (@TargetDB, @ActiveLogPercent, @ActiveLogMB, @TotalLogMB, @LongTransactions, @Recommendation, 
            CASE WHEN @ActiveLogPercent > @AlertThresholdPercent OR @LongTransactions > 5 THEN 1 ELSE 0 END);
    
    -- Display current status
    PRINT '=== Simple Recovery Model Log Monitoring ===';
    PRINT 'Database: ' + @TargetDB;
    PRINT 'Active Log Usage: ' + CAST(@ActiveLogPercent AS VARCHAR) + '% (' + CAST(@ActiveLogMB AS VARCHAR) + ' MB of ' + CAST(@TotalLogMB AS VARCHAR) + ' MB)';
    PRINT 'Long-running Transactions: ' + CAST(@LongTransactions AS VARCHAR);
    PRINT 'Recommendation: ' + @Recommendation;
    
    IF @ActiveLogPercent > @AlertThresholdPercent
    BEGIN
        PRINT 'ALERT: Log usage exceeds threshold of ' + CAST(@AlertThresholdPercent AS VARCHAR) + '%';
        
        -- Attempt to reduce log usage by performing checkpoint
        CHECKPOINT;
        
        -- Check log space after checkpoint
        DECLARE @PostCheckpointLogPercent DECIMAL(5,2);
        SELECT @PostCheckpointLogPercent = 
            CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
                  FROM sys.dm_db_log_info(DB_ID(@TargetDB))) / 
                 (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1) * 100 AS DECIMAL(5,2));
        
        PRINT 'Log usage after checkpoint: ' + CAST(@PostCheckpointLogPercent AS VARCHAR) + '%';
    END
    
    RETURN @ActiveLogPercent;
END;
GO

-- Create monitoring table
CREATE TABLE SimpleRecoveryLogMonitoring (
    MonitoringID INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(128),
    ActiveLogPercent DECIMAL(5,2),
    ActiveLogMB DECIMAL(18,2),
    TotalLogMB DECIMAL(18,2),
    LongTransactionCount INT,
    Recommendation NVARCHAR(500),
    AlertTriggered BIT,
    CheckTime DATETIME DEFAULT GETDATE()
);

-- Grant execute permission
GRANT EXECUTE ON sp_MonitorSimpleRecoveryLog TO [YourDbaUser];
```

### Bulk-Logged Recovery Model

The Bulk-Logged Recovery Model is a hybrid approach that provides better performance for bulk operations while maintaining point-in-time recovery capability.

**Configuration and Bulk Operation Management:**
```sql
-- Set database to bulk-logged recovery model
ALTER DATABASE SalesDB
SET RECOVERY BULK_LOGGED;

-- Monitor bulk-logged operations
SELECT 
    'Bulk-Logged Recovery Model Analysis' AS AnalysisType,
    DB_NAME() AS DatabaseName,
    recovery_model_desc AS RecoveryModel,
    -- Check for bulk operations
    CASE 
        WHEN EXISTS (
            SELECT 1 FROM sys.dm_db_index_usage_stats ius
            INNER JOIN sys.indexes i ON ius.object_id = i.object_id AND ius.index_id = i.index_id
            WHERE ius.database_id = DB_ID()
            AND ius.user_updates > 100000
            AND ius.last_user_update > DATEADD(HOUR, -24, GETDATE())
        ) THEN 'Recent bulk operations detected'
        ELSE 'No bulk operations detected recently'
    END AS BulkOperationStatus,
    -- Log space impact
    (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) AS TotalLogSizeMB,
    (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
     FROM sys.dm_db_log_info(DB_ID())) AS ActiveLogSpaceMB,
    -- Recovery considerations
    CASE 
        WHEN EXISTS (
            SELECT 1 FROM fn_dblog(NULL, NULL) 
            WHERE [Operation] = 'LOP_HK' -- Hekaton operations
            AND [Current LSN] > '00000000:00000000:0000'
        ) THEN 'Contains Hekaton/columnstore operations - point-in-time recovery limited'
        ELSE 'Standard point-in-time recovery supported'
    END AS RecoveryCapability;

-- Analyze bulk operations impact on transaction log
SELECT 
    'Bulk Operation Log Impact' AS AnalysisType,
    -- Identify large table modifications
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.user_updates AS Updates,
    ips.user_seeks AS Seeks,
    ips.user_scans AS Scans,
    ips.user_lookups AS Lookups,
    ips.last_user_update AS LastUpdate,
    -- Bulk operation indicators
    CASE 
        WHEN ips.user_updates > 10000 AND ips.last_user_update > DATEADD(HOUR, -1, GETDATE())
        THEN 'Recent bulk update activity'
        WHEN ips.user_updates > 50000
        THEN 'High volume updates'
        ELSE 'Normal activity'
    END AS UpdatePattern,
    -- Log space considerations
    CASE 
        WHEN ips.user_updates > 50000 THEN 'May require more frequent log backups'
        ELSE 'Normal log backup frequency adequate'
    END AS LogBackupRecommendation
FROM sys.dm_db_index_usage_stats ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.database_id = DB_ID()
AND ips.user_updates > 1000
AND ips.last_user_update >= DATEADD(HOUR, -24, GETDATE())
ORDER BY ips.user_updates DESC;

-- Create procedure for bulk-logged recovery model management
CREATE OR ALTER PROCEDURE sp_ManageBulkLoggedRecovery
    @DatabaseName NVARCHAR(128) = NULL,
    @SwitchToFullAfterBulk BIT = 1,
    @MaxBulkOperations INT = 100,
    @AlertEmail NVARCHAR(200) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDB NVARCHAR(128) = DB_NAME();
    DECLARE @TargetDB NVARCHAR(128) = ISNULL(@DatabaseName, @CurrentDB);
    DECLARE @CurrentRecoveryModel NVARCHAR(60);
    DECLARE @BulkOperationCount INT;
    
    SELECT @CurrentRecoveryModel = recovery_model_desc 
    FROM sys.databases 
    WHERE name = @TargetDB;
    
    IF @CurrentRecoveryModel != 'BULK_LOGGED'
    BEGIN
        PRINT 'Database ' + @TargetDB + ' is not in BULK_LOGGED recovery model. Current: ' + @CurrentRecoveryModel;
        RETURN;
    END
    
    -- Count bulk operations in the last hour
    SELECT @BulkOperationCount = COUNT(DISTINCT 
        OBJECT_SCHEMA_NAME(ips.object_id) + '.' + OBJECT_NAME(ips.object_id)
    )
    FROM sys.dm_db_index_usage_stats ips
    WHERE ips.database_id = DB_ID(@TargetDB)
    AND ips.user_updates > 10000
    AND ips.last_user_update > DATEADD(HOUR, -1, GETDATE());
    
    PRINT '=== Bulk-Logged Recovery Model Management ===';
    PRINT 'Database: ' + @TargetDB;
    PRINT 'Current Recovery Model: ' + @CurrentRecoveryModel;
    PRINT 'Bulk Operations in Last Hour: ' + CAST(@BulkOperationCount AS VARCHAR);
    
    -- Check if we should switch to full recovery model
    IF @BulkOperationCount > @MaxBulkOperations
    BEGIN
        PRINT 'WARNING: High number of bulk operations detected (' + CAST(@BulkOperationCount AS VARCHAR) + ')';
        
        IF @SwitchToFullAfterBulk = 1
        BEGIN
            DECLARE @SQL NVARCHAR(500) = 'ALTER DATABASE ' + QUOTENAME(@TargetDB) + ' SET RECOVERY FULL;';
            
            BEGIN TRY
                EXEC sp_executesql @SQL;
                PRINT 'Switched to FULL recovery model due to high bulk operation count';
                
                -- Send alert if email configured
                IF @AlertEmail IS NOT NULL
                BEGIN
                    -- Email alert implementation would go here
                    PRINT 'Alert would be sent to: ' + @AlertEmail;
                END
                
                -- Log the switch
                INSERT INTO RecoveryModelChanges (DatabaseName, FromRecoveryModel, ToRecoveryModel, ChangeReason, ChangeTime)
                VALUES (@TargetDB, 'BULK_LOGGED', 'FULL', 'High bulk operation count', GETDATE());
                
            END TRY
            BEGIN CATCH
                PRINT 'Failed to switch recovery model: ' + ERROR_MESSAGE();
                
                -- Log the failure
                INSERT INTO RecoveryModelChanges (DatabaseName, FromRecoveryModel, ToRecoveryModel, ChangeReason, ErrorMessage, ChangeTime)
                VALUES (@TargetDB, 'BULK_LOGGED', 'FULL', 'High bulk operation count', ERROR_MESSAGE(), GETDATE());
            END CATCH
        END
        ELSE
        BEGIN
            PRINT 'Bulk-logged recovery model maintained - consider switching to full for better protection';
        END
    END
    ELSE
    BEGIN
        PRINT 'Bulk operation count is within acceptable limits';
    END
    
    -- Monitor log space and recommend actions
    DECLARE @ActiveLogPercent DECIMAL(5,2);
    SELECT @ActiveLogPercent = 
        CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID(@TargetDB))) / 
             (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1) * 100 AS DECIMAL(5,2));
    
    PRINT 'Active Log Usage: ' + CAST(@ActiveLogPercent AS VARCHAR) + '%';
    
    IF @ActiveLogPercent > 80
    BEGIN
        PRINT 'ALERT: High log usage - recommend log backup or checkpoint';
        CHECKPOINT;
    END
    
    -- Return current status
    SELECT 
        DatabaseName = @TargetDB,
        CurrentRecoveryModel = @CurrentRecoveryModel,
        BulkOperationCount = @BulkOperationCount,
        ActiveLogPercent = @ActiveLogPercent,
        RecommendedAction = CASE 
            WHEN @BulkOperationCount > @MaxBulkOperations THEN 'Consider switching to FULL recovery model'
            WHEN @ActiveLogPercent > 80 THEN 'Perform log backup or checkpoint'
            ELSE 'Continue monitoring'
        END;
END;
GO

-- Create tables for tracking recovery model changes
CREATE TABLE RecoveryModelChanges (
    ChangeID INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(128),
    FromRecoveryModel NVARCHAR(60),
    ToRecoveryModel NVARCHAR(60),
    ChangeReason NVARCHAR(200),
    ErrorMessage NVARCHAR(MAX),
    ChangeTime DATETIME DEFAULT GETDATE()
);

-- Grant execute permission
GRANT EXECUTE ON sp_ManageBulkLoggedRecovery TO [YourDbaUser];
```

## Wednesday - Transaction Log File Management

### Log File Sizing and Configuration

**Optimal Log File Configuration:**
```sql
-- Analyze current log file configuration and usage
SELECT 
    'Log File Configuration Analysis' AS AnalysisType,
    DB_NAME() AS DatabaseName,
    -- File configuration
    df.name AS LogFileName,
    df.physical_name AS PhysicalPath,
    df.size * 8 / 1024 AS CurrentSizeMB,
    df.max_size * 8 / 1024 AS MaxSizeMB,
    CASE df.is_percent_growth 
        WHEN 1 THEN CAST(df.growth AS VARCHAR) + '%'
        ELSE CAST(df.growth * 8 / 1024 AS VARCHAR) + ' MB'
    END AS GrowthSetting,
    -- Usage statistics
    (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
     FROM sys.dm_db_log_info(DB_ID())) AS ActiveSpaceMB,
    (SELECT SUM(CASE WHEN status = 0 THEN size * 8.0 / 1024 ELSE 0 END) 
     FROM sys.dm_db_log_info(DB_ID())) AS ReusableSpaceMB,
    -- VLF analysis
    (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID())) AS TotalVLFs,
    (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID()) WHERE status = 2) AS ActiveVLFs,
    (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID()) WHERE status = 0) AS ReusableVLFs,
    -- Efficiency metrics
    CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
          FROM sys.dm_db_log_info(DB_ID())) / (df.size * 8.0 / 1024) * 100 AS DECIMAL(5,2)) AS PercentActive,
    -- Recommendations
    CASE 
        WHEN (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID())) / (df.size * 8.0 / 1024) > 0.8 
        THEN 'LOG FULL - Immediate action required'
        WHEN (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID())) / (df.size * 8.0 / 1024) > 0.6 
        THEN 'High usage - monitor closely and backup log'
        WHEN df.max_size * 8 / 1024 < df.size * 8 / 1024 * 2 
        THEN 'Max size too small - increase max_size'
        WHEN (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID())) > 100 
        THEN 'Too many VLFs - consider log file rebuild'
        ELSE 'Configuration appears optimal'
    END AS ConfigurationStatus
FROM sys.database_files df
WHERE df.type = 1; -- Log files only

-- Analyze VLF distribution for optimization
SELECT 
    'VLF Distribution Analysis' AS AnalysisType,
    COUNT(*) AS TotalVLFs,
    MIN(FileSize * 8 / 1024) AS MinVLFSizeMB,
    MAX(FileSize * 8 / 1024) AS MaxVLFSizeMB,
    AVG(FileSize * 8 / 1024) AS AvgVLFSizeMB,
    -- VLF size distribution
    COUNT(CASE WHEN FileSize * 8 / 1024 < 64 THEN 1 END) AS SmallVLFs,
    COUNT(CASE WHEN FileSize * 8 / 1024 BETWEEN 64 AND 512 THEN 1 END) AS MediumVLFs,
    COUNT(CASE WHEN FileSize * 8 / 1024 > 512 THEN 1 END) AS LargeVLFs,
    -- Status distribution
    COUNT(CASE WHEN Status = 2 THEN 1 END) AS ActiveVLFs,
    COUNT(CASE WHEN Status = 0 THEN 1 END) AS ReusableVLFs,
    -- Optimization recommendations
    CASE 
        WHEN COUNT(*) > 100 THEN 'TOO MANY VLFS - Rebuild log file'
        WHEN MIN(FileSize * 8 / 1024) < 1 THEN 'Some VLFs too small - Rebuild recommended'
        WHEN COUNT(CASE WHEN FileSize * 8 / 1024 > 1024 THEN 1 END) > COUNT(*) * 0.3 THEN 'Too many large VLFs - Consider smaller VLFs'
        ELSE 'VLF distribution is acceptable'
    END AS VLFOptimizationRecommendation
FROM (
    DBCC LOGINFO
) AS fi;

-- Create optimal log file configuration procedure
CREATE OR ALTER PROCEDURE sp_OptimizeLogFile
    @DatabaseName NVARCHAR(128) = NULL,
    @TargetLogSizeMB INT = NULL,
    @VLFSizeMB DECIMAL(10,2) = 64.0, -- Target VLF size
    @GrowthMB INT = 256, -- Growth increment in MB
    @MaxSizeMB INT = NULL, -- Maximum size, if null use 10x initial size
    @ShrinkLogFile BIT = 0, -- Allow shrinking if log is underutilized
    @BackupBeforeRebuild BIT = 1
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDB NVARCHAR(128) = DB_NAME();
    DECLARE @TargetDB NVARCHAR(128) = ISNULL(@DatabaseName, @CurrentDB);
    DECLARE @CurrentLogSizeMB INT;
    DECLARE @CurrentMaxSizeMB INT;
    DECLARE @LogFileName NVARCHAR(128);
    DECLARE @LogPhysicalPath NVARCHAR(500);
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @BackupFile NVARCHAR(500);
    DECLARE @StartTime DATETIME;
    
    SET @StartTime = GETDATE();
    
    -- Verify database exists and get current configuration
    IF DB_ID(@TargetDB) IS NULL
        RAISERROR('Database %s does not exist', 16, 1, @TargetDB);
    
    SELECT 
        @CurrentLogSizeMB = size * 8 / 1024,
        @CurrentMaxSizeMB = max_size * 8 / 1024,
        @LogFileName = name,
        @LogPhysicalPath = physical_name
    FROM sys.database_files 
    WHERE database_id = DB_ID(@TargetDB) AND type = 1;
    
    -- Calculate target sizes if not provided
    IF @TargetLogSizeMB IS NULL
        SET @TargetLogSizeMB = @CurrentLogSizeMB;
    
    IF @MaxSizeMB IS NULL
        SET @MaxSizeMB = @TargetLogSizeMB * 10;
    
    -- Get current VLF count
    DECLARE @CurrentVLFCount INT;
    DECLARE @ActiveLogPercent DECIMAL(5,2);
    
    -- Get log space usage
    DECLARE @LogSpaceQuery NVARCHAR(500) = '
        SELECT 
            @VLFCount = COUNT(*),
            @ActivePercent = 
            CAST(SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) / 
                 (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) * 100 AS DECIMAL(5,2))
        FROM sys.dm_db_log_info(DB_ID(''' + @TargetDB + '''))';
    
    EXEC sp_executesql @LogSpaceQuery, 
        N'@VLFCount INT OUTPUT, @ActivePercent DECIMAL(5,2) OUTPUT', 
        @VLFCount = @CurrentVLFCount OUTPUT, @ActivePercent = @ActiveLogPercent OUTPUT;
    
    PRINT '=== Log File Optimization for ' + @TargetDB + ' ===';
    PRINT 'Current log file size: ' + CAST(@CurrentLogSizeMB AS VARCHAR) + ' MB';
    PRINT 'Current max size: ' + CAST(@CurrentMaxSizeMB AS VARCHAR) + ' MB';
    PRINT 'Current VLF count: ' + CAST(@CurrentVLFCount AS VARCHAR);
    PRINT 'Current active log space: ' + CAST(@ActiveLogPercent AS VARCHAR) + '%';
    PRINT 'Target log file size: ' + CAST(@TargetLogSizeMB AS VARCHAR) + ' MB';
    
    -- Determine if optimization is needed
    IF @CurrentVLFCount > 100 OR @ActiveLogPercent > 80 OR @CurrentLogSizeMB * 0.3 < @TargetLogSizeMB
    BEGIN
        PRINT 'Log file optimization recommended';
        
        -- Backup transaction log before rebuilding (if full/bulk-logged recovery model)
        DECLARE @RecoveryModel NVARCHAR(60);
        SELECT @RecoveryModel = recovery_model_desc 
        FROM sys.databases 
        WHERE name = @TargetDB;
        
        IF @RecoveryModel IN ('FULL', 'BULK_LOGGED') AND @BackupBeforeRebuild = 1
        BEGIN
            SET @BackupFile = 'C:\SQLBackups\LogBackup_' + @TargetDB + '_' + 
                            CONVERT(NVARCHAR(20), GETDATE(), 112) + '_' + 
                            REPLACE(CONVERT(NVARCHAR(20), GETDATE(), 108), ':', '') + '.trn';
            
            SET @SQL = 'BACKUP LOG [' + @TargetDB + '] TO DISK = ''' + @BackupFile + ''' WITH COMPRESSION;';
            
            BEGIN TRY
                PRINT 'Creating backup before log file rebuild...';
                EXEC sp_executesql @SQL;
                PRINT 'Backup completed: ' + @BackupFile;
            END TRY
            BEGIN CATCH
                PRINT 'WARNING: Log backup failed before rebuild: ' + ERROR_MESSAGE();
            END CATCH
        END
        
        -- Perform log file rebuild
        SET @SQL = 'USE ' + QUOTENAME(@TargetDB) + '; ' +
                  'DBCC SHRINKFILE(' + QUOTENAME(@LogFileName) + ', 1); ' + -- Shrink to minimum
                  'ALTER DATABASE ' + QUOTENAME(@TargetDB) + ' ' +
                  'MODIFY FILE (' +
                  'NAME = ' + QUOTENAME(@LogFileName) + ', ' +
                  'SIZE = ' + CAST(@TargetLogSizeMB AS VARCHAR) + 'MB, ' +
                  'MAXSIZE = ' + CAST(@MaxSizeMB AS VARCHAR) + 'MB, ' +
                  'FILEGROWTH = ' + CAST(@GrowthMB AS VARCHAR) + 'MB' +
                  ');';
        
        BEGIN TRY
            PRINT 'Rebuilding log file...';
            EXEC sp_executesql @SQL;
            PRINT 'Log file rebuilt successfully';
            
            -- Log the optimization
            INSERT INTO LogFileOptimizations (DatabaseName, OperationType, OldSizeMB, NewSizeMB, OldMaxSizeMB, NewMaxSizeMB, OldVLFCount, NewVLFCount, StartTime, EndTime, Status)
            VALUES (@TargetDB, 'Rebuild', @CurrentLogSizeMB, @TargetLogSizeMB, @CurrentMaxSizeMB, @MaxSizeMB, @CurrentVLFCount, NULL, @StartTime, GETDATE(), 'Success');
            
        END TRY
        BEGIN CATCH
            PRINT 'Log file rebuild failed: ' + ERROR_MESSAGE();
            
            -- Log the failure
            INSERT INTO LogFileOptimizations (DatabaseName, OperationType, OldSizeMB, NewSizeMB, ErrorMessage, StartTime, EndTime, Status)
            VALUES (@TargetDB, 'Rebuild', @CurrentLogSizeMB, @TargetLogSizeMB, ERROR_MESSAGE(), @StartTime, GETDATE(), 'Failed');
            
            RAISERROR('Log file optimization failed: %s', 16, 1, ERROR_MESSAGE());
        END CATCH
    END
    ELSE
    BEGIN
        PRINT 'Log file configuration appears optimal - no changes needed';
    END
    
    -- Return post-optimization status
    DECLARE @NewVLFCount INT;
    DECLARE @NewActiveLogPercent DECIMAL(5,2);
    
    -- Get updated VLF count and log space usage
    EXEC sp_executesql @LogSpaceQuery, 
        N'@VLFCount INT OUTPUT, @ActivePercent DECIMAL(5,2) OUTPUT', 
        @VLFCount = @NewVLFCount OUTPUT, @ActivePercent = @NewActiveLogPercent OUTPUT;
    
    -- Update optimization log with new VLF count
    UPDATE LogFileOptimizations 
    SET NewVLFCount = @NewVLFCount,
        NewActiveLogPercent = @NewActiveLogPercent,
        EndTime = GETDATE()
    WHERE DatabaseName = @TargetDB 
    AND StartTime = @StartTime 
    AND OperationType = 'Rebuild'
    AND Status = 'Success';
    
    -- Display final status
    SELECT 
        DatabaseName = @TargetDB,
        OperationStatus = CASE WHEN @CurrentVLFCount > 100 OR @ActiveLogPercent > 80 THEN 'Optimized' ELSE 'No Change' END,
        PreviousVLFCount = @CurrentVLFCount,
        CurrentVLFCount = @NewVLFCount,
        VLFReduction = @CurrentVLFCount - @NewVLFCount,
        PreviousActiveLogPercent = @ActiveLogPercent,
        CurrentActiveLogPercent = @NewActiveLogPercent,
        OptimizationTime = DATEDIFF(SECOND, @StartTime, GETDATE())
END;
GO

-- Create table to track log file optimizations
CREATE TABLE LogFileOptimizations (
    OptimizationID INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(128),
    OperationType NVARCHAR(50), -- Rebuild, Shrink, Expand
    OldSizeMB INT,
    NewSizeMB INT,
    OldMaxSizeMB INT,
    NewMaxSizeMB INT,
    OldVLFCount INT,
    NewVLFCount INT,
    OldActiveLogPercent DECIMAL(5,2),
    NewActiveLogPercent DECIMAL(5,2),
    ErrorMessage NVARCHAR(MAX),
    StartTime DATETIME,
    EndTime DATETIME,
    Status NVARCHAR(20)
);

-- Grant execute permission
GRANT EXECUTE ON sp_OptimizeLogFile TO [YourDbaUser];
```

### Log File Monitoring and Alerts

```sql
-- Create comprehensive log monitoring procedure
CREATE OR ALTER PROCEDURE sp_MonitorTransactionLog
    @DatabaseName NVARCHAR(128) = NULL,
    @AlertThresholdPercent DECIMAL(5,2) = 80.0,
    @WarningThresholdPercent DECIMAL(5,2) = 60.0,
    @MaxActiveVLFs INT = 50,
    @GenerateAlerts BIT = 1,
    @EmailRecipients NVARCHAR(MAX) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDB NVARCHAR(128) = DB_NAME();
    DECLARE @TargetDB NVARCHAR(128) = ISNULL(@DatabaseName, @CurrentDB);
    DECLARE @AlertLevel NVARCHAR(20);
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @ActiveLogPercent DECIMAL(5,2);
    DECLARE @ActiveLogMB DECIMAL(18,2);
    DECLARE @TotalLogMB DECIMAL(18,2);
    DECLARE @VLFCount INT;
    DECLARE @ActiveVLFCount INT;
    DECLARE @LastBackupTime DATETIME;
    DECLARE @RecoveryModel NVARCHAR(60);
    DECLARE @LongTransactionCount INT;
    
    -- Get database information
    SELECT 
        @RecoveryModel = recovery_model_desc,
        @LastBackupTime = CAST(last_log_backup_lsn AS DATETIME) -- Note: This may not work in all SQL Server versions
    FROM sys.databases 
    WHERE name = @TargetDB;
    
    -- Get log space metrics
    SELECT 
        @ActiveLogPercent = 
        CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID(@TargetDB))) / 
             (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1) * 100 AS DECIMAL(5,2)),
        @ActiveLogMB = (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
                        FROM sys.dm_db_log_info(DB_ID(@TargetDB))),
        @TotalLogMB = (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1),
        @VLFCount = (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID(@TargetDB))),
        @ActiveVLFCount = (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID(@TargetDB)) WHERE status = 2);
    
    -- Count long-running transactions
    SELECT @LongTransactionCount = COUNT(*)
    FROM sys.dm_exec_requests
    WHERE database_id = DB_ID(@TargetDB)
    AND DATEDIFF(SECOND, start_time, GETDATE()) > 300; -- 5 minutes
    
    -- Determine alert level
    IF @ActiveLogPercent >= @AlertThresholdPercent
        SET @AlertLevel = 'CRITICAL';
    ELSE IF @ActiveLogPercent >= @WarningThresholdPercent
        SET @AlertLevel = 'WARNING';
    ELSE
        SET @AlertLevel = 'NORMAL';
    
    -- Generate alert message
    SET @AlertMessage = 
        '=== Transaction Log Monitoring Alert ===' + CHAR(13) + CHAR(10) +
        'Database: ' + @TargetDB + CHAR(13) + CHAR(10) +
        'Alert Level: ' + @AlertLevel + CHAR(13) + CHAR(10) +
        'Active Log Usage: ' + CAST(@ActiveLogPercent AS VARCHAR) + '% (' + CAST(@ActiveLogMB AS VARCHAR) + ' MB of ' + CAST(@TotalLogMB AS VARCHAR) + ' MB)' + CHAR(13) + CHAR(10) +
        'Recovery Model: ' + @RecoveryModel + CHAR(13) + CHAR(10) +
        'VLF Status: ' + CAST(@ActiveVLFCount AS VARCHAR) + ' active of ' + CAST(@VLFCount AS VARCHAR) + ' total' + CHAR(13) + CHAR(10) +
        'Long-running Transactions: ' + CAST(@LongTransactionCount AS VARCHAR) + CHAR(13) + CHAR(10) +
        'Timestamp: ' + CONVERT(VARCHAR(30), GETDATE(), 120) + CHAR(13) + CHAR(10);
    
    -- Add recommendations
    SET @AlertMessage = @AlertMessage + CHAR(13) + CHAR(10) + 'Recommendations:' + CHAR(13) + CHAR(10);
    
    IF @ActiveLogPercent >= @AlertThresholdPercent
    BEGIN
        IF @RecoveryModel IN ('FULL', 'BULK_LOGGED')
            SET @AlertMessage = @AlertMessage + '- IMMEDIATE: Perform transaction log backup' + CHAR(13) + CHAR(10);
        
        SET @AlertMessage = @AlertMessage + '- Consider shrinking log file if regularly at this level' + CHAR(13) + CHAR(10);
        SET @AlertMessage = @AlertMessage + '- Investigate long-running transactions' + CHAR(13) + CHAR(10);
    END
    ELSE IF @ActiveLogPercent >= @WarningThresholdPercent
    BEGIN
        SET @AlertMessage = @AlertMessage + '- Schedule log backup soon (if full/bulk-logged model)' + CHAR(13) + CHAR(10);
        SET @AlertMessage = @AlertMessage + '- Monitor log growth patterns' + CHAR(13) + CHAR(10);
    END
    
    IF @VLFCount > @MaxActiveVLFs
        SET @AlertMessage = @AlertMessage + '- High VLF count: Consider log file rebuild to reduce VLF count' + CHAR(13) + CHAR(10);
    
    IF @LongTransactionCount > 3
        SET @AlertMessage = @AlertMessage + '- Multiple long-running transactions: Review and optimize queries' + CHAR(13) + CHAR(10);
    
    -- Log monitoring results
    INSERT INTO TransactionLogMonitoring (DatabaseName, AlertLevel, ActiveLogPercent, ActiveLogMB, TotalLogMB, VLFCount, ActiveVLFCount, LongTransactionCount, RecoveryModel, AlertMessage, MonitoringTime)
    VALUES (@TargetDB, @AlertLevel, @ActiveLogPercent, @ActiveLogMB, @TotalLogMB, @VLFCount, @ActiveVLFCount, @LongTransactionCount, @RecoveryModel, @AlertMessage, GETDATE());
    
    -- Display results
    PRINT @AlertMessage;
    
    -- Send alerts if configured
    IF @GenerateAlerts = 1 AND @AlertLevel IN ('WARNING', 'CRITICAL')
    BEGIN
        PRINT 'ALERT: Sending notification for ' + @AlertLevel + ' level alert';
        
        -- In a real implementation, you would send email alerts here
        -- Example: msdb.dbo.sp_send_dbmail with @recipients = @EmailRecipients
        
        -- Log alert sending
        INSERT INTO AlertNotifications (DatabaseName, AlertLevel, AlertType, Recipients, AlertTime, Status)
        VALUES (@TargetDB, @AlertLevel, 'Transaction Log Alert', @EmailRecipients, GETDATE(), 'Sent');
    END
    
    -- Return status for further processing
    SELECT 
        DatabaseName = @TargetDB,
        AlertLevel = @AlertLevel,
        ActiveLogPercent = @ActiveLogPercent,
        ActiveLogMB = @ActiveLogMB,
        TotalLogMB = @TotalLogMB,
        VLFCount = @VLFCount,
        ActiveVLFCount = @ActiveVLFCount,
        LongTransactionCount = @LongTransactionCount,
        RecoveryModel = @RecoveryModel,
        RecommendedAction = CASE 
            WHEN @ActiveLogPercent >= @AlertThresholdPercent THEN 'BACKUP_LOG_OR_SHRINK'
            WHEN @ActiveLogPercent >= @WarningThresholdPercent THEN 'SCHEDULE_LOG_BACKUP'
            WHEN @VLFCount > @MaxActiveVLFs THEN 'REBUILD_LOG_FILE'
            WHEN @LongTransactionCount > 3 THEN 'OPTIMIZE_TRANSACTIONS'
            ELSE 'MONITOR'
        END
END;
GO

-- Create monitoring tables
CREATE TABLE TransactionLogMonitoring (
    MonitoringID INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(128),
    AlertLevel NVARCHAR(20), -- CRITICAL, WARNING, NORMAL
    ActiveLogPercent DECIMAL(5,2),
    ActiveLogMB DECIMAL(18,2),
    TotalLogMB DECIMAL(18,2),
    VLFCount INT,
    ActiveVLFCount INT,
    LongTransactionCount INT,
    RecoveryModel NVARCHAR(60),
    AlertMessage NVARCHAR(MAX),
    MonitoringTime DATETIME DEFAULT GETDATE()
);

CREATE TABLE AlertNotifications (
    NotificationID INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(128),
    AlertLevel NVARCHAR(20),
    AlertType NVARCHAR(100),
    Recipients NVARCHAR(MAX),
    Subject NVARCHAR(500),
    AlertTime DATETIME DEFAULT GETDATE(),
    Status NVARCHAR(20), -- Sent, Failed, Pending
    ErrorMessage NVARCHAR(MAX)
);

-- Grant execute permissions
GRANT EXECUTE ON sp_MonitorTransactionLog TO [YourDbaUser];
```

## Thursday - Transaction Log Backup and Recovery

### Automated Backup Strategies

**Log Backup Scheduling and Management:**
```sql
-- Create comprehensive log backup strategy procedure
CREATE OR ALTER PROCEDURE sp_LogBackupStrategy
    @DatabaseName NVARCHAR(128) = NULL,
    @BackupFrequencyMinutes INT = 15, -- How often to backup logs
    @RetentionHours INT = 72, -- Keep backups for 72 hours
    @BackupPath NVARCHAR(500) = 'C:\SQLBackups\TransactionLogs\',
    @CompressBackups BIT = 1,
    @VerifyBackups BIT = 1,
    @GenerateInventory BIT = 1
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDB NVARCHAR(128) = DB_NAME();
    DECLARE @TargetDB NVARCHAR(128) = ISNULL(@DatabaseName, @CurrentDB);
    DECLARE @RecoveryModel NVARCHAR(60);
    DECLARE @BackupFile NVARCHAR(500);
    DECLARE @BackupSizeMB DECIMAL(18,2);
    DECLARE @BackupDuration INT;
    DECLARE @StartTime DATETIME;
    DECLARE @EndTime DATETIME;
    DECLARE @LogSpaceBefore DECIMAL(5,2);
    DECLARE @LogSpaceAfter DECIMAL(5,2);
    DECLARE @BackupSuccess BIT = 0;
    
    SET @StartTime = GETDATE();
    
    -- Verify database exists and get recovery model
    IF DB_ID(@TargetDB) IS NULL
        RAISERROR('Database %s does not exist', 16, 1, @TargetDB);
    
    SELECT @RecoveryModel = recovery_model_desc 
    FROM sys.databases 
    WHERE name = @TargetDB;
    
    IF @RecoveryModel NOT IN ('FULL', 'BULK_LOGGED')
    BEGIN
        PRINT 'Database ' + @TargetDB + ' is not in FULL or BULK_LOGGED recovery model. Log backups not applicable.';
        RETURN;
    END
    
    -- Get log space before backup
    SELECT @LogSpaceBefore = 
        CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID(@TargetDB))) / 
             (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1) * 100 AS DECIMAL(5,2));
    
    PRINT '=== Log Backup Strategy Execution ===';
    PRINT 'Database: ' + @TargetDB;
    PRINT 'Recovery Model: ' + @RecoveryModel;
    PRINT 'Backup Frequency: Every ' + CAST(@BackupFrequencyMinutes AS VARCHAR) + ' minutes';
    PRINT 'Log Space Before Backup: ' + CAST(@LogSpaceBefore AS VARCHAR) + '%';
    
    -- Check if backup is due based on frequency
    DECLARE @LastBackupTime DATETIME;
    DECLARE @TimeSinceLastBackup INT;
    
    SELECT @LastBackupTime = MAX(EndTime)
    FROM BackupHistory
    WHERE DatabaseName = @TargetDB 
    AND BackupType = 'LOG' 
    AND Status = 'Success';
    
    IF @LastBackupTime IS NOT NULL
    BEGIN
        SET @TimeSinceLastBackup = DATEDIFF(MINUTE, @LastBackupTime, GETDATE());
        
        IF @TimeSinceLastBackup < @BackupFrequencyMinutes
        BEGIN
            PRINT 'Log backup not yet due. Last backup: ' + CAST(@LastBackupTime AS VARCHAR) + 
                  ' (' + CAST(@TimeSinceLastBackup AS VARCHAR) + ' minutes ago)';
            RETURN;
        END
    END
    
    -- Generate backup file name with timestamp
    DECLARE @DateTimeString NVARCHAR(20) = 
        CONVERT(NVARCHAR(20), GETDATE(), 112) + '_' + 
        REPLACE(CONVERT(NVARCHAR(20), GETDATE(), 108), ':', '');
    
    SET @BackupFile = @BackupPath + @TargetDB + '_LOG_' + @DateTimeString + '.trn';
    
    -- Create backup directory if it doesn't exist
    EXEC sp_configure 'show advanced options', 1;
    RECONFIGURE;
    EXEC sp_configure 'xp_cmdshell', 1;
    RECONFIGURE;
    
    DECLARE @DirCreationCmd NVARCHAR(500) = 'mkdir "' + @BackupPath + '"';
    EXEC xp_cmdshell @DirCreationCmd;
    
    -- Build and execute backup command
    DECLARE @BackupSQL NVARCHAR(1000) = 
        'BACKUP LOG [' + @TargetDB + '] ' +
        'TO DISK = ''' + @BackupFile + '''' +
        'WITH COMPRESSION, NO_TRUNCATE, STATS = 10, ' +
        'DESCRIPTION = ''' + @TargetDB + ' Transaction Log Backup - ' + 
        CONVERT(NVARCHAR(20), GETDATE(), 120) + '''';
    
    IF @CompressBackups = 0
        SET @BackupSQL = REPLACE(@BackupSQL, 'COMPRESSION, ', '');
    
    BEGIN TRY
        PRINT 'Starting transaction log backup...';
        PRINT 'Backup file: ' + @BackupFile;
        
        EXEC sp_executesql @BackupSQL;
        SET @BackupSuccess = 1;
        
        SET @EndTime = GETDATE();
        SET @BackupDuration = DATEDIFF(SECOND, @StartTime, @EndTime);
        
        PRINT 'Backup completed in ' + CAST(@BackupDuration AS VARCHAR) + ' seconds';
        
        -- Get backup size
        SELECT @BackupSizeMB = backup_size / 1024.0 / 1024
        FROM msdb.dbo.backupset
        WHERE database_name = @TargetDB 
        AND type = 'L'
        AND backup_start_date = @StartTime;
        
        -- Verify backup if requested
        IF @VerifyBackups = 1
        BEGIN
            PRINT 'Verifying backup integrity...';
            DECLARE @VerifySQL NVARCHAR(500) = 'RESTORE VERIFYONLY FROM DISK = ''' + @BackupFile + '''';
            
            BEGIN TRY
                EXEC sp_executesql @VerifySQL;
                PRINT 'Backup verification passed';
            END TRY
            BEGIN CATCH
                PRINT 'WARNING: Backup verification failed: ' + ERROR_MESSAGE();
            END CATCH
        END
        
        -- Get log space after backup
        SELECT @LogSpaceAfter = 
            CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
                  FROM sys.dm_db_log_info(DB_ID(@TargetDB))) / 
                 (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1) * 100 AS DECIMAL(5,2));
        
        -- Log successful backup
        INSERT INTO BackupHistory (
            DatabaseName, BackupType, BackupFile, StartTime, EndTime, 
            BackupSizeMB, Status, LogSpaceBeforePercent, LogSpaceAfterPercent
        ) VALUES (
            @TargetDB, 'LOG', @BackupFile, @StartTime, @EndTime,
            @BackupSizeMB, 'Success', @LogSpaceBefore, @LogSpaceAfter
        );
        
        -- Log space improvement
        DECLARE @LogSpaceReduction DECIMAL(5,2) = @LogSpaceBefore - @LogSpaceAfter;
        PRINT 'Log space reduced by ' + CAST(@LogSpaceReduction AS VARCHAR) + '% (' + 
              CAST(@LogSpaceBefore AS VARCHAR) + '% → ' + CAST(@LogSpaceAfter AS VARCHAR) + '%)';
        
        -- Clean up old backups
        EXEC sp_CleanupOldBackups @TargetDB, 'LOG', @RetentionHours / (@BackupFrequencyMinutes / 60.0);
        
        -- Generate backup inventory if requested
        IF @GenerateInventory = 1
        BEGIN
            EXEC sp_GenerateBackupInventory @TargetDB, 'LOG';
        END
        
    END TRY
    BEGIN CATCH
        SET @EndTime = GETDATE();
        SET @BackupSuccess = 0;
        
        -- Log failed backup
        INSERT INTO BackupHistory (
            DatabaseName, BackupType, BackupFile, StartTime, EndTime, Status, ErrorMessage
        ) VALUES (
            @TargetDB, 'LOG', @BackupFile, @StartTime, @EndTime, 'Failed', ERROR_MESSAGE()
        );
        
        RAISERROR('Log backup failed: %s', 16, 1, ERROR_MESSAGE());
    END CATCH
    
    -- Return backup summary
    SELECT 
        DatabaseName = @TargetDB,
        BackupStatus = CASE WHEN @BackupSuccess = 1 THEN 'Success' ELSE 'Failed' END,
        BackupSizeMB = @BackupSizeMB,
        BackupDurationSec = @BackupDuration,
        LogSpaceBeforePercent = @LogSpaceBefore,
        LogSpaceAfterPercent = @LogSpaceAfter,
        LogSpaceReductionPercent = @LogSpaceReduction,
        BackupFile = @BackupFile,
        NextBackupDue = DATEADD(MINUTE, @BackupFrequencyMinutes, GETDATE())
    
    RETURN CASE WHEN @BackupSuccess = 1 THEN 0 ELSE -1 END;
END;
GO

-- Create backup inventory procedure
CREATE OR ALTER PROCEDURE sp_GenerateBackupInventory
    @DatabaseName NVARCHAR(128),
    @BackupType NVARCHAR(50) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @InventoryFile NVARCHAR(500) = 
        'C:\SQLBackups\Inventory\BackupInventory_' + @DatabaseName + '_' + 
        CONVERT(NVARCHAR(20), GETDATE(), 112) + '.txt';
    
    -- Create inventory directory if it doesn't exist
    EXEC xp_cmdshell 'mkdir "C:\SQLBackups\Inventory"';
    
    -- Generate inventory
    DECLARE @InventoryQuery NVARCHAR(MAX) = 
        'SELECT ''' + REPLICATE('=', 80) + ''' AS Separator, ' +
        '''BACKUP INVENTORY - ' + @DatabaseName + ' - ' + CONVERT(VARCHAR(20), GETDATE(), 120) + ''' AS Header, ' +
        '''Database: ' + @DatabaseName + ''' AS DatabaseInfo, ' +
        '''Backup Type: ' + ISNULL(@BackupType, 'ALL') + ''' AS BackupTypeInfo, ' +
        '''Time Range: Last 7 Days'' AS TimeRange, ' +
        '''''' AS EmptyLine, ' +
        'DatabaseName, BackupType, BackupFile, StartTime, EndTime, ' +
        'BackupSizeMB, Status, LogSpaceBeforePercent, LogSpaceAfterPercent ' +
        'FROM BackupHistory ' +
        'WHERE DatabaseName = ''' + @DatabaseName + '''' +
        CASE WHEN @BackupType IS NOT NULL THEN ' AND BackupType = ''' + @BackupType + '''' ELSE '' END +
        ' AND StartTime >= DATEADD(DAY, -7, GETDATE()) ' +
        'ORDER BY StartTime DESC';
    
    -- Export to file (using BCP or other methods in real implementation)
    PRINT 'Backup inventory generated: ' + @InventoryFile;
END;

-- Grant execute permissions
GRANT EXECUTE ON sp_LogBackupStrategy TO [YourDbaUser];
GRANT EXECUTE ON sp_GenerateBackupInventory TO [YourDbaUser];
```

### Point-in-Time Recovery Planning

**Recovery Planning and Testing:**
```sql
-- Create procedure for recovery planning analysis
CREATE OR ALTER PROCEDURE sp_RecoveryPlanningAnalysis
    @DatabaseName NVARCHAR(128) = NULL,
    @TargetRecoveryTime DATETIME = NULL,
    @RecoveryPoint DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDB NVARCHAR(128) = DB_NAME();
    DECLARE @TargetDB NVARCHAR(128) = ISNULL(@DatabaseName, @CurrentDB);
    DECLARE @LastFullBackup DATETIME;
    DECLARE @LastDifferentialBackup DATETIME;
    DECLARE @LastLogBackup DATETIME;
    DECLARE @FullBackupFile NVARCHAR(500);
    DECLARE @DiffBackupFile NVARCHAR(500);
    DECLARE @LogBackupsToRestore INT;
    DECLARE @TotalRestoreTime INT;
    DECLARE @RTOAchievable BIT = 1;
    
    -- Get latest backup information
    SELECT TOP 1 @LastFullBackup = backup_start_date, @FullBackupFile = bmf.physical_device_name
    FROM msdb.dbo.backupset bs
    INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
    WHERE bs.database_name = @TargetDB AND bs.type = 'D'
    ORDER BY bs.backup_start_date DESC;
    
    SELECT TOP 1 @LastDifferentialBackup = backup_start_date, @DiffBackupFile = bmf.physical_device_name
    FROM msdb.dbo.backupset bs
    INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
    WHERE bs.database_name = @TargetDB AND bs.type = 'I'
    ORDER BY bs.backup_start_date DESC;
    
    SELECT TOP 1 @LastLogBackup = backup_start_date
    FROM msdb.dbo.backupset
    WHERE database_name = @TargetDB AND type = 'L'
    ORDER BY backup_start_date DESC;
    
    -- Calculate transaction logs needed for recovery
    IF @RecoveryPoint IS NOT NULL
    BEGIN
        -- Count log backups needed for point-in-time recovery
        SELECT @LogBackupsToRestore = COUNT(*)
        FROM msdb.dbo.backupset
        WHERE database_name = @TargetDB 
        AND type = 'L'
        AND backup_start_date > ISNULL(@LastDifferentialBackup, @LastFullBackup)
        AND backup_start_date <= @RecoveryPoint;
    END
    ELSE
    BEGIN
        -- Count all log backups since last differential
        SELECT @LogBackupsToRestore = COUNT(*)
        FROM msdb.dbo.backupset
        WHERE database_name = @TargetDB 
        AND type = 'L'
        AND backup_start_date > ISNULL(@LastDifferentialBackup, @LastFullBackup);
    END
    
    -- Estimate restore time (basic calculation - in practice, use historical data)
    DECLARE @DatabaseSizeGB DECIMAL(18,2);
    DECLARE @LogBackupCount INT;
    
    SELECT @DatabaseSizeGB = CAST(SUM(CASE WHEN type = 0 THEN size * 8.0 / 1024 / 1024 ELSE 0 END) AS DECIMAL(18,2))
    FROM sys.database_files
    WHERE database_id = DB_ID(@TargetDB);
    
    SELECT @LogBackupCount = COUNT(*)
    FROM msdb.dbo.backupset
    WHERE database_name = @TargetDB 
    AND backup_start_date >= DATEADD(HOUR, -24, GETDATE());
    
    -- Estimate times based on size and backup count
    DECLARE @FullRestoreTime INT = CAST(@DatabaseSizeGB * 2 AS INT); -- 2 minutes per GB
    DECLARE @DiffRestoreTime INT = CAST(@DatabaseSizeGB * 0.5 AS INT); -- 0.5 minutes per GB
    DECLARE @LogRestoreTime INT = @LogBackupsToRestore * 2; -- 2 minutes per log backup
    
    SET @TotalRestoreTime = @FullRestoreTime + @DiffRestoreTime + @LogRestoreTime;
    
    -- Analyze RTO achievement
    IF @TargetRecoveryTime IS NOT NULL
    BEGIN
        DECLARE @TargetDuration INT = DATEDIFF(MINUTE, GETDATE(), @TargetRecoveryTime);
        SET @RTOAchievable = CASE WHEN @TotalRestoreTime <= @TargetDuration THEN 1 ELSE 0 END;
    END
    
    PRINT '=== Recovery Planning Analysis ===';
    PRINT 'Database: ' + @TargetDB;
    PRINT 'Database Size: ' + CAST(@DatabaseSizeGB AS VARCHAR) + ' GB';
    
    IF @LastFullBackup IS NOT NULL
        PRINT 'Last Full Backup: ' + CAST(@LastFullBackup AS VARCHAR);
    ELSE
        PRINT 'WARNING: No full backup found';
    
    IF @LastDifferentialBackup IS NOT NULL
        PRINT 'Last Differential Backup: ' + CAST(@LastDifferentialBackup AS VARCHAR);
    ELSE
        PRINT 'No differential backup found';
    
    IF @LastLogBackup IS NOT NULL
        PRINT 'Last Log Backup: ' + CAST(@LastLogBackup AS VARCHAR);
    ELSE
        PRINT 'No log backup found';
    
    PRINT 'Log Backups to Restore: ' + CAST(@LogBackupsToRestore AS VARCHAR);
    PRINT 'Estimated Total Restore Time: ' + CAST(@TotalRestoreTime AS VARCHAR) + ' minutes';
    
    IF @TargetRecoveryTime IS NOT NULL
    BEGIN
        DECLARE @TargetDuration INT = DATEDIFF(MINUTE, GETDATE(), @TargetRecoveryTime);
        PRINT 'Target Recovery Time: ' + CAST(@TargetDuration AS VARCHAR) + ' minutes';
        PRINT 'RTO Achievable: ' + CASE WHEN @RTOAchievable = 1 THEN 'YES' ELSE 'NO' END;
        
        IF @RTOAchievable = 0
        BEGIN
            PRINT 'RECOMMENDATIONS:';
            PRINT '- Consider more frequent differential backups';
            PRINT '- Optimize restore procedures';
            PRINT '- Review backup storage location';
        END
    END
    
    -- Generate restore commands
    IF @RecoveryPoint IS NOT NULL
    BEGIN
        PRINT 'Generated Restore Commands for Point-in-Time Recovery:';
        PRINT '';
        PRINT '-- Step 1: Restore Full Backup';
        PRINT 'RESTORE DATABASE ' + @TargetDB + '_Restore';
        PRINT 'FROM DISK = ''' + @FullBackupFile + '''';
        PRINT 'WITH NORECOVERY, REPLACE;';
        PRINT '';
        
        IF @LastDifferentialBackup IS NOT NULL
        BEGIN
            PRINT '-- Step 2: Restore Differential Backup';
            PRINT 'RESTORE DATABASE ' + @TargetDB + '_Restore';
            PRINT 'FROM DISK = ''' + @DiffBackupFile + '''';
            PRINT 'WITH NORECOVERY;';
            PRINT '';
        END
        
        PRINT '-- Step 3: Restore Transaction Log Backups';
        PRINT '-- (You need to restore each log backup in sequence)';
        PRINT '-- Example:';
        PRINT '-- RESTORE LOG ' + @TargetDB + '_Restore';
        PRINT '-- FROM DISK = ''path_to_log_backup.trn''';
        PRINT '-- WITH NORECOVERY;';
        PRINT '';
        PRINT '-- Final log restore with point-in-time recovery';
        PRINT '-- RESTORE LOG ' + @TargetDB + '_Restore';
        PRINT '-- FROM DISK = ''path_to_final_log_backup.trn''';
        PRINT '-- WITH RECOVERY, STOPAT = ''' + CONVERT(VARCHAR(30), @RecoveryPoint, 120) + ''';';
    END
    ELSE
    BEGIN
        PRINT 'Generated Restore Commands for Full Recovery:';
        PRINT '-- Restore to point of last backup (not point-in-time)';
        PRINT '-- RESTORE DATABASE ' + @TargetDB + '_Restore';
        PRINT 'FROM DISK = ''' + @FullBackupFile + '''';
        
        IF @LastDifferentialBackup IS NOT NULL
            PRINT 'RESTORE DATABASE ' + @TargetDB + '_Restore FROM DISK = ''' + @DiffBackupFile + ''' WITH RECOVERY;';
        ELSE
            PRINT 'WITH RECOVERY;';
    END
    
    -- Return analysis results
    SELECT 
        DatabaseName = @TargetDB,
        LastFullBackup = @LastFullBackup,
        LastDifferentialBackup = @LastDifferentialBackup,
        LastLogBackup = @LastLogBackup,
        LogBackupsToRestore = @LogBackupsToRestore,
        DatabaseSizeGB = @DatabaseSizeGB,
        EstimatedRestoreTimeMinutes = @TotalRestoreTime,
        RTOAchievable = @RTOAchievable,
        FullBackupFile = @FullBackupFile,
        DifferentialBackupFile = @DiffBackupFile,
        RecoveryPoint = @RecoveryPoint
END;
GO

-- Grant execute permission
GRANT EXECUTE ON sp_RecoveryPlanningAnalysis TO [YourDbaUser];
```

## Friday - Transaction Log Troubleshooting and Maintenance

### Common Log Issues and Solutions

#### Log File Full Errors

**Diagnostic and Resolution Procedures:**
```sql
-- Create comprehensive log troubleshooting procedure
CREATE OR ALTER PROCEDURE sp_TroubleshootLogFileFull
    @DatabaseName NVARCHAR(128) = NULL,
    @EmergencyMode BIT = 0, -- Set to 1 for emergency actions
    @AlertAdmin BIT = 1
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDB NVARCHAR(128) = DB_NAME();
    DECLARE @TargetDB NVARCHAR(128) = ISNULL(@DatabaseName, @CurrentDB);
    DECLARE @RecoveryModel NVARCHAR(60);
    DECLARE @ActiveLogPercent DECIMAL(5,2);
    DECLARE @ActiveLogMB DECIMAL(18,2);
    DECLARE @TotalLogMB DECIMAL(18,2);
    DECLARE @VLFCount INT;
    DECLARE @ActiveVLFCount INT;
    DECLARE @LongTransactions INT;
    DECLARE @BackupChainIntact BIT = 1;
    DECLARE @LogFileFull BIT = 0;
    DECLARE @Solution NVARCHAR(500);
    DECLARE @SolutionSteps NVARCHAR(MAX);
    
    -- Verify database exists
    IF DB_ID(@TargetDB) IS NULL
        RAISERROR('Database %s does not exist', 16, 1, @TargetDB);
    
    -- Get database information
    SELECT @RecoveryModel = recovery_model_desc 
    FROM sys.databases 
    WHERE name = @TargetDB;
    
    -- Get log space metrics
    SELECT 
        @ActiveLogPercent = 
        CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID(@TargetDB))) / 
             (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1) * 100 AS DECIMAL(5,2)),
        @ActiveLogMB = (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
                        FROM sys.dm_db_log_info(DB_ID(@TargetDB))),
        @TotalLogMB = (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1),
        @VLFCount = (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID(@TargetDB))),
        @ActiveVLFCount = (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID(@TargetDB)) WHERE status = 2);
    
    -- Count long-running transactions
    SELECT @LongTransactions = COUNT(*)
    FROM sys.dm_exec_requests
    WHERE database_id = DB_ID(@TargetDB)
    AND DATEDIFF(SECOND, start_time, GETDATE()) > 300;
    
    -- Check log backup chain integrity
    IF @RecoveryModel IN ('FULL', 'BULK_LOGGED')
    BEGIN
        DECLARE @LastLogBackupTime DATETIME;
        SELECT @LastLogBackupTime = MAX(backup_start_date)
        FROM msdb.dbo.backupset
        WHERE database_name = @TargetDB AND type = 'L';
        
        IF @LastLogBackupTime IS NULL OR DATEDIFF(HOUR, @LastLogBackupTime, GETDATE()) > 2
            SET @BackupChainIntact = 0;
    END
    
    -- Determine if log file is full
    SET @LogFileFull = CASE WHEN @ActiveLogPercent > 95 THEN 1 ELSE 0 END;
    
    PRINT '=== LOG FILE FULL TROUBLESHOOTING ===';
    PRINT 'Database: ' + @TargetDB;
    PRINT 'Current Time: ' + CONVERT(VARCHAR(30), GETDATE(), 120);
    PRINT 'Recovery Model: ' + @RecoveryModel;
    PRINT 'Active Log Usage: ' + CAST(@ActiveLogPercent AS VARCHAR) + '% (' + CAST(@ActiveLogMB AS VARCHAR) + ' MB of ' + CAST(@TotalLogMB AS VARCHAR) + ' MB)';
    PRINT 'VLF Status: ' + CAST(@ActiveVLFCount AS VARCHAR) + ' active of ' + CAST(@VLFCount AS VARCHAR) + ' total VLFs';
    PRINT 'Long Transactions: ' + CAST(@LongTransactions AS VARCHAR);
    PRINT 'Log Backup Chain Intact: ' + CASE WHEN @BackupChainIntact = 1 THEN 'YES' ELSE 'NO' END;
    PRINT 'Log File Status: ' + CASE WHEN @LogFileFull = 1 THEN 'FULL - EMERGENCY ACTION REQUIRED' ELSE 'Not Full' END;
    
    -- Generate solution based on recovery model and situation
    SET @SolutionSteps = '';
    
    IF @LogFileFull = 1 AND @EmergencyMode = 0
    BEGIN
        -- Emergency backup attempt first
        PRINT 'EMERGENCY: Attempting immediate log backup...';
        
        BEGIN TRY
            EXEC sp_TransactionLogBackup @TargetDB, 'C:\SQLBackups\Emergency\', 'LOG', 1, 0;
            PRINT 'Emergency backup completed successfully';
            
            -- Get updated log usage
            SELECT @ActiveLogPercent = 
                CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
                      FROM sys.dm_db_log_info(DB_ID(@TargetDB))) / 
                     (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1) * 100 AS DECIMAL(5,2));
            
            PRINT 'Log usage after emergency backup: ' + CAST(@ActiveLogPercent AS VARCHAR) + '%';
            
            IF @ActiveLogPercent < 80
                PRINT 'Emergency backup resolved the issue';
            
        END TRY
        BEGIN CATCH
            PRINT 'Emergency backup failed: ' + ERROR_MESSAGE();
        END CATCH
    END
    
    -- Continue with troubleshooting based on current status
    IF @ActiveLogPercent > 80
    BEGIN
        IF @RecoveryModel IN ('FULL', 'BULK_LOGGED') AND @BackupChainIntact = 0
        BEGIN
            SET @SolutionSteps = @SolutionSteps + 'ISSUE: Log backup chain broken or no recent log backup' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + 'SOLUTION: Take full backup immediately to reset log chain' + CHAR(13) + CHAR(10);
            
            IF @EmergencyMode = 1
            BEGIN
                DECLARE @FullBackupCmd NVARCHAR(500) = 'BACKUP DATABASE [' + @TargetDB + '] TO DISK = ''C:\SQLBackups\Emergency\' + @TargetDB + '_FULL_' + 
                    CONVERT(NVARCHAR(20), GETDATE(), 112) + '_' + REPLACE(CONVERT(NVARCHAR(20), GETDATE(), 108), ':', '') + 
                    '.bak'' WITH FORMAT, COMPRESSION, CHECKSUM;';
                
                BEGIN TRY
                    EXEC sp_executesql @FullBackupCmd;
                    PRINT 'Full backup completed to reset log chain';
                END TRY
                BEGIN CATCH
                    PRINT 'Full backup failed: ' + ERROR_MESSAGE();
                END CATCH
            END
        END
        
        IF @LongTransactions > 0
        BEGIN
            SET @SolutionSteps = @SolutionSteps + 'ISSUE: Long-running transactions preventing log reuse' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + 'SOLUTION: ' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + '1. Identify long-running queries:' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + '   SELECT session_id, start_time, DATEDIFF(SECOND, start_time, GETDATE()) AS DurationSec' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + '   FROM sys.dm_exec_requests' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + '   WHERE database_id = DB_ID(''' + @TargetDB + ''')' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + '2. Optimize or kill long transactions if appropriate' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + '3. Perform checkpoint after transaction resolution' + CHAR(13) + CHAR(10);
        END
        
        IF @RecoveryModel = 'SIMPLE'
        BEGIN
            SET @SolutionSteps = @SolutionSteps + 'ISSUE: Database in SIMPLE recovery model with high log usage' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + 'SOLUTION: ' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + '1. Check for uncommitted transactions' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + '2. Perform checkpoint: CHECKPOINT;' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + '3. Consider shrinking log file if this is a recurring issue' + CHAR(13) + CHAR(10);
            
            IF @EmergencyMode = 1
            BEGIN
                BEGIN TRY
                    EXEC('CHECKPOINT;');
                    PRINT 'Checkpoint executed';
                END TRY
                BEGIN CATCH
                    PRINT 'Checkpoint failed: ' + ERROR_MESSAGE();
                END CATCH
            END
        END
        
        -- Log file size recommendations
        IF @ActiveLogPercent > 90
        BEGIN
            SET @SolutionSteps = @SolutionSteps + 'ISSUE: Log file size insufficient for workload' + CHAR(13) + CHAR(10);
            SET @SolutionSteps = @SolutionSteps + 'SOLUTION: Increase log file size or number of log files' + CHAR(13) + CHAR(10);
            
            IF @EmergencyMode = 1
            BEGIN
                DECLARE @LogFileName NVARCHAR(128);
                SELECT @LogFileName = name FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1;
                
                DECLARE @ExpandSQL NVARCHAR(500) = 'ALTER DATABASE ' + QUOTENAME(@TargetDB) + ' ' +
                    'MODIFY FILE (NAME = ' + QUOTENAME(@LogFileName) + ', SIZE = ' + CAST(@TotalLogMB * 2 AS VARCHAR) + 'MB);';
                
                BEGIN TRY
                    EXEC sp_executesql @ExpandSQL;
                    PRINT 'Log file expanded to ' + CAST(@TotalLogMB * 2 AS VARCHAR) + 'MB';
                END TRY
                BEGIN CATCH
                    PRINT 'Log file expansion failed: ' + ERROR_MESSAGE();
                END CATCH
            END
        END
    END
    
    -- If log is still full after attempts, suggest emergency procedures
    IF @ActiveLogPercent > 95 AND @EmergencyMode = 1
    BEGIN
        SET @SolutionSteps = @SolutionSteps + 'EMERGENCY PROCEDURES:' + CHAR(13) + CHAR(10);
        SET @SolutionSteps = @SolutionSteps + '1. Switch database to SIMPLE recovery model (if acceptable):' + CHAR(13) + CHAR(10);
        SET @SolutionSteps = @SolutionSteps + '   ALTER DATABASE ' + QUOTENAME(@TargetDB) + ' SET RECOVERY SIMPLE;' + CHAR(13) + CHAR(10);
        SET @SolutionSteps = @SolutionSteps + '2. Shrink log file:' + CHAR(13) + CHAR(10);
        SET @SolutionSteps = @SolutionSteps + '   DBCC SHRINKFILE(' + QUOTENAME((SELECT name FROM sys.database_files WHERE database_id = DB_ID(@TargetDB) AND type = 1)) + ', 1);' + CHAR(13) + CHAR(10);
        SET @SolutionSteps = @SolutionSteps + '3. Switch back to FULL recovery model and take full backup' + CHAR(13) + CHAR(10);
        SET @SolutionSteps = @SolutionSteps + '4. Re-establish log backup chain' + CHAR(13) + CHAR(10);
    END
    
    -- Log troubleshooting results
    INSERT INTO LogTroubleshootingLog (DatabaseName, ActiveLogPercent, ActiveLogMB, TotalLogMB, VLFCount, ActiveVLFCount, LongTransactionCount, RecoveryModel, BackupChainIntact, LogFileFull, SolutionSteps, TroubleshootingTime)
    VALUES (@TargetDB, @ActiveLogPercent, @ActiveLogMB, @TotalLogMB, @VLFCount, @ActiveVLFCount, @LongTransactions, @RecoveryModel, @BackupChainIntact, @LogFileFull, @SolutionSteps, GETDATE());
    
    -- Display recommendations
    IF @SolutionSteps != ''
    BEGIN
        PRINT '';
        PRINT 'TROUBLESHOOTING RECOMMENDATIONS:';
        PRINT @SolutionSteps;
    END
    ELSE
    BEGIN
        PRINT 'Log space usage is within acceptable limits';
    END
    
    -- Send alert if configured
    IF @AlertAdmin = 1 AND (@ActiveLogPercent > 80 OR @LogFileFull = 1)
    BEGIN
        PRINT 'Alert: High transaction log usage detected on ' + @TargetDB;
        -- In real implementation, send email alert here
    END
    
    -- Return current status
    SELECT 
        DatabaseName = @TargetDB,
        ActiveLogPercent = @ActiveLogPercent,
        LogFileStatus = CASE WHEN @LogFileFull = 1 THEN 'FULL' ELSE 'OK' END,
        RecoveryModel = @RecoveryModel,
        LongTransactionCount = @LongTransactions,
        BackupChainIntact = @BackupChainIntact,
        EmergencyActionRequired = CASE WHEN @ActiveLogPercent > 95 THEN 1 ELSE 0 END,
        RecommendationsAvailable = CASE WHEN @SolutionSteps != '' THEN 1 ELSE 0 END
END;
GO

-- Create troubleshooting log table
CREATE TABLE LogTroubleshootingLog (
    TroubleshootingID INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(128),
    ActiveLogPercent DECIMAL(5,2),
    ActiveLogMB DECIMAL(18,2),
    TotalLogMB DECIMAL(18,2),
    VLFCount INT,
    ActiveVLFCount INT,
    LongTransactionCount INT,
    RecoveryModel NVARCHAR(60),
    BackupChainIntact BIT,
    LogFileFull BIT,
    SolutionSteps NVARCHAR(MAX),
    TroubleshootingTime DATETIME DEFAULT GETDATE()
);

-- Grant execute permission
GRANT EXECUTE ON sp_TroubleshootLogFileFull TO [YourDbaUser];
```

### PowerShell Log Management Automation

```powershell
# Comprehensive Transaction Log Management Script
# Save as: Manage-TransactionLogs.ps1

param(
    [Parameter(Mandatory=$true)]
    [string]$ServerInstance,
    
    [Parameter(Mandatory=$false)]
    [string[]]$DatabaseNames,
    
    [Parameter(Mandatory=$false)]
    [string]$BackupRootPath = 'C:\SQLBackups\TransactionLogs',
    
    [Parameter(Mandatory=$false)]
    [int]$BackupFrequencyMinutes = 15,
    
    [Parameter(Mandatory=$false)]
    [int]$RetentionHours = 72,
    
    [Parameter(Mandatory=$false)]
    [decimal]$AlertThresholdPercent = 80.0,
    
    [Parameter(Mandatory=$false)]
    [switch]$EmergencyMode,
    
    [Parameter(Mandatory=$false)]
    [switch]$GenerateReports
)

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
    
    $logPath = "C:\Logs\TransactionLogManagement_$(Get-Date -Format 'yyyyMMdd').log"
    if (-not (Test-Path (Split-Path $logPath))) {
        New-Item -ItemType Directory -Path (Split-Path $logPath) -Force | Out-Null
    }
    Add-Content -Path $logPath -Value $logMessage
}

function Test-DatabaseConnection {
    param([string]$Server, [string]$Database = 'master')
    try {
        $connectionString = "Server=$Server;Database=$Database;Integrated Security=True;Connection Timeout=10;"
        $connection = New-Object System.Data.SqlClient.SqlConnection($connectionString)
        $connection.Open()
        $connection.Close()
        return $true
    }
    catch {
        Write-Log "Failed to connect to SQL Server: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

function Get-TransactionLogStatus {
    param([string]$Server, [string]$Database)
    
    $query = @"
    SELECT 
        DB_NAME() AS DatabaseName,
        recovery_model_desc AS RecoveryModel,
        (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) AS TotalLogSizeMB,
        (SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
         FROM sys.dm_db_log_info(DB_ID())) AS ActiveLogSizeMB,
        CAST((SELECT SUM(CASE WHEN status = 2 THEN size * 8.0 / 1024 ELSE 0 END) 
              FROM sys.dm_db_log_info(DB_ID())) / 
             (SELECT SUM(size * 8.0 / 1024) FROM sys.database_files WHERE type = 1) * 100 AS DECIMAL(5,2)) AS ActiveLogPercent,
        (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID())) AS TotalVLFs,
        (SELECT COUNT(*) FROM sys.dm_db_log_info(DB_ID()) WHERE status = 2) AS ActiveVLFs
    FROM sys.databases
    WHERE name = '$Database'
"@
    
    try {
        $connectionString = "Server=$Server;Database=master;Integrated Security=True;"
        $connection = New-Object System.Data.SqlClient.SqlConnection($connectionString)
        $command = $connection.CreateCommand()
        $command.CommandText = $query
        $connection.Open()
        
        $adapter = New-Object System.Data.SqlClient.SqlDataAdapter($command)
        $dataset = New-Object System.Data.DataSet
        $adapter.Fill($dataset) | Out-Null
        
        $connection.Close()
        return $dataset.Tables[0]
    }
    catch {
        Write-Log "Failed to get transaction log status for $Database`: $($_.Exception.Message)" 'ERROR'
        return $null
    }
}

function Start-TransactionLogBackup {
    param(
        [string]$Server,
        [string]$Database,
        [string]$BackupPath,
        [int]$RetentionHours,
        [bool]$Compress = $true
    )
    
    $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    $backupFile = "$BackupPath\$Database`_LOG`_$timestamp.trn"
    
    # Ensure backup directory exists
    if (-not (Test-Path $BackupPath)) {
        New-Item -ItemType Directory -Path $BackupPath -Force | Out-Null
    }
    
    $backupQuery = @"
    BACKUP LOG [$Database]
    TO DISK = '$backupFile'
    WITH COMPRESSION, NO_TRUNCATE, STATS = 10
"@
    
    if (-not $Compress) {
        $backupQuery = $backupQuery -replace ", COMPRESSION", ""
    }
    
    try {
        Write-Log "Starting transaction log backup for $Database to $backupFile"
        
        $connectionString = "Server=$Server;Database=master;Integrated Security=True;"
        $connection = New-Object System.Data.SqlClient.SqlConnection($connectionString)
        $command = $connection.CreateCommand()
        $command.CommandText = $backupQuery
        $command.CommandTimeout = 0  # No timeout for backups
        $connection.Open()
        $command.ExecuteNonQuery() | Out-Null
        $connection.Close()
        
        Write-Log "Transaction log backup completed successfully for $Database"
        
        # Clean up old backups
        $cutoffDate = (Get-Date).AddHours(-$RetentionHours)
        Get-ChildItem -Path $BackupPath -Filter "$Database`_LOG`*.trn" | Where-Object {
            $_.LastWriteTime -lt $cutoffDate
        } | Remove-Item -Force
        
        return $backupFile
    }
    catch {
        Write-Log "Transaction log backup failed for $Database`: $($_.Exception.Message)" 'ERROR'
        return $null
    }
}

function Test-TransactionLogFull {
    param([string]$Server, [string]$Database, [decimal]$ThresholdPercent)
    
    $logStatus = Get-TransactionLogStatus -Server $Server -Database $Database
    
    if ($logStatus -and $logStatus.Rows.Count -gt 0) {
        $row = $logStatus.Rows[0]
        $activeLogPercent = $row.ActiveLogPercent
        $recoveryModel = $row.RecoveryModel
        $activeLogSizeMB = $row.ActiveLogSizeMB
        $totalLogSizeMB = $row.TotalLogSizeMB
        
        # Check if log is full or near full
        $isFull = $activeLogPercent -gt $ThresholdPercent
        $isHighUsage = $activeLogPercent -gt ($ThresholdPercent * 0.8)
        
        # Determine action needed
        $action = switch ($activeLogPercent) {
            { $_ -gt 95 } { 'EMERGENCY_BACKUP_OR_SHRINK' }
            { $_ -gt $ThresholdPercent } { 'IMMEDIATE_BACKUP_REQUIRED' }
            { $_ -gt ($ThresholdPercent * 0.8) } { 'BACKUP_SOON' }
            default { 'MONITOR' }
        }
        
        return @{
            DatabaseName = $Database
            RecoveryModel = $recoveryModel
            ActiveLogPercent = $activeLogPercent
            ActiveLogSizeMB = $activeLogSizeMB
            TotalLogSizeMB = $totalLogSizeMB
            IsFull = $isFull
            IsHighUsage = $isHighUsage
            Action = $action
            RequiresBackup = $recoveryModel -in @('FULL', 'BULK_LOGGED') -and $isHighUsage
        }
    }
    
    return $null
}

function Send-LogAlert {
    param(
        [string]$Database,
        [decimal]$ActiveLogPercent,
        [string]$Action,
        [string]$Server
    )
    
    $emailBody = @"
    SQL Server Transaction Log Alert
    
    Server: $Server
    Database: $Database
    Active Log Usage: $([math]::Round($ActiveLogPercent, 2))%
    Action Required: $Action
    Timestamp: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')
    
    Please review the transaction log status and take appropriate action.
    "@
    
    Write-Log "ALERT: $Database log usage is $([math]::Round($ActiveLogPercent, 2))% - Action: $Action" 'WARNING'
    
    # In a real implementation, you would send email here
    # Example: Send-MailMessage -To "dba@company.com" -Subject "Transaction Log Alert - $Database" -Body $emailBody -SmtpServer "smtp.company.com"
}

function Start-TransactionLogOptimization {
    param([string]$Server, [string]$Database)
    
    $optimizationQuery = @"
    EXEC sp_OptimizeLogFile @DatabaseName = '$Database'
    "@
    
    try {
        Write-Log "Starting log file optimization for $Database"
        
        $connectionString = "Server=$Server;Database=master;Integrated Security=True;"
        $connection = New-Object System.Data.SqlClient.SqlConnection($connectionString)
        $command = $connection.CreateCommand()
        $command.CommandText = $optimizationQuery
        $connection.Open()
        $command.ExecuteNonQuery() | Out-Null
        $connection.Close()
        
        Write-Log "Log file optimization completed for $Database"
        return $true
    }
    catch {
        Write-Log "Log file optimization failed for $Database`: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

# Main execution
try {
    Write-Log "Starting transaction log management for $ServerInstance"
    
    # Test connection
    if (-not (Test-DatabaseConnection -Server $ServerInstance)) {
        exit 1
    }
    
    # Get list of databases to manage
    if (-not $DatabaseNames) {
        $databaseQuery = "SELECT name FROM sys.databases WHERE database_id > 4 AND state = 0"
        $connectionString = "Server=$ServerInstance;Database=master;Integrated Security=True;"
        $connection = New-Object System.Data.SqlClient.SqlConnection($connectionString)
        $command = $connection.CreateCommand()
        $command.CommandText = $databaseQuery
        $connection.Open()
        
        $adapter = New-Object System.Data.SqlClient.SqlDataAdapter($command)
        $dataset = New-Object System.Data.DataSet
        $adapter.Fill($dataset) | Out-Null
        $connection.Close()
        
        $DatabaseNames = $dataset.Tables[0] | Select-Object -ExpandProperty name
    }
    
    $managementResults = @()
    $totalDatabases = $DatabaseNames.Count
    $alertedDatabases = 0
    $backedUpDatabases = 0
    $optimizedDatabases = 0
    
    foreach ($database in $DatabaseNames) {
        Write-Log "Processing database: $database"
        
        # Check transaction log status
        $logTest = Test-TransactionLogFull -Server $ServerInstance -Database $database -ThresholdPercent $AlertThresholdPercent
        
        if ($logTest) {
            $managementResults += $logTest
            
            # Send alert if needed
            if ($logTest.IsFull -or $logTest.IsHighUsage) {
                Send-LogAlert -Database $database -ActiveLogPercent $logTest.ActiveLogPercent -Action $logTest.Action -Server $ServerInstance
                $alertedDatabases++
            }
            
            # Perform backup if needed
            if ($logTest.RequiresBackup -or $EmergencyMode) {
                $backupPath = Join-Path $BackupRootPath $database
                $backupFile = Start-TransactionLogBackup -Server $ServerInstance -Database $database -BackupPath $backupPath -RetentionHours $RetentionHours
                
                if ($backupFile) {
                    $backedUpDatabases++
                    
                    # Check log space after backup
                    Start-Sleep -Seconds 5  # Allow time for log to be truncated
                    $logStatusAfter = Get-TransactionLogStatus -Server $ServerInstance -Database $database
                    
                    if ($logStatusAfter -and $logStatusAfter.Rows.Count -gt 0) {
                        $logPercentAfter = $logStatusAfter.Rows[0].ActiveLogPercent
                        Write-Log "Log usage after backup: $([math]::Round($logPercentAfter, 2))%"
                    }
                }
            }
            
            # Optimize log file if high VLF count
            $logStatus = Get-TransactionLogStatus -Server $ServerInstance -Database $database
            if ($logStatus -and $logStatus.Rows.Count -gt 0) {
                $vlfCount = $logStatus.Rows[0].TotalVLFs
                
                if ($vlfCount -gt 100) {
                    Write-Log "High VLF count detected ($vlfCount) - triggering optimization"
                    $optimizationSuccess = Start-TransactionLogOptimization -Server $ServerInstance -Database $database
                    if ($optimizationSuccess) {
                        $optimizedDatabases++
                    }
                }
            }
        }
    }
    
    # Generate summary report
    Write-Log "=== Transaction Log Management Summary ==="
    Write-Log "Total databases processed: $totalDatabases"
    Write-Log "Databases with alerts: $alertedDatabases"
    Write-Log "Databases backed up: $backedUpDatabases"
    Write-Log "Databases optimized: $optimizedDatabases"
    
    # Generate detailed report if requested
    if ($GenerateReports) {
        $reportFile = "C:\Reports\TransactionLogManagement_$(Get-Date -Format 'yyyyMMdd_HHmmss').html"
        $reportPath = Split-Path $reportFile
        if (-not (Test-Path $reportPath)) {
            New-Item -ItemType Directory -Path $reportPath -Force | Out-Null
        }
        
        $htmlReport = @"
        <!DOCTYPE html>
        <html>
        <head>
            <title>Transaction Log Management Report</title>
            <style>
                body { font-family: Arial, sans-serif; margin: 20px; }
                .header { background-color: #f0f0f0; padding: 10px; border-bottom: 2px solid #ccc; }
                .alert { background-color: #ffebee; padding: 10px; margin: 10px 0; border-left: 4px solid #f44336; }
                .warning { background-color: #fff3e0; padding: 10px; margin: 10px 0; border-left: 4px solid #ff9800; }
                .success { background-color: #e8f5e8; padding: 10px; margin: 10px 0; border-left: 4px solid #4caf50; }
                table { border-collapse: collapse; width: 100%; margin: 20px 0; }
                th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
                th { background-color: #f2f2f2; }
                .critical { color: red; font-weight: bold; }
                .warning { color: orange; }
                .normal { color: green; }
            </style>
        </head>
        <body>
            <div class="header">
                <h1>Transaction Log Management Report</h1>
                <p>Generated: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')</p>
                <p>Server: $ServerInstance</p>
            </div>
            
            <div class="success">
                <h2>Executive Summary</h2>
                <p><strong>Total Databases:</strong> $totalDatabases</p>
                <p><strong>Alerts Generated:</strong> $alertedDatabases</p>
                <p><strong>Backups Performed:</strong> $backedUpDatabases</p>
                <p><strong>Optimizations Performed:</strong> $optimizedDatabases</p>
            </div>
            
            <h2>Database Status Details</h2>
            <table>
                <tr>
                    <th>Database</th>
                    <th>Recovery Model</th>
                    <th>Active Log %</th>
                    <th>Log Size (MB)</th>
                    <th>VLFs</th>
                    <th>Status</th>
                    <th>Action Required</th>
                </tr>
        "@
        
        foreach ($result in $managementResults) {
            $statusClass = if ($result.ActiveLogPercent -gt 80) { 'critical' } elseif ($result.ActiveLogPercent -gt 60) { 'warning' } else { 'normal' }
            $statusColor = if ($result.ActiveLogPercent -gt 80) { 'red' } elseif ($result.ActiveLogPercent -gt 60) { 'orange' } else { 'green' }
            
            $htmlReport += @"
                <tr>
                    <td>$($result.DatabaseName)</td>
                    <td>$($result.RecoveryModel)</td>
                    <td class="$statusClass">$([math]::Round($result.ActiveLogPercent, 2))%</td>
                    <td>$([math]::Round($result.ActiveLogSizeMB, 1))</td>
                    <td>$($result.ActiveVLFs)/$((Get-TransactionLogStatus -Server $ServerInstance -Database $result.DatabaseName).TotalVLFs)</td>
                    <td style="color: $statusColor">$(
                        if ($result.ActiveLogPercent -gt 95) { 'FULL' }
                        elseif ($result.ActiveLogPercent -gt 80) { 'HIGH' }
                        elseif ($result.ActiveLogPercent -gt 60) { 'WARNING' }
                        else { 'NORMAL' }
                    )</td>
                    <td>$($result.Action)</td>
                </tr>
        "@
        }
        
        $htmlReport += @"
            </table>
        </body>
        </html>
        "@
        
        $htmlReport | Out-File -FilePath $reportFile -Encoding UTF8
        Write-Log "Detailed report saved to: $reportFile"
    }
    
    Write-Log "Transaction log management completed successfully"
}
catch {
    Write-Log "Transaction log management failed: $($_.Exception.Message)" 'ERROR'
    exit 1
}
```

### Hands-On Lab: Complete Transaction Log Management System

**Scenario**: Implement a comprehensive transaction log management system for a multi-database production environment.

**Tasks**:
1. Analyze current transaction log usage patterns across all databases
2. Implement optimal recovery model strategies for each database type
3. Create automated log backup procedures with frequency optimization
4. Set up transaction log monitoring and alerting system
5. Implement log file optimization and VLF management
6. Create comprehensive disaster recovery testing procedures
7. Develop PowerShell automation for complete log management lifecycle

**Deliverables**:
- Transaction log analysis report with optimization recommendations
- Automated backup scripts with intelligent scheduling
- Monitoring and alerting system configuration
- Log file optimization procedures
- Disaster recovery testing results and procedures
- PowerShell automation suite for complete log management

## Summary

This week covered comprehensive transaction log management for SQL Server databases. You learned to:

- Understand SQL Server transaction log architecture and internal mechanisms
- Configure and manage different recovery models (Full, Simple, Bulk-Logged)
- Implement optimal checkpoint processes and log file configuration
- Design and execute transaction log backup strategies
- Monitor transaction log space usage and performance impact
- Troubleshoot common log issues including log file full errors
- Automate transaction log management using PowerShell and stored procedures
- Plan and test disaster recovery scenarios with point-in-time recovery

Mastering these transaction log management skills will ensure you can maintain database integrity, optimize performance, and implement robust disaster recovery solutions in production environments.