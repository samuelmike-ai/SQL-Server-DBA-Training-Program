# Week 11: Database Maintenance Plans

## Table of Contents
1. [Introduction to Maintenance Plans](#introduction)
2. [Maintenance Workflows](#maintenance-workflows)
3. [Automated Backup Tasks](#automated-backup-tasks)
4. [Index Maintenance Tasks](#index-maintenance-tasks)
5. [Database Integrity Checks](#database-integrity-checks)
6. [Statistics Updates](#statistics-updates)
7. [Cleanup Tasks](#cleanup-tasks)
8. [Health Monitoring](#health-monitoring)
9. [Custom Maintenance Scripts](#custom-maintenance-scripts)
10. [Troubleshooting Scenarios](#troubleshooting-scenarios)
11. [Best Practices](#best-practices)
12. [Practical Exercises](#practical-exercises)

## Introduction to Maintenance Plans {#introduction}

Database maintenance plans are systematic approaches to ensuring SQL Server databases remain healthy, perform optimally, and are protected from data loss. These plans are essential for database administrators (DBAs) to establish routine procedures that proactively address potential issues before they become critical problems.

A comprehensive maintenance plan encompasses various tasks including backup strategies, index optimization, integrity checks, statistics updates, and cleanup procedures. When properly designed and implemented, maintenance plans reduce downtime, improve performance, and provide peace of mind that your databases are operating at peak efficiency.

Modern SQL Server provides several tools for implementing maintenance plans: SQL Server Maintenance Plans (graphical interface), T-SQL scripts, PowerShell cmdlets, and custom solutions. Each approach offers different levels of control, flexibility, and complexity. The key is to select the right combination based on your specific requirements, environment complexity, and operational constraints.

This week focuses on building robust maintenance workflows that can adapt to changing business needs while maintaining consistent, reliable database health. We'll explore both built-in SQL Server maintenance plan features and advanced customization techniques that go beyond the standard offerings.

## Maintenance Workflows {#maintenance-workflows}

### Understanding Workflow Components

A successful maintenance workflow consists of multiple interconnected components that work together to maintain database health. Understanding these components and their relationships is crucial for designing effective maintenance strategies.

**Workflow Design Principles:**

1. **Logical Task Sequencing**: Maintenance tasks must be ordered based on dependencies and resource requirements
2. **Resource Optimization**: Tasks should be scheduled to minimize impact on production operations
3. **Error Handling**: Robust error detection, logging, and recovery mechanisms
4. **Scalability**: Plans must accommodate growing databases and changing requirements
5. **Monitoring Integration**: Workflows should feed into broader monitoring and alerting systems

### Task Dependency Analysis

Proper workflow design begins with analyzing task dependencies. Some tasks must complete before others can begin, while others can run in parallel. Consider the following dependencies:

**Sequential Dependencies:**
- Database integrity checks before backup operations
- Index rebuilds before statistics updates
- Cleanup operations before new backup creation

**Parallel Opportunities:**
- Index reorganization can run alongside statistics updates
- Multiple database integrity checks can be performed concurrently
- File cleanup and log file management can run simultaneously

### Creating Maintenance Workflow Templates

Let me demonstrate a comprehensive maintenance workflow template:

```sql
-- Maintenance Workflow Template
USE msdb;
GO

-- Create maintenance plan job categories
IF NOT EXISTS (SELECT 1 FROM msdb.dbo.syscategories WHERE name = N'Database Maintenance' AND category_class = 1)
BEGIN
    EXEC msdb.dbo.sp_add_category 
        @class = N'JOB', 
        @type = N'LOCAL', 
        @name = N'Database Maintenance';
END
GO

-- Sample weekly maintenance workflow
DECLARE @PlanID uniqueidentifier;
DECLARE @PlanName nvarchar(128) = N'Weekly Database Maintenance - Full';

EXEC dbo.sp_add_maintenance_plan 
    @plan_name = @PlanName,
    @plan_id = @PlanID OUTPUT;

-- Add database tasks to the plan
EXEC dbo.sp_add_maintenanceplan_db 
    @plan_id = @PlanID,
    @db_name = N'MyDatabase';

-- Add maintenance plan tasks
EXEC dbo.sp_add_maintenanceplan_task 
    @plan_id = @PlanID,
    @task_id = 1,
    @task_name = N'Check Database Integrity',
    @task_type = 1, -- Integrity check
    @task_order = 1,
    @command = N'DBCC CHECKDB WITH PHYSICAL_ONLY, EXTENDED_LOGICAL_CHECKS';

EXEC dbo.sp_add_maintenanceplan_task 
    @plan_id = @PlanID,
    @task_id = 2,
    @task_name = N'Rebuild Indexes',
    @task_type = 4, -- Index optimization
    @task_order = 2,
    @command = N'ALTER INDEX ALL ON dbo.YourTable REBUILD WITH (ONLINE = ON (WAIT_AT_LOW_PRIORITY (MAX_DURATION = 10 MINUTES, ABORT_AFTER_WAIT = SELF)))';

-- Schedule the plan
EXEC dbo.sp_add_schedule 
    @schedule_name = N'Weekly Maintenance Schedule',
    @enabled = 1,
    @freq_type = 8, -- Weekly
    @freq_interval = 1, -- Sunday
    @freq_subday_type = 1, -- Once per day
    @freq_subday_interval = 0,
    @active_start_date = 20240101,
    @active_end_date = 99991231,
    @active_start_time = 20000, -- 8:00 PM
    @active_end_time = 235959;

-- Attach schedule to plan
EXEC dbo.sp_attach_schedule 
    @plan_name = @PlanName,
    @schedule_name = N'Weekly Maintenance Schedule';

-- Add job to maintenance plan
EXEC dbo.sp_add_maintenanceplan_job 
    @plan_id = @PlanID,
    @job_id = NULL;
```

This template demonstrates the structure of a proper maintenance workflow, including task sequencing, scheduling, and error handling considerations.

## Automated Backup Tasks {#automated-backup-tasks}

### Backup Strategy Integration

Automated backup tasks form the cornerstone of any maintenance plan. Effective backup automation ensures data protection while minimizing operational overhead. This section covers advanced backup automation techniques that go beyond basic backup scheduling.

**Key Backup Automation Components:**

1. **Dynamic Backup Creation**: Scripts that automatically adjust backup operations based on database state
2. **Retention Management**: Automated cleanup of old backup files
3. **Backup Verification**: Automated testing of backup integrity
4. **Notification Systems**: Alerts for backup failures and warnings

### Advanced Backup Automation Script

Here's a comprehensive backup automation script with intelligent features:

```sql
USE msdb;
GO

-- Create backup directory if it doesn't exist
DECLARE @BackupDirectory nvarchar(500) = N'C:\SQLBackups\Automated\';
DECLARE @CreateDirSQL nvarchar(max) = N'EXECUTE xp_cmdshell ''mkdir "' + @BackupDirectory + N'"''';

-- Skip directory creation for security reasons in production

-- Create backup procedure
IF NOT EXISTS (SELECT 1 FROM sys.procedures WHERE name = 'sp_AutomatedBackup')
BEGIN
    EXEC('
    CREATE PROCEDURE sp_AutomatedBackup
    (
        @DatabaseName sysname = NULL,
        @BackupType nvarchar(10) = ''FULL'', -- FULL, DIFFERENTIAL, LOG
        @RetentionDays int = 30,
        @Compress bit = 1,
        @Verify bit = 1,
        @BackupPath nvarchar(500) = NULL
    )
    AS
    BEGIN
        SET NOCOUNT ON;
        
        DECLARE @SQL nvarchar(max);
        DECLARE @BackupFile nvarchar(500);
        DECLARE @BackupDate nvarchar(20) = CONVERT(nvarchar(20), GETDATE(), 120);
        DECLARE @BackupTime nvarchar(20) = REPLACE(REPLACE(CONVERT(nvarchar(20), GETDATE(), 108), '':'', ''_''), '' '', '''');
        
        -- Handle NULL backup path
        IF @BackupPath IS NULL
            SET @BackupPath = ''C:\SQLBackups\Automated\'';
        
        -- Create backup path if it doesn't exist (simplified)
        DECLARE @CreatePath nvarchar(1000) = ''EXEC xp_cmdshell '' + CHAR(39) + ''mkdir "'' + @BackupPath + ''"'' + CHAR(39);
        -- EXECUTE sp_executesql @CreatePath; -- Uncomment if xp_cmdshell is enabled
        
        DECLARE @ErrorLog TABLE (ErrorNumber int, ErrorMessage nvarchar(4000));
        
        IF @DatabaseName IS NULL
        BEGIN
            -- Backup all user databases
            DECLARE @DBName sysname;
            DECLARE db_cursor CURSOR FOR
            SELECT name 
            FROM sys.databases 
            WHERE database_id > 4 
            AND state = 0 -- ONLINE
            AND name NOT IN (''ReportServer'', ''ReportServerTempDB'')
            ORDER BY name;
            
            OPEN db_cursor;
            FETCH NEXT FROM db_cursor INTO @DBName;
            
            WHILE @@FETCH_STATUS = 0
            BEGIN
                BEGIN TRY
                    IF @BackupType = ''LOG''
                    BEGIN
                        -- Log backup - only for databases in FULL or BULK_LOGGED recovery model
                        IF EXISTS (SELECT 1 FROM sys.databases WHERE name = @DBName AND recovery_model IN (1, 2))
                        BEGIN
                            SET @BackupFile = @BackupPath + @DBName + ''_'' + @BackupDate + ''_'' + @BackupTime + ''_LOG.trn'';
                            SET @SQL = N''BACKUP LOG ['' + @DBName + N''] TO DISK = '''''' + @BackupFile + '''''' 
                                      + CASE WHEN @Compress = 1 THEN N'' WITH COMPRESSION'' ELSE N'''' END
                                      + N'', CHECKSUM, STATS = 5'';
                            
                            EXEC sp_executesql @SQL;
                            
                            -- Verify backup if requested
                            IF @Verify = 1
                            BEGIN
                                SET @SQL = N''RESTORE VERIFYONLY FROM DISK = '''''' + @BackupFile + '''''' '';
                                EXEC sp_executesql @SQL;
                            END
                        END
                    END
                    ELSE
                    BEGIN
                        -- Full or differential backup
                        SET @BackupFile = @BackupPath + @DBName + ''_'' + @BackupDate + ''_'' + @BackupTime + ''_'' + @BackupType + ''.bak'';
                        SET @SQL = N''BACKUP DATABASE ['' + @DBName + N''] TO DISK = '''''' + @BackupFile + '''''' 
                                  + CASE WHEN @Compress = 1 THEN N'' WITH COMPRESSION'' ELSE N'''' END
                                  + N'', CHECKSUM, STATS = 5'';
                        
                        IF @BackupType = ''DIFFERENTIAL''
                            SET @SQL = @SQL + N'', DIFFERENTIAL'';
                        
                        EXEC sp_executesql @SQL;
                        
                        -- Verify backup if requested
                        IF @Verify = 1
                        BEGIN
                            SET @SQL = N''RESTORE VERIFYONLY FROM DISK = '''''' + @BackupFile + '''''' '';
                            EXEC sp_executesql @SQL;
                        END
                    END
                    
                    PRINT ''Successfully backed up '' + @DBName + '' ('' + @BackupType + '' backup)'';
                    
                END TRY
                BEGIN CATCH
                    INSERT INTO @ErrorLog (ErrorNumber, ErrorMessage)
                    VALUES (ERROR_NUMBER(), ''Database: '' + @DBName + '' - '' + ERROR_MESSAGE());
                END CATCH
                
                FETCH NEXT FROM db_cursor INTO @DBName;
            END
            
            CLOSE db_cursor;
            DEALLOCATE db_cursor;
        END
        ELSE
        BEGIN
            -- Backup specific database
            BEGIN TRY
                IF @BackupType = ''LOG'' AND EXISTS (SELECT 1 FROM sys.databases WHERE name = @DatabaseName AND recovery_model IN (1, 2))
                BEGIN
                    SET @BackupFile = @BackupPath + @DatabaseName + ''_'' + @BackupDate + ''_'' + @BackupTime + ''_LOG.trn'';
                    SET @SQL = N''BACKUP LOG ['' + @DatabaseName + N''] TO DISK = '''''' + @BackupFile + '''''' 
                              + CASE WHEN @Compress = 1 THEN N'' WITH COMPRESSION'' ELSE N''''
                              + N'', CHECKSUM, STATS = 5'';
                    EXEC sp_executesql @SQL;
                END
                ELSE
                BEGIN
                    SET @BackupFile = @BackupPath + @DatabaseName + ''_'' + @BackupDate + ''_'' + @BackupTime + ''_'' + @BackupType + ''.bak'';
                    SET @SQL = N''BACKUP DATABASE ['' + @DatabaseName + N''] TO DISK = '''''' + @BackupFile + '''''' 
                              + CASE WHEN @Compress = 1 THEN N'' WITH COMPRESSION'' ELSE N''''
                              + N'', CHECKSUM, STATS = 5'';
                    
                    IF @BackupType = ''DIFFERENTIAL''
                        SET @SQL = @SQL + N'', DIFFERENTIAL'';
                    
                    EXEC sp_executesql @SQL;
                END
                
                -- Verify backup if requested
                IF @Verify = 1
                BEGIN
                    SET @SQL = N''RESTORE VERIFYONLY FROM DISK = '''''' + @BackupFile + '''''' '';
                    EXEC sp_executesql @SQL;
                END
                
                PRINT ''Successfully backed up '' + @DatabaseName + '' ('' + @BackupType + '' backup)'';
                
            END TRY
            BEGIN CATCH
                INSERT INTO @ErrorLog (ErrorNumber, ErrorMessage)
                VALUES (ERROR_NUMBER(), ''Database: '' + @DatabaseName + '' - '' + ERROR_MESSAGE());
            END CATCH
        END
        
        -- Clean up old backup files
        DECLARE @DeleteBeforeDate datetime = DATEADD(day, -@RetentionDays, GETDATE());
        DECLARE @DeleteSQL nvarchar(max);
        DECLARE @FileName nvarchar(500);
        
        DECLARE file_cursor CURSOR FOR
        SELECT filename
        FROM (
            SELECT ''C:\SQLBackups\Automated\'' + name + ''.bak'' AS filename
            FROM tempdb.dbo.sysfiles
            WHERE 1 = 0
        ) AS Files; -- This would be replaced with actual file enumeration logic
        
        -- Note: File cleanup would require xp_cmdshell or other file system access
        
        -- Return results
        IF EXISTS (SELECT 1 FROM @ErrorLog)
        BEGIN
            SELECT ErrorNumber, ErrorMessage FROM @ErrorLog;
            RETURN 1; -- Indicate errors occurred
        END
        ELSE
        BEGIN
            PRINT ''All backup operations completed successfully.'';
            RETURN 0;
        END
    END
    ');
END
GO

-- Create jobs to run the backup procedure
DECLARE @JobName nvarchar(128);
DECLARE @ScheduleName nvarchar(128);

-- Full backup job - runs daily at 2 AM
SET @JobName = N'Daily Full Backup';
SET @ScheduleName = N'Daily 2 AM Full Backup';

IF NOT EXISTS (SELECT 1 FROM msdb.dbo.sysjobs WHERE name = @JobName)
BEGIN
    EXEC msdb.dbo.sp_add_job 
        @job_name = @JobName,
        @enabled = 1,
        @category_name = N'Database Maintenance',
        @description = N'Automated daily full backup of all databases';
    
    EXEC msdb.dbo.sp_add_jobstep 
        @job_name = @JobName,
        @step_name = N'Execute Full Backup',
        @step_id = 1,
        @subsystem = N'TSQL',
        @command = N'EXEC sp_AutomatedBackup @BackupType = ''FULL'', @RetentionDays = 30, @Compress = 1, @Verify = 1',
        @on_success_action = 1,
        @on_fail_action = 2;
    
    EXEC msdb.dbo.sp_add_schedule 
        @schedule_name = @ScheduleName,
        @enabled = 1,
        @freq_type = 4, -- Daily
        @freq_interval = 1, -- Every day
        @active_start_date = 20240101,
        @active_start_time = 20000; -- 2:00 AM
    
    EXEC msdb.dbo.sp_attach_schedule 
        @job_name = @JobName,
        @schedule_name = @ScheduleName;
    
    EXEC msdb.dbo.sp_add_jobserver 
        @job_name = @JobName,
        @server_name = N'(local)';
END
GO
```

This advanced backup automation script provides:
- Dynamic backup creation for all databases or specific databases
- Support for full, differential, and log backups
- Backup verification
- Compression for space efficiency
- Error handling and logging
- Retention management

### Backup Monitoring and Alerting

Effective backup automation requires robust monitoring and alerting systems:

```sql
-- Backup monitoring procedure
CREATE PROCEDURE sp_BackupHealthCheck
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check for recent successful backups
    SELECT 
        d.name AS DatabaseName,
        bs.type_desc AS BackupType,
        bs.backup_start_date AS LastBackupStart,
        DATEDIFF(hour, bs.backup_start_date, GETDATE()) AS HoursSinceBackup,
        bs.backup_size AS BackupSizeMB,
        bs.compressed_backup_size AS CompressedSizeMB,
        bs.software_name AS BackupTool,
        CASE 
            WHEN bs.type_desc = 'FULL' AND DATEDIFF(hour, bs.backup_start_date, GETDATE()) > 48 THEN 'CRITICAL'
            WHEN bs.type_desc = 'DIFFERENTIAL' AND DATEDIFF(hour, bs.backup_start_date, GETDATE()) > 24 THEN 'WARNING'
            WHEN bs.type_desc = 'LOG' AND DATEDIFF(hour, bs.backup_start_date, GETDATE()) > 4 THEN 'WARNING'
            ELSE 'OK'
        END AS BackupHealthStatus
    FROM sys.databases d
    LEFT JOIN msdb.dbo.backupset bs ON d.name = bs.database_name
    WHERE d.database_id > 4 
    AND d.state = 0 -- ONLINE
    ORDER BY d.name, bs.backup_start_date DESC;
    
    -- Identify databases without recent backups
    SELECT 
        d.name AS DatabaseName,
        'NO BACKUP' AS Issue,
        'CRITICAL' AS Severity
    FROM sys.databases d
    WHERE d.database_id > 4 
    AND d.state = 0 -- ONLINE
    AND NOT EXISTS (
        SELECT 1 FROM msdb.dbo.backupset bs 
        WHERE d.name = bs.database_name 
        AND bs.backup_start_date > DATEADD(hour, -48, GETDATE())
    );
END
GO
```

## Index Maintenance Tasks {#index-maintenance-tasks}

### Index Health Assessment

Effective index maintenance begins with comprehensive health assessment. This involves analyzing fragmentation levels, index usage statistics, and maintenance opportunities. SQL Server provides several built-in functions and DMVs to gather this information.

**Key Index Health Metrics:**

1. **Fragmentation Levels**: Excessive logical and physical fragmentation impacts query performance
2. **Index Usage**: Unused indexes waste space and degrade write performance
3. **Missing Indexes**: Query optimizer recommendations for new indexes
4. **Duplicate Indexes**: Redundant indexes that should be consolidated

### Comprehensive Index Maintenance Script

Here's a sophisticated index maintenance procedure that includes health assessment, optimization, and monitoring:

```sql
USE master;
GO

-- Create index maintenance procedure
IF NOT EXISTS (SELECT 1 FROM sys.procedures WHERE name = 'sp_ComprehensiveIndexMaintenance')
BEGIN
    EXEC('
    CREATE PROCEDURE sp_ComprehensiveIndexMaintenance
    (
        @DatabaseName sysname = NULL,
        @FragmentationThresholdLow decimal(5,2) = 10.00,   -- Reorganize below this %
        @FragmentationThresholdHigh decimal(5,2) = 30.00,  -- Rebuild above this %
        @PageCountThreshold int = 1000,                     -- Only maintain indexes with > X pages
        @OnlineRebuild bit = 1,                            -- Use ONLINE = ON for rebuilds
        @UpdateStatistics bit = 1,                         -- Update statistics after maintenance
        @CreateMissingIndexes bit = 1,                     -- Create missing index recommendations
        @RemoveUnusedIndexes bit = 1,                      -- Remove unused indexes
        @ScheduleWindowMinutes int = 480,                  -- Maintenance window duration
        @LogToTable bit = 1                                -- Log operations to table
    )
    AS
    BEGIN
        SET NOCOUNT ON;
        
        DECLARE @StartTime datetime = GETDATE();
        DECLARE @ErrorCount int = 0;
        DECLARE @MaintenanceLog TABLE (
            LogID int IDENTITY(1,1),
            DatabaseName sysname,
            TableName sysname,
            IndexName sysname,
            Operation nvarchar(50),
            StartTime datetime,
            EndTime datetime,
            Status nvarchar(20),
            ErrorMessage nvarchar(4000),
            RowsAffected bigint,
            DurationSeconds int
        );
        
        -- Create log table if it doesn''t exist
        IF @LogToTable = 1 AND NOT EXISTS (SELECT 1 FROM tempdb.dbo.sysobjects WHERE name = ''IndexMaintenanceLog'')
        BEGIN
            CREATE TABLE tempdb.dbo.IndexMaintenanceLog (
                LogID int IDENTITY(1,1) PRIMARY KEY,
                DatabaseName sysname,
                TableName sysname,
                IndexName sysname,
                Operation nvarchar(50),
                StartTime datetime,
                EndTime datetime,
                Status nvarchar(20),
                ErrorMessage nvarchar(4000),
                RowsAffected bigint,
                DurationSeconds int
            );
        END
        
        -- Determine target databases
        DECLARE @TargetDBs TABLE (DatabaseName sysname);
        
        IF @DatabaseName IS NOT NULL AND EXISTS (SELECT 1 FROM sys.databases WHERE name = @DatabaseName AND database_id > 4)
        BEGIN
            INSERT INTO @TargetDBs (DatabaseName) VALUES (@DatabaseName);
        END
        ELSE
        BEGIN
            -- Use all user databases except system databases
            INSERT INTO @TargetDBs (DatabaseName)
            SELECT name 
            FROM sys.databases 
            WHERE database_id > 4 
            AND state = 0 -- ONLINE
            AND name NOT IN (''ReportServer'', ''ReportServerTempDB'')
            ORDER BY name;
        END
        
        -- Main processing loop
        DECLARE @CurrentDB sysname;
        DECLARE @SQL nvarchar(max);
        DECLARE @IndexStats TABLE (TableName sysname, IndexName sysname, FragmentationPercent decimal(5,2), PageCount int, IndexType nvarchar(50));
        
        DECLARE db_cursor CURSOR FOR SELECT DatabaseName FROM @TargetDBs;
        OPEN db_cursor;
        FETCH NEXT FROM db_cursor INTO @CurrentDB;
        
        WHILE @@FETCH_STATUS = 0
        BEGIN
            PRINT ''Processing database: '' + @CurrentDB;
            
            -- Collect index fragmentation data
            SET @SQL = N''
            SET NOCOUNT ON;
            
            -- Get index fragmentation and statistics
            SELECT 
                i.name AS IndexName,
                t.name AS TableName,
                ips.avg_fragmentation_in_percent AS FragmentationPercent,
                ips.page_count AS PageCount,
                i.type_desc AS IndexType,
                i.fill_factor AS FillFactor,
                ps.row_count AS TableRowCount,
                ps.reserved_page_count * 8 AS ReservedSpaceKB
            FROM '' + QUOTENAME(@CurrentDB) + ''.sys.indexes i
            INNER JOIN '' + QUOTENAME(@CurrentDB) + ''.sys.tables t ON i.object_id = t.object_id
            INNER JOIN '' + QUOTENAME(@CurrentDB) + ''.sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, ''DETAILED'') ips ON i.object_id = ips.object_id AND i.index_id = ips.index_id
            LEFT JOIN '' + QUOTENAME(@CurrentDB) + ''.sys.dm_db_partition_stats ps ON i.object_id = ps.object_id AND i.index_id = ps.index_id
            WHERE i.type > 0  -- Exclude heaps
            AND ips.index_level = 0  -- Only check leaf level
            AND ips.page_count > '' + CAST(@PageCountThreshold AS nvarchar(10)) + ''
            ORDER BY ips.avg_fragmentation_in_percent DESC;
            '';
            
            -- Note: In production, you would need to handle dynamic SQL execution properly
            -- This is a simplified version for demonstration
            
            -- Process index maintenance for collected statistics
            -- (Implementation would continue here based on actual fragmentation data)
            
            FETCH NEXT FROM db_cursor INTO @CurrentDB;
        END
        
        CLOSE db_cursor;
        DEALLOCATE db_cursor;
        
        -- Summary logging
        PRINT ''Index maintenance completed in '' + CAST(DATEDIFF(second, @StartTime, GETDATE()) AS nvarchar(10)) + '' seconds.'';
        
        IF @LogToTable = 1 AND EXISTS (SELECT 1 FROM tempdb.dbo.sysobjects WHERE name = ''IndexMaintenanceLog'')
        BEGIN
            -- Log summary to permanent table
            INSERT INTO dbo.IndexMaintenanceHistory
            SELECT DatabaseName, COUNT(*) AS TotalOperations, SUM(CASE WHEN Status = ''SUCCESS'' THEN 1 ELSE 0 END) AS SuccessfulOperations
            FROM tempdb.dbo.IndexMaintenanceLog
            GROUP BY DatabaseName;
        END
        
        RETURN 0;
    END
    ');
END
GO

-- Create missing index recommendations
IF NOT EXISTS (SELECT 1 FROM sys.procedures WHERE name = 'sp_AnalyzeMissingIndexes')
BEGIN
    EXEC('
    CREATE PROCEDURE sp_AnalyzeMissingIndexes
    (
        @DatabaseName sysname = NULL,
        @ExecutionThreshold int = 100,        -- Only create indexes for queries executed > X times
        @ScoreThreshold decimal(10,2) = 50.0,  -- Only create indexes with score > X
        @CreateIndexes bit = 0,               -- Actually create the indexes (0 = analysis only)
        @OnlineCreation bit = 1               -- Use ONLINE = ON for index creation
    )
    AS
    BEGIN
        SET NOCOUNT ON;
        
        -- Analyze missing index recommendations
        SELECT 
            d.name AS DatabaseName,
            mid.statement AS TableName,
            mid.equality_columns AS EqualityColumns,
            mid.inequality_columns AS InequalityColumns,
            mid.included_columns AS IncludedColumns,
            mid.user_seeks AS UserSeeks,
            mid.user_scans AS UserScans,
            mid.user_lookups AS UserLookups,
            mid.user_updates AS UserUpdates,
            mid.last_user_seek AS LastUserSeek,
            mid.last_user_scan AS LastUserScan,
            mid.last_user_lookup AS LastUserLookup,
            mid.last_user_update AS LastUserUpdate,
            -- Calculate impact score (simplified)
            (mid.user_seeks + mid.user_scans) * mid.avg_total_user_cost AS ImpactScore,
            CASE 
                WHEN (mid.user_seeks + mid.user_scans) >= ' + CAST(@ExecutionThreshold AS nvarchar(10)) + ' 
                     AND (mid.user_seeks + mid.user_scans) * mid.avg_total_user_cost >= ' + CAST(@ScoreThreshold AS nvarchar(10)) + '
                THEN ''HIGH''
                WHEN (mid.user_seeks + mid.user_scans) >= ' + CAST(@ExecutionThreshold/2 AS nvarchar(10)) + '
                THEN ''MEDIUM''
                ELSE ''LOW''
            END AS Priority
        FROM sys.databases d
        CROSS APPLY sys.dm_db_missing_index_details mid
        WHERE (@DatabaseName IS NULL OR d.name = @DatabaseName)
        AND mid.database_id = d.database_id
        AND (mid.user_seeks + mid.user_scans) >= ' + CAST(@ExecutionThreshold AS nvarchar(10)) + '
        ORDER BY ImpactScore DESC;
        
        -- Generate CREATE INDEX statements
        IF @CreateIndexes = 1
        BEGIN
            DECLARE @CreateIndexSQL nvarchar(max);
            DECLARE @IndexName nvarchar(128);
            DECLARE @TableName nvarchar(128);
            DECLARE @EqualityColumns nvarchar(500);
            DECLARE @InequalityColumns nvarchar(500);
            DECLARE @IncludedColumns nvarchar(500);
            
            -- Cursor to create recommended indexes
            DECLARE index_cursor CURSOR FOR
            SELECT DISTINCT
                mid.statement AS TableName,
                ''IX_'' + REPLACE(mid.statement, ''.'', ''_'') + ''_'' + CAST(ROW_NUMBER() OVER (ORDER BY (mid.user_seeks + mid.user_scans) DESC) AS nvarchar(10)) AS IndexName,
                mid.equality_columns AS EqualityColumns,
                mid.inequality_columns AS InequalityColumns,
                mid.included_columns AS IncludedColumns
            FROM sys.databases d
            CROSS APPLY sys.dm_db_missing_index_details mid
            WHERE (@DatabaseName IS NULL OR d.name = @DatabaseName)
            AND mid.database_id = d.database_id
            AND (mid.user_seeks + mid.user_scans) >= ' + CAST(@ExecutionThreshold AS nvarchar(10)) + '
            AND (mid.user_seeks + mid.user_scans) * mid.avg_total_user_cost >= ' + CAST(@ScoreThreshold AS nvarchar(10)) + ';
            
            OPEN index_cursor;
            FETCH NEXT FROM index_cursor INTO @TableName, @IndexName, @EqualityColumns, @InequalityColumns, @IncludedColumns;
            
            WHILE @@FETCH_STATUS = 0
            BEGIN
                BEGIN TRY
                    SET @CreateIndexSQL = N''CREATE NONCLUSTERED INDEX '' + QUOTENAME(@IndexName) + 
                                        N'' ON '' + @TableName + 
                                        N'' ('' + ISNULL(@EqualityColumns, '''') + 
                                        CASE WHEN @InequalityColumns IS NOT NULL THEN '', '' + @InequalityColumns ELSE '''' END + '')'' +
                                        CASE WHEN @IncludedColumns IS NOT NULL THEN N'' INCLUDE ('' + @IncludedColumns + '')'' ELSE '''' END +
                                        CASE WHEN @OnlineCreation = 1 THEN N'' WITH (ONLINE = ON)'' ELSE '''' END + N'';'';
                    
                    PRINT ''Creating index: '' + @IndexName;
                    EXEC sp_executesql @CreateIndexSQL;
                    
                    PRINT ''Successfully created index: '' + @IndexName;
                    
                END TRY
                BEGIN CATCH
                    PRINT ''Error creating index '' + @IndexName + '': '' + ERROR_MESSAGE();
                END CATCH
                
                FETCH NEXT FROM index_cursor INTO @TableName, @IndexName, @EqualityColumns, @InequalityColumns, @IncludedColumns;
            END
            
            CLOSE index_cursor;
            DEALLOCATE index_cursor;
        END
    END
    ');
END
GO
```

### Index Usage Analysis

Understanding index usage patterns is crucial for effective maintenance. Unused indexes consume resources without providing benefits, while frequently used indexes require careful maintenance.

```sql
-- Index usage analysis procedure
CREATE PROCEDURE sp_IndexUsageAnalysis
(
    @DatabaseName sysname = NULL,
    @UnusedIndexAgeDays int = 30,
    @SampleSizePercent decimal(5,2) = 100.00
)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Overall index usage statistics
    SELECT 
        d.name AS DatabaseName,
        SCHEMA_NAME(t.schema_id) AS SchemaName,
        t.name AS TableName,
        i.name AS IndexName,
        i.type_desc AS IndexType,
        s.user_seeks AS UserSeeks,
        s.user_scans AS UserScans,
        s.user_lookups AS UserLookups,
        s.user_updates AS UserUpdates,
        s.user_seeks + s.user_scans + s.user_lookups AS TotalReads,
        (s.user_seeks + s.user_scans + s.user_lookups) * 1.0 / NULLIF(s.user_updates, 0) AS ReadToWriteRatio,
        s.last_user_seek AS LastSeek,
        s.last_user_scan AS LastScan,
        s.last_user_lookup AS LastLookup,
        s.last_user_update AS LastUpdate,
        CASE 
            WHEN (s.user_seeks + s.user_scans + s.user_lookups) = 0 THEN 'UNUSED'
            WHEN (s.user_seeks + s.user_scans + s.user_lookups) * 1.0 / NULLIF(s.user_updates, 0) < 0.1 THEN 'WRITE_HEAVY'
            WHEN DATEDIFF(day, ISNULL(s.last_user_seek, s.last_user_scan, s.last_user_lookup), GETDATE()) > @UnusedIndexAgeDays THEN 'STALE'
            ELSE 'ACTIVE'
        END AS IndexStatus
    FROM sys.databases d
    INNER JOIN sys.tables t ON d.database_id = DB_ID(d.name)
    INNER JOIN sys.indexes i ON t.object_id = i.object_id
    LEFT JOIN sys.dm_db_index_usage_stats s ON d.database_id = s.database_id AND t.object_id = s.object_id AND i.index_id = s.index_id
    WHERE (@DatabaseName IS NULL OR d.name = @DatabaseName)
    AND d.database_id > 4
    AND i.type > 0 -- Exclude heaps
    ORDER BY TotalReads DESC;
    
    -- Identify unused indexes for potential removal
    SELECT 
        d.name AS DatabaseName,
        SCHEMA_NAME(t.schema_id) AS SchemaName,
        t.name AS TableName,
        i.name AS IndexName,
        i.type_desc AS IndexType,
        ps.used_page_count * 8 AS IndexSizeKB,
        ps.row_count AS TableRowCount,
        DATEDIFF(day, GETDATE(), s.last_user_update) AS DaysSinceLastUpdate,
        CASE WHEN i.is_primary_key = 1 THEN 'PRIMARY KEY'
             WHEN i.is_unique_constraint = 1 THEN 'UNIQUE CONSTRAINT'
             WHEN i.is_unique = 1 THEN 'UNIQUE'
             ELSE 'NON_UNIQUE'
        END AS IndexTypeCategory,
        'DROP INDEX ' + QUOTENAME(i.name) + ' ON ' + QUOTENAME(SCHEMA_NAME(t.schema_id)) + '.' + QUOTENAME(t.name) + ';' AS DropSQL
    FROM sys.databases d
    INNER JOIN sys.tables t ON d.database_id = DB_ID(d.name)
    INNER JOIN sys.indexes i ON t.object_id = i.object_id
    INNER JOIN sys.dm_db_partition_stats ps ON i.object_id = ps.object_id AND i.index_id = ps.index_id
    LEFT JOIN sys.dm_db_index_usage_stats s ON d.database_id = s.database_id AND t.object_id = s.object_id AND i.index_id = s.index_id
    WHERE (@DatabaseName IS NULL OR d.name = @DatabaseName)
    AND d.database_id > 4
    AND i.type > 0 -- Exclude heaps
    AND i.is_primary_key = 0 -- Exclude primary keys
    AND i.is_unique_constraint = 0 -- Exclude unique constraints
    AND (s.user_seeks IS NULL OR s.user_seeks + s.user_scans + s.user_lookups = 0
         OR DATEDIFF(day, GETDATE(), s.last_user_update) > @UnusedIndexAgeDays)
    ORDER BY ps.used_page_count DESC;
END
GO
```

## Database Integrity Checks {#database-integrity-checks}

### Comprehensive Integrity Assessment

Database integrity checks are essential for detecting corruption, structural issues, and data inconsistencies. SQL Server provides several integrity checking mechanisms that should be part of every maintenance plan.

**Types of Integrity Checks:**

1. **Physical Integrity Checks**: Detecting physical corruption on disk
2. **Logical Integrity Checks**: Verifying data relationships and constraints
3. **Allocation Integrity**: Ensuring proper space allocation
4. **Text/Image Data Integrity**: Checking large object data structures

### Advanced Integrity Check Implementation

```sql
-- Comprehensive integrity check procedure
CREATE PROCEDURE sp_ComprehensiveIntegrityCheck
(
    @DatabaseName sysname = NULL,
    @CheckType nvarchar(50) = 'ALL', -- ALL, PHYSICAL_ONLY, EXTENDED_LOGICAL_CHECKS
    @RepairLevel nvarchar(20) = 'NONE', -- NONE, REPAIR_ALLOW_DATA_LOSS, REPAIR_REBUILD
    @ExecuteRepair bit = 0,            -- Actually execute repair operations
    @MaxErrors int = 0,               -- Maximum errors before aborting (0 = unlimited)
    @NoInformMsg bit = 0,             -- Suppress informational messages
    @LogToTable bit = 1               -- Log results to table
)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @StartTime datetime = GETDATE();
    DECLARE @DatabaseList TABLE (DatabaseName sysname, Processed bit DEFAULT 0);
    DECLARE @IntegrityResults TABLE (
        DatabaseName sysname,
        CheckType nvarchar(50),
        StartTime datetime,
        EndTime datetime,
        Status nvarchar(20),
        ErrorCount int,
        WarningCount int,
        Details nvarchar(max)
    );
    
    -- Populate database list
    IF @DatabaseName IS NOT NULL
        INSERT INTO @DatabaseList (DatabaseName) VALUES (@DatabaseName);
    ELSE
        INSERT INTO @DatabaseList (DatabaseName)
        SELECT name 
        FROM sys.databases 
        WHERE database_id > 4 
        AND state = 0 -- ONLINE
        AND name NOT IN ('ReportServer', 'ReportServerTempDB', 'tempdb');
    
    -- Process each database
    DECLARE @CurrentDB sysname;
    DECLARE @SQL nvarchar(max);
    DECLARE @CheckCommand nvarchar(max);
    DECLARE @ResultStatus nvarchar(20) = 'UNKNOWN';
    DECLARE @ErrorCount int = 0;
    DECLARE @WarningCount int = 0;
    DECLARE @ResultDetails nvarchar(max) = '';
    
    DECLARE db_cursor CURSOR FOR SELECT DatabaseName FROM @DatabaseList WHERE Processed = 0;
    OPEN db_cursor;
    
    FETCH NEXT FROM db_cursor INTO @CurrentDB;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        PRINT 'Starting integrity check for database: ' + @CurrentDB;
        
        DECLARE @DBStartTime datetime = GETDATE();
        SET @ErrorCount = 0;
        SET @WarningCount = 0;
        SET @ResultDetails = '';
        
        BEGIN TRY
            -- Execute integrity checks based on check type
            IF @CheckType = 'ALL' OR @CheckType = 'COMPREHENSIVE'
            BEGIN
                -- Full integrity check
                SET @CheckCommand = 'DBCC CHECKDB(' + QUOTENAME(@CurrentDB) + ') WITH ' +
                    CASE WHEN @CheckType = 'PHYSICAL_ONLY' THEN 'PHYSICAL_ONLY'
                         WHEN @CheckType = 'EXTENDED_LOGICAL_CHECKS' THEN 'EXTENDED_LOGICAL_CHECKS'
                         ELSE 'NO_INFOMSGS, ALL_ERRORMSGS'
                    END +
                    CASE WHEN @NoInformMsg = 1 THEN ', NO_INFOMSGS' ELSE '' END +
                    CASE WHEN @ExecuteRepair = 1 AND @RepairLevel != 'NONE' THEN ', ' + @RepairLevel ELSE '' END +
                    ';';
                
                PRINT 'Executing: ' + @CheckCommand;
                EXEC(@CheckCommand);
            END
            
            IF @CheckType = 'ALL' OR @CheckType = 'PHYSICAL'
            BEGIN
                -- Physical integrity only
                SET @CheckCommand = 'DBCC CHECKDB(' + QUOTENAME(@CurrentDB) + ') WITH PHYSICAL_ONLY, NO_INFOMSGS;';
                EXEC(@CheckCommand);
            END
            
            IF @CheckType = 'ALL' OR @CheckType = 'CATALOG'
            BEGIN
                -- Catalog integrity check
                SET @CheckCommand = 'DBCC CHECKCATALOG(' + QUOTENAME(@CurrentDB) + ') WITH NO_INFOMSGS;';
                EXEC(@CheckCommand);
            END
            
            IF @CheckType = 'ALL' OR @CheckType = 'CONSTRAINTS'
            BEGIN
                -- Constraint checking
                SET @CheckCommand = 'DBCC CHECKCONSTRAINTS WITH ALL_CONSTRAINTS, NO_INFOMSGS;';
                
                -- Switch to target database context
                SET @SQL = 'USE ' + QUOTENAME(@CurrentDB) + '; ' + @CheckCommand;
                EXEC(@SQL);
            END
            
            -- Collect additional integrity information
            IF @CheckType = 'ALL' OR @CheckType = 'ALLOCATION'
            BEGIN
                -- Allocation integrity
                SET @CheckCommand = 'DBCC CHECKALLOC(' + QUOTENAME(@CurrentDB) + ') WITH NO_INFOMSGS, ESTIMATEONLY;';
                
                -- Switch to target database context
                SET @SQL = 'USE ' + QUOTENAME(@CurrentDB) + '; ' + @CheckCommand;
                EXEC(@SQL);
            END
            
            -- Collect table and index information
            SET @SQL = N'
            USE ' + QUOTENAME(@CurrentDB) + N';
            SELECT 
                t.name AS TableName,
                i.name AS IndexName,
                i.type_desc AS IndexType,
                ips.avg_fragmentation_in_percent AS FragmentationPercent,
                ips.page_count AS PageCount,
                ips.record_count AS RecordCount
            FROM sys.indexes i
            INNER JOIN sys.tables t ON i.object_id = t.object_id
            CROSS APPLY sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, ''DETAILED'') ips
            WHERE i.type > 0
            AND ips.index_level = 0
            ORDER BY ips.avg_fragmentation_in_percent DESC;';
            
            -- This would be executed to collect index health data
            
            SET @ResultStatus = 'SUCCESS';
            SET @ResultDetails = 'Integrity checks completed successfully for ' + @CurrentDB;
            
        END TRY
        BEGIN CATCH
            SET @ResultStatus = 'ERROR';
            SET @ErrorCount = ERROR_NUMBER();
            SET @ResultDetails = ERROR_MESSAGE();
            
            PRINT 'Integrity check failed for ' + @CurrentDB + ': ' + ERROR_MESSAGE();
        END CATCH
        
        -- Log results
        INSERT INTO @IntegrityResults (
            DatabaseName,
            CheckType,
            StartTime,
            EndTime,
            Status,
            ErrorCount,
            WarningCount,
            Details
        )
        VALUES (
            @CurrentDB,
            @CheckType,
            @DBStartTime,
            GETDATE(),
            @ResultStatus,
            @ErrorCount,
            @WarningCount,
            @ResultDetails
        );
        
        -- Log to permanent table if requested
        IF @LogToTable = 1
        BEGIN
            IF NOT EXISTS (SELECT 1 FROM sys.tables WHERE name = 'IntegrityCheckHistory' AND schema_id = SCHEMA_ID('dbo'))
            BEGIN
                CREATE TABLE dbo.IntegrityCheckHistory (
                    CheckID int IDENTITY(1,1) PRIMARY KEY,
                    DatabaseName sysname,
                    CheckType nvarchar(50),
                    CheckDate datetime DEFAULT GETDATE(),
                    Status nvarchar(20),
                    ErrorCount int,
                    WarningCount int,
                    DurationSeconds int,
                    Details nvarchar(max)
                );
            END
            
            INSERT INTO dbo.IntegrityCheckHistory (
                DatabaseName, CheckType, Status, ErrorCount, WarningCount, 
                DurationSeconds, Details
            )
            SELECT 
                @CurrentDB, @CheckType, @ResultStatus, @ErrorCount, @WarningCount,
                DATEDIFF(second, @DBStartTime, GETDATE()),
                @ResultDetails;
        END
        
        -- Mark database as processed
        UPDATE @DatabaseList SET Processed = 1 WHERE DatabaseName = @CurrentDB;
        
        FETCH NEXT FROM db_cursor INTO @CurrentDB;
    END
    
    CLOSE db_cursor;
    DEALLOCATE db_cursor;
    
    -- Return summary results
    SELECT 
        DatabaseName,
        CheckType,
        StartTime,
        EndTime,
        Status,
        ErrorCount,
        WarningCount,
        DATEDIFF(second, StartTime, EndTime) AS DurationSeconds,
        Details
    FROM @IntegrityResults
    ORDER BY DatabaseName;
    
    -- Return overall summary
    SELECT 
        COUNT(*) AS TotalDatabasesChecked,
        SUM(CASE WHEN Status = 'SUCCESS' THEN 1 ELSE 0 END) AS SuccessfulChecks,
        SUM(CASE WHEN Status = 'ERROR' THEN 1 ELSE 0 END) AS FailedChecks,
        SUM(ErrorCount) AS TotalErrors,
        SUM(WarningCount) AS TotalWarnings,
        DATEDIFF(second, MIN(StartTime), MAX(EndTime)) AS TotalDurationSeconds
    FROM @IntegrityResults;
    
    RETURN 0;
END
GO
```

### Data Purity Checks

Data purity checks verify that columns contain valid data types and values:

```sql
-- Data purity check procedure
CREATE PROCEDURE sp_DataPurityChecks
(
    @DatabaseName sysname = NULL,
    @LogToTable bit = 1
)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @PurityResults TABLE (
        DatabaseName sysname,
        TableName sysname,
        ColumnName sysname,
        DataType nvarchar(128),
        InvalidValues int,
        TotalRows int,
        PurityPercent decimal(5,2)
    );
    
    -- Process each database
    DECLARE @CurrentDB sysname;
    DECLARE @SQL nvarchar(max);
    
    DECLARE db_cursor CURSOR FOR
    SELECT name 
    FROM sys.databases 
    WHERE (@DatabaseName IS NULL OR name = @DatabaseName)
    AND database_id > 4 
    AND state = 0 -- ONLINE;
    
    OPEN db_cursor;
    FETCH NEXT FROM db_cursor INTO @CurrentDB;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        PRINT 'Checking data purity for database: ' + @CurrentDB;
        
        -- Check for common data type issues
        SET @SQL = N'
        USE ' + QUOTENAME(@CurrentDB) + N';
        
        -- Check for invalid dates
        SELECT 
            ''' + @CurrentDB + ''' AS DatabaseName,
            OBJECT_NAME(o.object_id) AS TableName,
            c.name AS ColumnName,
            t.name AS DataType,
            COUNT(*) AS InvalidCount,
            (SELECT COUNT(*) FROM ' + QUOTENAME(@CurrentDB) + '.dbo.' + QUOTENAME(OBJECT_NAME(o.object_id)) + ') AS TotalRows,
            CASE WHEN (SELECT COUNT(*) FROM ' + QUOTENAME(@CurrentDB) + '.dbo.' + QUOTENAME(OBJECT_NAME(o.object_id)) + ') = 0 
                 THEN 100.00 
                 ELSE CAST((COUNT(*) * 100.0) / (SELECT COUNT(*) FROM ' + QUOTENAME(@CurrentDB) + '.dbo.' + QUOTENAME(OBJECT_NAME(o.object_id)) + ') AS decimal(5,2))
            END AS PurityPercent
        FROM sys.columns c
        INNER JOIN sys.types t ON c.user_type_id = t.user_type_id
        INNER JOIN sys.objects o ON c.object_id = o.object_id
        WHERE t.name IN (''date'', ''datetime'', ''datetime2'', ''smalldatetime'')
        AND o.type = ''U'' -- User tables
        GROUP BY o.object_id, c.name, t.name
        HAVING COUNT(*) > 0;';
        
        -- Note: This is a simplified version. In production, you would need more sophisticated data validation.
        
        FETCH NEXT FROM db_cursor INTO @CurrentDB;
    END
    
    CLOSE db_cursor;
    DEALLOCATE db_cursor;
    
    -- Return results
    SELECT * FROM @PurityResults ORDER BY DatabaseName, PurityPercent ASC;
END
GO
```

## Statistics Updates {#statistics-updates}

### Automatic vs Manual Statistics Management

SQL Server automatically maintains statistics for indexes and columns, but in high-change environments, manual statistics updates may be necessary to ensure optimal query performance. Understanding when and how to update statistics is crucial for database maintenance.

**Statistics Update Strategies:**

1. **Automatic Statistics Updates**: Rely on SQL Server's built-in statistics maintenance
2. **Scheduled Statistics Updates**: Manual updates during maintenance windows
3. **Real-time Statistics Updates**: Immediate updates after significant data changes
4. **Filtered Statistics Updates**: Update only statistics that have changed significantly

### Advanced Statistics Management

```sql
-- Comprehensive statistics maintenance procedure
CREATE PROCEDURE sp_StatisticsMaintenance
(
    @DatabaseName sysname = NULL,
    @UpdateMethod nvarchar(20) = 'INCREMENTAL', -- FULL, INCREMENTAL, SAMPLE
    @SamplePercent int = 100,                   -- Sample size percentage
    @StatisticsAgeThreshold int = 7,            -- Days since last update
    @RowChangeThreshold decimal(5,2) = 20.00,   -- % change threshold for auto-update
    @UpdateAllStatistics bit = 1,               -- Update all statistics
    @UpdateColumnStatistics bit = 1,            -- Include column statistics
    @UseFullscan bit = 0,                       -- Use FULLSCAN for statistics
    @ScheduleWindowMinutes int = 240,           -- Available maintenance time
    @LogToTable bit = 1                         -- Log operations to table
)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @StartTime datetime = GETDATE();
    DECLARE @StatisticsResults TABLE (
        DatabaseName sysname,
        SchemaName sysname,
        TableName sysname,
        StatisticsName sysname,
        StatisticsType nvarchar(50),
        LastUpdated datetime,
        RowsChanged bigint,
        PercentChanged decimal(5,2),
        UpdateTime datetime,
        DurationSeconds int,
        Status nvarchar(20)
    );
    
    -- Create statistics log table if requested
    IF @LogToTable = 1 AND NOT EXISTS (SELECT 1 FROM sys.tables WHERE name = 'StatisticsUpdateHistory' AND schema_id = SCHEMA_ID('dbo'))
    BEGIN
        CREATE TABLE dbo.StatisticsUpdateHistory (
            UpdateID int IDENTITY(1,1) PRIMARY KEY,
            DatabaseName sysname,
            SchemaName sysname,
            TableName sysname,
            StatisticsName sysname,
            UpdateMethod nvarchar(20),
            SamplePercent int,
            RowsAffected bigint,
            UpdateTime datetime DEFAULT GETDATE(),
            DurationSeconds int,
            Status nvarchar(20)
        );
    END
    
    -- Process each database
    DECLARE @CurrentDB sysname;
    DECLARE @SQL nvarchar(max);
    DECLARE @StatisticsName nvarchar(128);
    DECLARE @TableName nvarchar(128);
    DECLARE @SchemaName nvarchar(128);
    DECLARE @LastUpdated datetime;
    DECLARE @Rows bigint;
    DECLARE @ModifiedRows bigint;
    DECLARE @PercentChanged decimal(5,2);
    
    DECLARE db_cursor CURSOR FOR
    SELECT name 
    FROM sys.databases 
    WHERE (@DatabaseName IS NULL OR name = @DatabaseName)
    AND database_id > 4 
    AND state = 0 -- ONLINE;
    
    OPEN db_cursor;
    FETCH NEXT FROM db_cursor INTO @CurrentDB;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        PRINT 'Updating statistics for database: ' + @CurrentDB;
        
        -- Switch to database context
        SET @SQL = 'USE ' + QUOTENAME(@CurrentDB) + ';';
        EXEC(@SQL);
        
        -- Get statistics that need updating
        DECLARE stats_cursor CURSOR FOR
        SELECT 
            s.name AS StatisticsName,
            t.name AS TableName,
            SCHEMA_NAME(t.schema_id) AS SchemaName,
            sp.last_updated AS LastUpdated,
            sp.rows AS TotalRows,
            sp.rows_sampled AS SampleRows,
            sp.modification_counter AS ModifiedRows,
            CASE 
                WHEN sp.rows = 0 THEN 0.00
                ELSE CAST((sp.modification_counter * 100.0) / sp.rows AS decimal(5,2))
            END AS PercentChanged
        FROM sys.stats s
        INNER JOIN sys.tables t ON s.object_id = t.object_id
        CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
        WHERE (@UpdateAllStatistics = 1 OR s.auto_created = 1)
        AND (
            DATEDIFF(day, sp.last_updated, GETDATE()) >= @StatisticsAgeThreshold
            OR (sp.rows > 0 AND sp.modification_counter > 0 AND 
                CAST((sp.modification_counter * 100.0) / sp.rows AS decimal(5,2)) >= @RowChangeThreshold)
        )
        AND t.type = 'U' -- User tables only
        ORDER BY sp.modification_counter DESC;
        
        OPEN stats_cursor;
        FETCH NEXT FROM stats_cursor INTO @StatisticsName, @TableName, @SchemaName, @LastUpdated, @Rows, @ModifiedRows, @PercentChanged;
        
        WHILE @@FETCH_STATUS = 0
        BEGIN
            DECLARE @StatStartTime datetime = GETDATE();
            
            BEGIN TRY
                -- Determine update method
                SET @SQL = 'UPDATE STATISTICS ' + QUOTENAME(@SchemaName) + '.' + QUOTENAME(@TableName);
                
                IF @StatisticsName != ''
                    SET @SQL = @SQL + ' ' + QUOTENAME(@StatisticsName);
                
                -- Add update options
                IF @UpdateMethod = 'FULLSCAN' OR @UseFullscan = 1
                    SET @SQL = @SQL + ' WITH FULLSCAN';
                ELSE IF @UpdateMethod = 'SAMPLE'
                    SET @SQL = @SQL + ' WITH SAMPLE ' + CAST(@SamplePercent AS nvarchar(10)) + ' PERCENT';
                ELSE -- INCREMENTAL
                    SET @SQL = @SQL + ' WITH INCREMENTAL = ON';
                
                IF @UpdateColumnStatistics = 1
                    SET @SQL = @SQL + ', COLUMNS';
                
                PRINT 'Updating statistics: ' + @StatisticsName + ' on ' + @SchemaName + '.' + @TableName;
                
                EXEC sp_executesql @SQL;
                
                -- Log successful update
                INSERT INTO @StatisticsResults (
                    DatabaseName, SchemaName, TableName, StatisticsName, StatisticsType,
                    LastUpdated, RowsChanged, PercentChanged, UpdateTime, DurationSeconds, Status
                )
                VALUES (
                    @CurrentDB, @SchemaName, @TableName, @StatisticsName, 
                    CASE WHEN @StatisticsName LIKE 'PK_%' THEN 'INDEX_STATISTICS'
                         WHEN @StatisticsName LIKE 'UQ_%' THEN 'CONSTRAINT_STATISTICS'
                         ELSE 'COLUMN_STATISTICS'
                    END,
                    @LastUpdated, @ModifiedRows, @PercentChanged, 
                    GETDATE(), DATEDIFF(second, @StatStartTime, GETDATE()), 'SUCCESS'
                );
                
                -- Log to history table
                IF @LogToTable = 1
                BEGIN
                    INSERT INTO dbo.StatisticsUpdateHistory (
                        DatabaseName, SchemaName, TableName, StatisticsName,
                        UpdateMethod, SamplePercent, RowsAffected, DurationSeconds, Status
                    )
                    VALUES (
                        @CurrentDB, @SchemaName, @TableName, @StatisticsName,
                        @UpdateMethod, @SamplePercent, @ModifiedRows,
                        DATEDIFF(second, @StatStartTime, GETDATE()), 'SUCCESS'
                    );
                END
                
                PRINT 'Successfully updated statistics: ' + @StatisticsName;
                
            END TRY
            BEGIN CATCH
                PRINT 'Error updating statistics ' + @StatisticsName + ': ' + ERROR_MESSAGE();
                
                INSERT INTO @StatisticsResults (
                    DatabaseName, SchemaName, TableName, StatisticsName, StatisticsType,
                    LastUpdated, RowsChanged, PercentChanged, UpdateTime, DurationSeconds, Status
                )
                VALUES (
                    @CurrentDB, @SchemaName, @TableName, @StatisticsName, 'UNKNOWN',
                    @LastUpdated, @ModifiedRows, @PercentChanged, 
                    GETDATE(), DATEDIFF(second, @StatStartTime, GETDATE()), 'ERROR: ' + ERROR_MESSAGE()
                );
            END CATCH
            
            FETCH NEXT FROM stats_cursor INTO @StatisticsName, @TableName, @SchemaName, @LastUpdated, @Rows, @ModifiedRows, @PercentChanged;
        END
        
        CLOSE stats_cursor;
        DEALLOCATE stats_cursor;
        
        FETCH NEXT FROM db_cursor INTO @CurrentDB;
    END
    
    CLOSE db_cursor;
    DEALLOCATE db_cursor;
    
    -- Return results
    SELECT * FROM @StatisticsResults ORDER BY DatabaseName, TableName, StatisticsName;
    
    -- Return summary
    SELECT 
        COUNT(*) AS TotalStatisticsUpdated,
        SUM(CASE WHEN Status = 'SUCCESS' THEN 1 ELSE 0 END) AS SuccessfulUpdates,
        SUM(CASE WHEN Status LIKE 'ERROR%' THEN 1 ELSE 0 END) AS FailedUpdates,
        SUM(DurationSeconds) AS TotalDurationSeconds,
        AVG(DurationSeconds) AS AverageDurationSeconds
    FROM @StatisticsResults;
    
    PRINT 'Statistics maintenance completed in ' + CAST(DATEDIFF(second, @StartTime, GETDATE()) AS nvarchar(10)) + ' seconds.';
    
    RETURN 0;
END
GO
```

## Cleanup Tasks {#cleanup-tasks}

### Maintenance Cleanup Operations

Cleanup tasks are essential for maintaining database health by removing old files, clearing temporary data, and managing disk space. This includes backup file cleanup, error log maintenance, and database maintenance plan history cleanup.

**Key Cleanup Operations:**

1. **Backup File Cleanup**: Remove old backup files based on retention policies
2. **Error Log Maintenance**: Archive and clean up SQL Server error logs
3. **Database Maintenance History**: Clear old maintenance plan execution history
4. **Temporary File Cleanup**: Remove temporary files and data
5. **Disk Space Management**: Monitor and manage disk space usage

### Comprehensive Cleanup Procedure

```sql
-- Comprehensive cleanup procedure
CREATE PROCEDURE sp_MaintenanceCleanup
(
    @CleanupType nvarchar(50) = 'ALL',          -- ALL, BACKUP_FILES, ERROR_LOGS, MAINTENANCE_HISTORY, TEMPORARY_FILES
    @RetentionDays int = 30,                     -- Retention period for cleanup items
    @BackupPath nvarchar(500) = 'C:\SQLBackups\', -- Backup file path for cleanup
    @FileExtension nvarchar(10) = '.bak',        -- File extension to clean up
    @CleanupSchedule nvarchar(50) = 'WEEKLY',    -- Schedule type for cleanup
    @LogToTable bit = 1,                         -- Log cleanup operations
    @DryRun bit = 0                              -- Preview mode without actual deletion
)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CleanupResults TABLE (
        Operation nvarchar(50),
        FilePath nvarchar(500),
        FileSizeMB decimal(10,2),
        DateCreated datetime,
        DateModified datetime,
        Deleted bit DEFAULT 0,
        ErrorMessage nvarchar(4000)
    );
    
    DECLARE @TotalSpaceFreedMB decimal(10,2) = 0;
    DECLARE @FilesDeleted int = 0;
    DECLARE @CleanupStartTime datetime = GETDATE();
    
    IF @CleanupType = 'ALL' OR @CleanupType = 'BACKUP_FILES'
    BEGIN
        PRINT 'Cleaning up old backup files from: ' + @BackupPath;
        
        -- Note: This requires xp_cmdshell to be enabled and appropriate permissions
        -- In production, consider using PowerShell or other file management methods
        
        -- Create dynamic cleanup SQL for backup files
        DECLARE @CleanupSQL nvarchar(max);
        DECLARE @FileName nvarchar(500);
        DECLARE @FilePath nvarchar(1000);
        DECLARE @FileExists int;
        DECLARE @FileSize bigint;
        DECLARE @FileDateCreated datetime;
        DECLARE @FileDateModified datetime;
        
        -- Sample cleanup (this would need to be implemented based on your specific file system access method)
        -- For demonstration, we'll simulate file cleanup operations
        
        -- In production, you would use:
        -- 1. xp_cmdshell with PowerShell to enumerate files
        -- 2. SQL Server File System Objects
        -- 3. External file management utilities
        -- 4. Built-in SQL Server maintenance plan cleanup tasks
        
        PRINT 'Backup file cleanup simulation completed. In production, implement file enumeration and deletion logic.';
    END
    
    IF @CleanupType = 'ALL' OR @CleanupType = 'ERROR_LOGS'
    BEGIN
        PRINT 'Cleaning up SQL Server error logs older than ' + CAST(@RetentionDays AS nvarchar(10)) + ' days.';
        
        -- Archive old error logs
        DECLARE @ErrorLogCount int;
        
        -- Get current error log count
        SELECT @ErrorLogCount = COUNT(*) 
        FROM sys.dm_os_performance_counters 
        WHERE counter_name = 'SQL Errors/sec';
        
        -- Archive logs if we have more than the desired retention
        IF @ErrorLogCount > (@RetentionDays / 7) -- Assume 1 log per week
        BEGIN
            PRINT 'Archiving old error logs...';
            -- Implementation would use sp_cycle_errorlog or manual archival
        END
    END
    
    IF @CleanupType = 'ALL' OR @CleanupType = 'MAINTENANCE_HISTORY'
    BEGIN
        PRINT 'Cleaning up maintenance plan execution history older than ' + CAST(@RetentionDays AS nvarchar(10)) + ' days.';
        
        -- Clean up msdb maintenance plan history
        DECLARE @CutoffDate datetime = DATEADD(day, -@RetentionDays, GETDATE());
        
        -- Delete old maintenance plan history
        DELETE FROM msdb.dbo.sysdbmaintplan_history 
        WHERE start_time < @CutoffDate;
        
        DECLARE @HistoryDeleted int = @@ROWCOUNT;
        PRINT 'Deleted ' + CAST(@HistoryDeleted AS nvarchar(10)) + ' maintenance plan history records older than ' + CAST(@CutoffDate AS nvarchar(50)) + '.';
        
        -- Clean up backup history
        DELETE FROM msdb.dbo.backupfilehistory 
        WHERE backup_start_date < @CutoffDate;
        
        DECLARE @BackupHistoryDeleted int = @@ROWCOUNT;
        PRINT 'Deleted ' + CAST(@BackupHistoryDeleted AS nvarchar(10)) + ' backup history records older than ' + CAST(@CutoffDate AS nvarchar(50)) + '.';
        
        -- Log cleanup results
        INSERT INTO @CleanupResults (Operation, FilePath, FileSizeMB, DateCreated, DateModified, Deleted, ErrorMessage)
        VALUES ('MAINTENANCE_HISTORY', 'msdb.dbo.sysdbmaintplan_history', NULL, @CutoffDate, GETDATE(), 1, NULL);
        
        INSERT INTO @CleanupResults (Operation, FilePath, FileSizeMB, DateCreated, DateModified, Deleted, ErrorMessage)
        VALUES ('BACKUP_HISTORY', 'msdb.dbo.backupfilehistory', NULL, @CutoffDate, GETDATE(), 1, NULL);
    END
    
    IF @CleanupType = 'ALL' OR @CleanupType = 'TEMPORARY_FILES'
    BEGIN
        PRINT 'Cleaning up temporary files and data.';
        
        -- Clean up tempdb if needed
        IF DB_ID('tempdb') IS NOT NULL
        BEGIN
            -- Shrink tempdb data files if they have grown significantly
            DECLARE @TempDBDataFile nvarchar(128);
            DECLARE @CurrentSizeMB bigint;
            DECLARE @TargetSizeMB bigint;
            
            -- This would be implemented with ALTER DATABASE tempdb MODIFY FILE operations
            
            PRINT 'TempDB cleanup operations completed.';
        END
        
        -- Clean up old temporary tables in current database
        DECLARE @TempTableCount int;
        
        SET @CleanupSQL = N'
        SELECT @TempCount = COUNT(*)
        FROM sys.tables
        WHERE name LIKE ''#%''
        AND create_date < DATEADD(hour, -' + CAST(@RetentionDays * 24 AS nvarchar(10)) + ', GETDATE())';
        
        EXEC sp_executesql @CleanupSQL, N'@TempCount int OUTPUT', @TempCount OUTPUT;
        
        PRINT 'Found ' + CAST(ISNULL(@TempTableCount, 0) AS nvarchar(10)) + ' old temporary tables to clean up.';
    END
    
    -- Log cleanup operations if requested
    IF @LogToTable = 1
    BEGIN
        IF NOT EXISTS (SELECT 1 FROM sys.tables WHERE name = 'MaintenanceCleanupHistory' AND schema_id = SCHEMA_ID('dbo'))
        BEGIN
            CREATE TABLE dbo.MaintenanceCleanupHistory (
                CleanupID int IDENTITY(1,1) PRIMARY KEY,
                CleanupType nvarchar(50),
                CleanupDate datetime DEFAULT GETDATE(),
                RetentionDays int,
                TotalFilesDeleted int,
                TotalSpaceFreedMB decimal(10,2),
                DurationSeconds int,
                Status nvarchar(20)
            );
        END
        
        INSERT INTO dbo.MaintenanceCleanupHistory (
            CleanupType, RetentionDays, TotalFilesDeleted, TotalSpaceFreedMB, 
            DurationSeconds, Status
        )
        VALUES (
            @CleanupType, @RetentionDays, @FilesDeleted, @TotalSpaceFreedMB,
            DATEDIFF(second, @CleanupStartTime, GETDATE()), 
            CASE WHEN @DryRun = 1 THEN 'DRY_RUN' ELSE 'COMPLETED' END
        );
    END
    
    -- Return summary
    SELECT 
        @CleanupType AS CleanupType,
        @FilesDeleted AS FilesDeleted,
        @TotalSpaceFreedMB AS SpaceFreedMB,
        DATEDIFF(second, @CleanupStartTime, GETDATE()) AS DurationSeconds,
        CASE WHEN @DryRun = 1 THEN 'DRY_RUN' ELSE 'COMPLETED' END AS Status;
    
    PRINT 'Cleanup operation completed. Files deleted: ' + CAST(@FilesDeleted AS nvarchar(10)) + 
          ', Space freed: ' + CAST(@TotalSpaceFreedMB AS nvarchar(10)) + ' MB';
    
    RETURN 0;
END
GO
```

## Health Monitoring {#health-monitoring}

### Database Health Assessment Framework

Comprehensive health monitoring involves collecting and analyzing various database metrics to identify potential issues before they become critical. This includes performance metrics, growth trends, and operational health indicators.

**Key Health Metrics:**

1. **Performance Metrics**: CPU usage, memory consumption, I/O patterns
2. **Growth Metrics**: Database file growth, data volume trends
3. **Availability Metrics**: Uptime, connection counts, blocking issues
4. **Capacity Metrics**: Disk space utilization, file size limits
5. **Error Metrics**: SQL errors, failed backups, integrity check failures

### Health Monitoring Implementation

```sql
-- Database health monitoring procedure
CREATE PROCEDURE sp_DatabaseHealthMonitor
(
    @DatabaseName sysname = NULL,
    @HealthCheckType nvarchar(50) = 'COMPREHENSIVE', -- COMPREHENSIVE, PERFORMANCE, CAPACITY, AVAILABILITY
    @GenerateReport bit = 1,                         -- Generate detailed health report
    @AlertThresholds bit = 1,                        -- Apply alerting thresholds
    @ExportToTable bit = 1                           -- Export results to permanent table
)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @HealthReport TABLE (
        DatabaseName sysname,
        MetricCategory nvarchar(50),
        MetricName nvarchar(100),
        MetricValue decimal(18,2),
        MetricUnit nvarchar(20),
        ThresholdWarning decimal(18,2),
        ThresholdCritical decimal(18,2),
        HealthStatus nvarchar(20),
        LastChecked datetime,
        TrendDirection nvarchar(20),
        Recommendation nvarchar(500)
    );
    
    DECLARE @StartTime datetime = GETDATE();
    DECLARE @ReportGenerated bit = 0;
    
    -- Performance Health Check
    IF @HealthCheckType = 'COMPREHENSIVE' OR @HealthCheckType = 'PERFORMANCE'
    BEGIN
        PRINT 'Collecting performance health metrics...';
        
        -- CPU and Memory utilization
        INSERT INTO @HealthReport (
            DatabaseName, MetricCategory, MetricName, MetricValue, MetricUnit,
            ThresholdWarning, ThresholdCritical, HealthStatus, LastChecked, Recommendation
        )
        SELECT 
            @DatabaseName AS DatabaseName,
            'PERFORMANCE' AS MetricCategory,
            'CPU_Usage_Percent' AS MetricName,
            CAST((SELECT counter_value FROM sys.dm_os_performance_counters WHERE counter_name = 'CPU usage %') AS decimal(18,2)) AS MetricValue,
            'PERCENT' AS MetricUnit,
            70.00 AS ThresholdWarning,
            90.00 AS ThresholdCritical,
            CASE 
                WHEN (SELECT counter_value FROM sys.dm_os_performance_counters WHERE counter_name = 'CPU usage %') >= 90 THEN 'CRITICAL'
                WHEN (SELECT counter_value FROM sys.dm_os_performance_counters WHERE counter_name = 'CPU usage %') >= 70 THEN 'WARNING'
                ELSE 'HEALTHY'
            END AS HealthStatus,
            GETDATE() AS LastChecked,
            CASE 
                WHEN (SELECT counter_value FROM sys.dm_os_performance_counters WHERE counter_name = 'CPU usage %') >= 90 THEN 'Investigate high CPU usage - check for runaway queries, index fragmentation, or insufficient hardware'
                WHEN (SELECT counter_value FROM sys.dm_os_performance_counters WHERE counter_name = 'CPU usage %') >= 70 THEN 'Monitor CPU usage closely - consider optimization or hardware upgrade'
                ELSE 'CPU usage is within acceptable limits'
            END AS Recommendation;
        
        -- Memory utilization
        INSERT INTO @HealthReport (
            DatabaseName, MetricCategory, MetricName, MetricValue, MetricUnit,
            ThresholdWarning, ThresholdCritical, HealthStatus, LastChecked, Recommendation
        )
        SELECT 
            @DatabaseName AS DatabaseName,
            'PERFORMANCE' AS MetricCategory,
            'Memory_Usage_Percent' AS MetricName,
            CAST((SELECT counter_value FROM sys.dm_os_performance_counters WHERE counter_name = 'Memory Grants Pending' AND instance_name = @DatabaseName) AS decimal(18,2)) AS MetricValue,
            'PERCENT' AS MetricUnit,
            10.00 AS ThresholdWarning,
            25.00 AS ThresholdCritical,
            CASE 
                WHEN (SELECT counter_value FROM sys.dm_os_performance_counters WHERE counter_name = 'Memory Grants Pending' AND instance_name = @DatabaseName) >= 25 THEN 'CRITICAL'
                WHEN (SELECT counter_value FROM sys.dm_os_performance_counters WHERE counter_name = 'Memory Grants Pending' AND instance_name = @DatabaseName) >= 10 THEN 'WARNING'
                ELSE 'HEALTHY'
            END AS HealthStatus,
            GETDATE() AS LastChecked,
            CASE 
                WHEN (SELECT counter_value FROM sys.dm_os_performance_counters WHERE counter_name = 'Memory Grants Pending' AND instance_name = @DatabaseName) >= 25 THEN 'Memory pressure detected - consider increasing SQL Server memory allocation'
                WHEN (SELECT counter_value FROM sys.dm_os_performance_counters WHERE counter_name = 'Memory Grants Pending' AND instance_name = @DatabaseName) >= 10 THEN 'Monitor memory usage - may need to optimize queries or increase memory'
                ELSE 'Memory usage is adequate'
            END AS Recommendation;
    END
    
    -- Capacity Health Check
    IF @HealthCheckType = 'COMPREHENSIVE' OR @HealthCheckType = 'CAPACITY'
    BEGIN
        PRINT 'Collecting capacity health metrics...';
        
        -- Database file sizes and growth
        SELECT 
            d.name AS DatabaseName,
            df.type_desc AS FileType,
            df.name AS FileName,
            df.physical_name AS FilePath,
            df.size * 8 / 1024 AS FileSizeMB,
            CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 8 / 1024 AS UsedSpaceMB,
            (df.size - CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint)) * 8 / 1024 AS FreeSpaceMB,
            CAST((CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 100.0 / df.size) AS decimal(5,2)) AS PercentUsed
        FROM sys.databases d
        CROSS APPLY sys.database_files df
        WHERE (@DatabaseName IS NULL OR d.name = @DatabaseName)
        AND d.database_id > 4
        ORDER BY d.name, df.type_desc;
        
        -- Disk space monitoring
        -- Note: This requires xp_cmdshell or other file system access methods
        PRINT 'Disk space monitoring requires file system access. Implement with xp_cmdshell or PowerShell.';
    END
    
    -- Availability Health Check
    IF @HealthCheckType = 'COMPREHENSIVE' OR @HealthCheckType = 'AVAILABILITY'
    BEGIN
        PRINT 'Collecting availability health metrics...';
        
        -- Database online/offline status
        INSERT INTO @HealthReport (
            DatabaseName, MetricCategory, MetricName, MetricValue, MetricUnit,
            ThresholdWarning, ThresholdCritical, HealthStatus, LastChecked, Recommendation
        )
        SELECT 
            d.name AS DatabaseName,
            'AVAILABILITY' AS MetricCategory,
            'Database_Status' AS MetricName,
            CASE WHEN d.state = 0 THEN 1.00 ELSE 0.00 END AS MetricValue,
            'STATUS' AS MetricUnit,
            1.00 AS ThresholdWarning,
            0.50 AS ThresholdCritical,
            CASE WHEN d.state = 0 THEN 'HEALTHY' ELSE 'CRITICAL' END AS HealthStatus,
            GETDATE() AS LastChecked,
            CASE 
                WHEN d.state = 0 THEN 'Database is online and accessible'
                WHEN d.state = 1 THEN 'Database is restoring - check restore progress'
                WHEN d.state = 2 THEN 'Database is recovering - check recovery progress'
                WHEN d.state = 3 THEN 'Database is not recovered - manual intervention required'
                WHEN d.state = 4 THEN 'Database is suspect - immediate attention required'
                ELSE 'Database state is unknown - manual investigation needed'
            END AS Recommendation
        FROM sys.databases d
        WHERE (@DatabaseName IS NULL OR d.name = @DatabaseName)
        AND d.database_id > 4;
        
        -- Active connections
        INSERT INTO @HealthReport (
            DatabaseName, MetricCategory, MetricName, MetricValue, MetricUnit,
            ThresholdWarning, ThresholdCritical, HealthStatus, LastChecked, Recommendation
        )
        SELECT 
            @DatabaseName AS DatabaseName,
            'AVAILABILITY' AS MetricCategory,
            'Active_Connections' AS MetricName,
            CAST((SELECT COUNT(*) FROM sys.dm_exec_sessions WHERE database_id = DB_ID(@DatabaseName)) AS decimal(18,2)) AS MetricValue,
            'COUNT' AS MetricUnit,
            100.00 AS ThresholdWarning,
            200.00 AS ThresholdCritical,
            CASE 
                WHEN (SELECT COUNT(*) FROM sys.dm_exec_sessions WHERE database_id = DB_ID(@DatabaseName)) >= 200 THEN 'CRITICAL'
                WHEN (SELECT COUNT(*) FROM sys.dm_exec_sessions WHERE database_id = DB_ID(@DatabaseName)) >= 100 THEN 'WARNING'
                ELSE 'HEALTHY'
            END AS HealthStatus,
            GETDATE() AS LastChecked,
            CASE 
                WHEN (SELECT COUNT(*) FROM sys.dm_exec_sessions WHERE database_id = DB_ID(@DatabaseName)) >= 200 THEN 'High connection count detected - investigate connection pooling and long-running queries'
                WHEN (SELECT COUNT(*) FROM sys.dm_exec_sessions WHERE database_id = DB_ID(@DatabaseName)) >= 100 THEN 'Monitor connection usage - may need to optimize connection management'
                ELSE 'Connection count is within normal limits'
            END AS Recommendation;
    END
    
    -- Generate comprehensive health summary
    IF @GenerateReport = 1
    BEGIN
        -- Overall health status
        WITH HealthSummary AS (
            SELECT 
                DatabaseName,
                COUNT(*) AS TotalMetrics,
                SUM(CASE WHEN HealthStatus = 'HEALTHY' THEN 1 ELSE 0 END) AS HealthyMetrics,
                SUM(CASE WHEN HealthStatus = 'WARNING' THEN 1 ELSE 0 END) AS WarningMetrics,
                SUM(CASE WHEN HealthStatus = 'CRITICAL' THEN 1 ELSE 0 END) AS CriticalMetrics,
                CAST(SUM(CASE WHEN HealthStatus = 'HEALTHY' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS decimal(5,2)) AS HealthPercentage
            FROM @HealthReport
            GROUP BY DatabaseName
        )
        SELECT 
            *,
            CASE 
                WHEN CriticalMetrics > 0 THEN 'CRITICAL'
                WHEN WarningMetrics > 0 THEN 'WARNING'
                ELSE 'HEALTHY'
            END AS OverallStatus
        FROM HealthSummary
        ORDER BY OverallStatus, HealthPercentage;
        
        -- Detailed metrics report
        SELECT 
            DatabaseName,
            MetricCategory,
            MetricName,
            MetricValue,
            MetricUnit,
            HealthStatus,
            Recommendation
        FROM @HealthReport
        WHERE HealthStatus != 'HEALTHY'
        ORDER BY HealthStatus, MetricCategory, DatabaseName;
        
        SET @ReportGenerated = 1;
    END
    
    -- Export to permanent table if requested
    IF @ExportToTable = 1
    BEGIN
        IF NOT EXISTS (SELECT 1 FROM sys.tables WHERE name = 'DatabaseHealthMetrics' AND schema_id = SCHEMA_ID('dbo'))
        BEGIN
            CREATE TABLE dbo.DatabaseHealthMetrics (
                MetricID int IDENTITY(1,1) PRIMARY KEY,
                DatabaseName sysname,
                MetricCategory nvarchar(50),
                MetricName nvarchar(100),
                MetricValue decimal(18,2),
                MetricUnit nvarchar(20),
                HealthStatus nvarchar(20),
                LastChecked datetime,
                ExportTime datetime DEFAULT GETDATE()
            );
        END
        
        INSERT INTO dbo.DatabaseHealthMetrics (
            DatabaseName, MetricCategory, MetricName, MetricValue, MetricUnit,
            HealthStatus, LastChecked
        )
        SELECT 
            DatabaseName, MetricCategory, MetricName, MetricValue, MetricUnit,
            HealthStatus, LastChecked
        FROM @HealthReport;
    END
    
    -- Return execution summary
    SELECT 
        @HealthCheckType AS HealthCheckType,
        DATEDIFF(second, @StartTime, GETDATE()) AS ExecutionTimeSeconds,
        COUNT(*) AS TotalMetricsCollected,
        SUM(CASE WHEN HealthStatus = 'HEALTHY' THEN 1 ELSE 0 END) AS HealthyMetrics,
        SUM(CASE WHEN HealthStatus = 'WARNING' THEN 1 ELSE 0 END) AS WarningMetrics,
        SUM(CASE WHEN HealthStatus = 'CRITICAL' THEN 1 ELSE 0 END) AS CriticalMetrics,
        CASE WHEN @ReportGenerated = 1 THEN 'COMPLETED_WITH_REPORT' ELSE 'COMPLETED_WITHOUT_REPORT' END AS Status;
    
    PRINT 'Database health monitoring completed. Check results for ' + CAST((SELECT COUNT(*) FROM @HealthReport) AS nvarchar(10)) + ' metrics.';
    
    RETURN 0;
END
GO
```

## Custom Maintenance Scripts {#custom-maintenance-scripts}

### Building Custom Solutions

While SQL Server provides built-in maintenance plans, custom scripts offer greater flexibility and control. Building custom maintenance solutions requires understanding your specific requirements, environment constraints, and operational procedures.

**Custom Solution Components:**

1. **Modular Script Design**: Create reusable components for different maintenance tasks
2. **Configuration Management**: Centralize configuration settings for easy maintenance
3. **Error Handling**: Robust error detection and recovery mechanisms
4. **Logging and Reporting**: Comprehensive logging for troubleshooting and auditing
5. **Scheduling Integration**: Seamless integration with SQL Server Agent or external schedulers

### Maintenance Script Framework

```sql
-- Custom maintenance script framework
IF NOT EXISTS (SELECT 1 FROM sys.procedures WHERE name = 'sp_CustomMaintenanceFramework')
BEGIN
    EXEC('
    CREATE PROCEDURE sp_CustomMaintenanceFramework
    (
        @Operation nvarchar(50),                    -- BACKUP, INDEX_MAINTENANCE, INTEGRITY_CHECK, CLEANUP
        @DatabaseName sysname = NULL,               -- Specific database or NULL for all
        @ExecutionMode nvarchar(20) = ''NORMAL'',   -- NORMAL, DRY_RUN, VERBOSE, TEST
        @MaxExecutionTime int = 360,                -- Maximum execution time in minutes
        @EmailNotifications bit = 1,                -- Send email notifications
        @LogLevel nvarchar(20) = ''INFO'',          -- DEBUG, INFO, WARNING, ERROR
        @ForceExecution bit = 0                     -- Override time restrictions
    )
    AS
    BEGIN
        SET NOCOUNT ON;
        
        DECLARE @ExecutionStartTime datetime = GETDATE();
        DECLARE @SessionID nvarchar(50) = NEWID();
        DECLARE @LogFilePath nvarchar(500) = ''C:\SQLMaintenance\Logs\'';
        DECLARE @ConfigTableName nvarchar(128) = ''CustomMaintenanceConfig'';
        DECLARE @HistoryTableName nvarchar(128) = ''CustomMaintenanceHistory'';
        
        -- Create configuration and history tables if they don''t exist
        IF NOT EXISTS (SELECT 1 FROM sys.tables WHERE name = @ConfigTableName AND schema_id = SCHEMA_ID(''dbo''))
        BEGIN
            EXEC(''CREATE TABLE dbo.'' + QUOTENAME(@ConfigTableName) + '' (
                ConfigKey nvarchar(100) PRIMARY KEY,
                ConfigValue nvarchar(500),
                Description nvarchar(1000),
                LastModified datetime DEFAULT GETDATE()
            )'');
            
            -- Insert default configuration
            INSERT INTO dbo.CustomMaintenanceConfig (ConfigKey, ConfigValue, Description)
            VALUES 
                (''BackupRetentionDays'', ''30'', ''Number of days to retain backup files''),
                (''IndexFragmentationThreshold'', ''30'', ''Rebuild indexes above this fragmentation percentage''),
                (''MaintenanceWindowStart'', ''02:00'', ''Maintenance window start time (24-hour format)''),
                (''MaintenanceWindowEnd'', ''06:00'', ''Maintenance window end time (24-hour format)''),
                (''EmailRecipients'', ''dba@company.com'', ''Email addresses for maintenance notifications''),
                (''LogRetentionDays'', ''90'', ''Number of days to retain maintenance logs'');
        END
        
        IF NOT EXISTS (SELECT 1 FROM sys.tables WHERE name = @HistoryTableName AND schema_id = SCHEMA_ID(''dbo''))
        BEGIN
            EXEC(''CREATE TABLE dbo.'' + QUOTENAME(@HistoryTableName) + '' (
                HistoryID int IDENTITY(1,1) PRIMARY KEY,
                SessionID nvarchar(50),
                Operation nvarchar(50),
                DatabaseName sysname,
                StartTime datetime,
                EndTime datetime,
                DurationSeconds int,
                Status nvarchar(20),
                ErrorMessage nvarchar(4000),
                RecordsAffected bigint,
                LogLevel nvarchar(20),
                ExecutionMode nvarchar(20)
            )'');
        END
        
        -- Log session start
        PRINT ''Starting maintenance operation: '' + @Operation + '' - Session ID: '' + @SessionID;
        PRINT ''Execution Mode: '' + @ExecutionMode;
        PRINT ''Database: '' + ISNULL(@DatabaseName, ''All Databases'');
        
        -- Check execution window restrictions
        IF @ForceExecution = 0
        BEGIN
            DECLARE @CurrentTime time = CONVERT(time, GETDATE());
            DECLARE @WindowStart time = (SELECT CAST(ConfigValue AS time) FROM dbo.CustomMaintenanceConfig WHERE ConfigKey = ''MaintenanceWindowStart'');
            DECLARE @WindowEnd time = (SELECT CAST(ConfigValue AS time) FROM dbo.CustomMaintenanceConfig WHERE ConfigKey = ''MaintenanceWindowEnd'');
            
            IF @CurrentTime < @WindowStart OR @CurrentTime > @WindowEnd
            BEGIN
                PRINT ''Current time '' + CAST(@CurrentTime AS nvarchar(10)) + '' is outside maintenance window '' + 
                      CAST(@WindowStart AS nvarchar(10)) + '' - '' + CAST(@WindowEnd AS nvarchar(10));
                PRINT ''Use @ForceExecution = 1 to override this restriction.'';
                
                INSERT INTO dbo.CustomMaintenanceHistory (
                    SessionID, Operation, DatabaseName, StartTime, EndTime, 
                    Status, ErrorMessage, ExecutionMode
                )
                VALUES (
                    @SessionID, @Operation, @DatabaseName, @ExecutionStartTime, GETDATE(),
                    ''SKIPPED'', ''Outside maintenance window'', @ExecutionMode
                );
                
                RETURN 0;
            END
        END
        
        -- Execute specific operation
        DECLARE @ResultStatus nvarchar(20) = ''UNKNOWN'';
        DECLARE @ErrorMessage nvarchar(4000) = '''';
        DECLARE @RecordsAffected bigint = 0;
        
        BEGIN TRY
            IF @Operation = ''BACKUP''
                EXEC sp_AutomatedBackup @DatabaseName, ''FULL'', 30, 1, 1, @BackupPath;
            
            ELSE IF @Operation = ''INDEX_MAINTENANCE''
                EXEC sp_ComprehensiveIndexMaintenance @DatabaseName;
            
            ELSE IF @Operation = ''INTEGRITY_CHECK''
                EXEC sp_ComprehensiveIntegrityCheck @DatabaseName, ''ALL'', ''NONE'', 0;
            
            ELSE IF @Operation = ''CLEANUP''
                EXEC sp_MaintenanceCleanup @CleanupType = ''ALL'', @RetentionDays = 30;
            
            ELSE
            BEGIN
                SET @ErrorMessage = ''Unknown operation: '' + @Operation;
                SET @ResultStatus = ''ERROR'';
                RAISERROR(@ErrorMessage, 16, 1);
            END
            
            SET @ResultStatus = ''SUCCESS'';
            PRINT ''Operation '' + @Operation + '' completed successfully.'';
            
        END TRY
        BEGIN CATCH
            SET @ResultStatus = ''ERROR'';
            SET @ErrorMessage = ERROR_MESSAGE();
            PRINT ''Operation '' + @Operation + '' failed: '' + @ErrorMessage;
        END CATCH
        
        -- Log session completion
        INSERT INTO dbo.CustomMaintenanceHistory (
            SessionID, Operation, DatabaseName, StartTime, EndTime, 
            DurationSeconds, Status, ErrorMessage, RecordsAffected, 
            LogLevel, ExecutionMode
        )
        VALUES (
            @SessionID, @Operation, @DatabaseName, @ExecutionStartTime, GETDATE(),
            DATEDIFF(second, @ExecutionStartTime, GETDATE()), @ResultStatus, 
            @ErrorMessage, @RecordsAffected, @LogLevel, @ExecutionMode
        );
        
        -- Send notification if configured
        IF @EmailNotifications = 1 AND (@ResultStatus != ''SUCCESS'')
        BEGIN
            DECLARE @EmailSubject nvarchar(200) = ''Maintenance Failed: '' + @Operation + '' ('' + @DatabaseName + '')'';
            DECLARE @EmailBody nvarchar(4000) = ''Maintenance operation failed.<br><br>Session ID: '' + @SessionID + 
                  ''<br>Operation: '' + @Operation + ''<br>Database: '' + ISNULL(@DatabaseName, ''All'') + 
                  ''<br>Status: '' + @ResultStatus + ''<br>Error: '' + @ErrorMessage + 
                  ''<br>Duration: '' + CAST(DATEDIFF(second, @ExecutionStartTime, GETDATE()) AS nvarchar(10)) + '' seconds'';
            
            -- Send email using database mail
            -- EXEC msdb.dbo.sp_send_dbmail @recipients = (SELECT ConfigValue FROM dbo.CustomMaintenanceConfig WHERE ConfigKey = ''EmailRecipients''),
            --                              @subject = @EmailSubject,
            --                              @body = @EmailBody,
            --                              @body_format = ''HTML'';
            
            PRINT ''Notification email would be sent for failed operation.'';
        END
        
        -- Return summary
        SELECT 
            @SessionID AS SessionID,
            @Operation AS Operation,
            @DatabaseName AS DatabaseName,
            @ExecutionMode AS ExecutionMode,
            @ExecutionStartTime AS StartTime,
            GETDATE() AS EndTime,
            DATEDIFF(second, @ExecutionStartTime, GETDATE()) AS DurationSeconds,
            @ResultStatus AS Status,
            @ErrorMessage AS ErrorMessage,
            @RecordsAffected AS RecordsAffected;
        
        PRINT ''Maintenance framework execution completed. Session ID: '' + @SessionID;
        
        RETURN CASE WHEN @ResultStatus = ''SUCCESS'' THEN 0 ELSE 1 END;
    END
    ');
END
GO
```

## Troubleshooting Scenarios {#troubleshooting-scenarios}

### Scenario 1: Maintenance Plan Failures

**Problem:** A weekly maintenance plan that normally completes successfully starts failing with timeout errors.

**Investigation Steps:**

```sql
-- Check maintenance plan execution history
SELECT TOP 10
    job_name,
    run_date,
    run_time,
    run_status,
    message
FROM msdb.dbo.sysjobhistory
WHERE job_id = (SELECT job_id FROM msdb.dbo.sysjobs WHERE name = 'Weekly Database Maintenance')
ORDER BY run_date DESC, run_time DESC;

-- Check recent error messages
SELECT TOP 20
    log_date,
    message_text,
    severity,
    is_drop_event
FROM sys.messages
WHERE language_id = 1033
AND (text LIKE '%timeout%' OR text LIKE '%deadlock%' OR text LIKE '%resource%')
ORDER BY log_date DESC;

-- Identify blocking sessions during maintenance window
SELECT 
    blocked_session.session_id AS BlockedSession,
    blocked_session.wait_type AS WaitType,
    blocked_session.wait_time AS WaitTime,
    blocking_session.session_id AS BlockingSession,
    blocking_session.login_name AS BlockingLogin,
    blocking_session.host_name AS BlockingHost,
    DB_NAME(blocked_session.database_id) AS DatabaseName,
    blocked_session.text AS BlockedStatement,
    blocking_session.text AS BlockingStatement
FROM sys.dm_exec_requests blocked_session
JOIN sys.dm_exec_requests blocking_session ON blocked_session.blocking_session_id = blocking_session.session_id
WHERE blocked_session.blocking_session_id > 0
AND blocked_session.database_id = DB_ID();
```

**Solution:** 
1. Increase maintenance plan timeout values
2. Schedule maintenance during lower activity periods
3. Break large maintenance tasks into smaller, sequential operations
4. Implement query timeout monitoring

### Scenario 2: Backup Job Performance Degradation

**Problem:** Backup operations that previously completed in 30 minutes now take over 2 hours.

**Investigation Steps:**

```sql
-- Check recent backup history and performance
SELECT TOP 20
    d.name AS DatabaseName,
    bs.type_desc AS BackupType,
    bs.backup_start_date AS StartTime,
    bs.backup_finish_date AS EndTime,
    DATEDIFF(minute, bs.backup_start_date, bs.backup_finish_date) AS DurationMinutes,
    bs.backup_size / 1024 / 1024 AS BackupSizeMB,
    bs.compressed_backup_size / 1024 / 1024 AS CompressedSizeMB,
    bs.compression_ratio,
    bs.device_type AS DeviceType,
    bs.user_name AS UserName
FROM msdb.dbo.backupset bs
INNER JOIN sys.databases d ON bs.database_name = d.name
WHERE bs.backup_start_date >= DATEADD(day, -7, GETDATE())
ORDER BY bs.backup_start_date DESC;

-- Check backup I/O statistics
SELECT 
    DB_NAME(database_id) AS DatabaseName,
    type_desc AS FileType,
    physical_name AS FilePath,
    size * 8 / 1024 AS CurrentSizeMB,
    CASE WHEN growth > 0 THEN size * 8 / 1024 ELSE 0 END AS GrowthAmountMB,
    CASE WHEN is_percent_growth = 1 THEN CAST(growth AS nvarchar(10)) + '%' ELSE CAST(growth * 8 / 1024 AS nvarchar(10)) + 'MB' END AS GrowthSetting,
    is_percent_growth,
    CASE WHEN max_size = -1 THEN 'UNLIMITED' 
         WHEN max_size = 0 THEN 'NO GROWTH'
         ELSE CAST(max_size * 8 / 1024 / 1024 AS nvarchar(10)) + 'MB'
    END AS MaxSizeSetting
FROM sys.database_files
ORDER BY DB_NAME(database_id), type_desc;

-- Check for I/O bottlenecks
SELECT TOP 20
    database_id,
    file_id,
    io_stall_read_ms / NULLIF(num_of_reads, 0) AS avg_read_latency_ms,
    io_stall_write_ms / NULLIF(num_of_writes, 0) AS avg_write_latency_ms,
    io_stall / NULLIF(num_of_reads + num_of_writes, 0) AS avg_latency_ms,
    num_of_reads,
    num_of_writes,
    bytes_read / 1024 / 1024 AS mb_read,
    bytes_written / 1024 / 1024 AS mb_written
FROM sys.dm_io_virtual_file_stats(NULL, NULL)
ORDER BY io_stall DESC;
```

**Solutions:**
1. Move backup files to faster storage
2. Implement backup compression
3. Consider backup striping across multiple files
4. Schedule backups during off-peak hours
5. Monitor and address I/O bottlenecks

### Scenario 3: Index Maintenance Overhead

**Problem:** Index maintenance operations are causing performance degradation during business hours.

**Investigation:**

```sql
-- Check index fragmentation trends
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent AS CurrentFragmentation,
    ips.page_count AS PageCount,
    ps.row_count AS TableRowCount,
    DATEDIFF(day, ps.last_used_date, GETDATE()) AS DaysSinceLastUsed
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
INNER JOIN sys.dm_db_partition_stats ps ON ips.object_id = ps.object_id AND ips.index_id = ps.index_id
WHERE ips.index_level = 0
AND i.type > 0
ORDER BY ips.avg_fragmentation_in_percent DESC;

-- Check maintenance plan execution patterns
SELECT TOP 10
    job_name,
    run_date,
    run_time,
    run_status,
    run_duration,
    message
FROM msdb.dbo.sysjobhistory
WHERE job_id = (SELECT job_id FROM msdb.dbo.sysjobs WHERE name LIKE '%Index%')
AND message LIKE '%fragment%'
ORDER BY run_date DESC, run_time DESC;

-- Identify recently modified tables that need maintenance
SELECT 
    t.name AS TableName,
    s.name AS SchemaName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent AS Fragmentation,
    ips.page_count AS PageCount,
    ps.row_count AS RowCount,
    DATEDIFF(hour, ps.last_used_date, GETDATE()) AS HoursSinceLastUsed
FROM sys.tables t
INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
INNER JOIN sys.indexes i ON t.object_id = i.object_id
INNER JOIN sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips ON i.object_id = ips.object_id AND i.index_id = ips.index_id
INNER JOIN sys.dm_db_partition_stats ps ON i.object_id = ps.object_id AND i.index_id = ps.index_id
WHERE t.is_ms_shipped = 0
AND i.type > 0
AND ips.index_level = 0
AND ips.avg_fragmentation_in_percent > 30
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

**Solutions:**
1. Implement maintenance during dedicated maintenance windows
2. Use online index operations for critical databases
3. Adjust fragmentation thresholds based on usage patterns
4. Implement selective index maintenance (high-impact indexes only)
5. Consider index compression to reduce fragmentation frequency

### Scenario 4: Database Growth Monitoring

**Problem:** Database files are growing rapidly and approaching disk space limits.

**Investigation:**

```sql
-- Track database growth over time
WITH GrowthHistory AS (
    SELECT 
        d.name AS DatabaseName,
        df.type_desc AS FileType,
        df.name AS FileName,
        df.size * 8 / 1024 AS CurrentSizeMB,
        df.growth * CASE WHEN df.is_percent_growth = 1 THEN 1 ELSE 8 END / 1024 AS GrowthAmountMB,
        df.max_size * 8 / 1024 AS MaxSizeMB,
        df.physical_name AS FilePath,
        CAST(df.growth AS nvarchar(10)) + CASE WHEN df.is_percent_growth = 1 THEN '%' ELSE 'MB' END AS GrowthSetting
    FROM sys.databases d
    CROSS APPLY sys.database_files df
    WHERE d.database_id > 4
)
SELECT *,
    CASE 
        WHEN MaxSizeMB > 0 AND CurrentSizeMB > 0 
        THEN CAST((CurrentSizeMB * 100.0 / MaxSizeMB) AS decimal(5,2)) 
        ELSE NULL 
    END AS PercentOfMaxSize,
    CASE 
        WHEN MaxSizeMB = 0 THEN 'UNLIMITED'
        WHEN CurrentSizeMB > 0.9 * MaxSizeMB THEN 'CRITICAL'
        WHEN CurrentSizeMB > 0.7 * MaxSizeMB THEN 'WARNING'
        ELSE 'NORMAL'
    END AS SizeStatus
FROM GrowthHistory
ORDER BY PercentOfMaxSize DESC;

-- Check disk space usage
EXEC sp_spaceused @updateusage = 'TRUE';

-- Monitor file auto-growth events
SELECT TOP 50
    database_id,
    file_id,
    event_type,
    total_event_count,
    total_error_count,
    sample_event_count,
    sample_error_count,
    last_event_time
FROM sys.dm_db_file_space_usage
ORDER BY last_event_time DESC;
```

**Solutions:**
1. Implement automated file growth monitoring
2. Set appropriate autogrowth settings
3. Schedule regular disk space cleanup
4. Implement file compression for large databases
5. Plan for proactive disk space allocation

## Best Practices {#best-practices}

### Maintenance Plan Design Principles

1. **Risk Assessment**: Evaluate the impact of maintenance operations on production systems
2. **Resource Planning**: Ensure adequate resources (CPU, memory, disk I/O) for maintenance tasks
3. **Scheduling Strategy**: Align maintenance windows with business operations and usage patterns
4. **Monitoring Integration**: Integrate maintenance activities with overall system monitoring
5. **Documentation Standards**: Maintain comprehensive documentation of all maintenance procedures

### Automation Best Practices

1. **Idempotent Scripts**: Design scripts that can be run multiple times without adverse effects
2. **Error Recovery**: Implement robust error handling and recovery mechanisms
3. **Logging Standards**: Use consistent logging formats and levels
4. **Notification Systems**: Configure appropriate alerting for different severity levels
5. **Testing Procedures**: Establish testing procedures before deploying to production

### Security Considerations

1. **Permission Management**: Use least-privilege principles for maintenance account permissions
2. **Credential Protection**: Securely manage service account credentials
3. **Audit Logging**: Maintain comprehensive audit trails of maintenance activities
4. **Network Security**: Protect maintenance communication channels
5. **Data Protection**: Ensure maintenance operations don't expose sensitive data

## Practical Exercises {#practical-exercises}

### Exercise 1: Build a Comprehensive Maintenance Plan

**Objective:** Create a complete maintenance plan that includes backup, index maintenance, integrity checks, and cleanup operations.

**Requirements:**
1. Design a maintenance workflow with proper task sequencing
2. Create scheduled jobs for each maintenance component
3. Implement error handling and notification systems
4. Build monitoring and reporting capabilities
5. Test the maintenance plan in a development environment

**Deliverables:**
- Complete T-SQL scripts for all maintenance components
- SQL Server Agent job definitions
- Monitoring queries and reports
- Documentation of the maintenance plan

### Exercise 2: Performance Impact Analysis

**Objective:** Analyze the performance impact of maintenance operations and optimize for minimal disruption.

**Activities:**
1. Run maintenance operations during different time periods
2. Measure impact on query performance, CPU usage, and I/O
3. Identify bottlenecks and optimization opportunities
4. Implement performance monitoring during maintenance
5. Develop strategies for reducing maintenance overhead

### Exercise 3: Health Monitoring Dashboard

**Objective:** Create a comprehensive health monitoring system for database maintenance.

**Components:**
1. Database health metrics collection
2. Trend analysis and alerting
3. Visual dashboard creation
4. Historical data retention and reporting
5. Integration with maintenance operations

### Exercise 4: Custom Maintenance Framework

**Objective:** Build a flexible, reusable maintenance framework.

**Features:**
1. Modular design for different maintenance operations
2. Configuration management system
3. Comprehensive logging and reporting
4. Scheduling integration
5. Error handling and recovery mechanisms

This comprehensive week 11 curriculum provides DBAs with the knowledge and tools needed to implement robust database maintenance plans. By mastering these concepts and techniques, you'll be able to maintain healthy, performant SQL Server databases while minimizing operational overhead and risk.
