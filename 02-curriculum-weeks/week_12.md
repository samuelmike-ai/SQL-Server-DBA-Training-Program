# Week 12: Troubleshooting and Diagnostics

## Table of Contents
1. [Introduction to SQL Server Troubleshooting](#introduction)
2. [Diagnostic Tools Overview](#diagnostic-tools)
3. [Performance Monitoring and Analysis](#performance-monitoring)
4. [Query Performance Troubleshooting](#query-performance)
5. [System Resource Analysis](#system-resources)
6. [Blocking and Deadlock Analysis](#blocking-deadlocks)
7. [Database Consistency Issues](#consistency-issues)
8. [High Availability Troubleshooting](#high-availability)
9. [Backup and Recovery Diagnostics](#backup-recovery)
10. [Advanced Diagnostic Techniques](#advanced-techniques)
11. [Troubleshooting Methodologies](#methodologies)
12. [Practical Troubleshooting Scenarios](#scenarios)

## Introduction to SQL Server Troubleshooting {#introduction}

SQL Server troubleshooting is both an art and a science that requires systematic approaches, deep technical knowledge, and methodical problem-solving skills. Effective troubleshooting involves identifying symptoms, analyzing root causes, and implementing appropriate solutions while minimizing disruption to business operations.

The complexity of modern SQL Server environments requires DBAs to master a comprehensive toolkit of diagnostic techniques, from basic query analysis to advanced performance monitoring and predictive diagnostics. This week focuses on building robust troubleshooting skills that enable rapid problem identification and resolution.

SQL Server provides extensive built-in diagnostic capabilities through Dynamic Management Views (DMVs), Extended Events, Performance Monitor counters, and various system stored procedures. Understanding how to leverage these tools effectively is crucial for maintaining optimal database performance and availability.

Key aspects of SQL Server troubleshooting include:

**Performance Troubleshooting:** Analyzing query execution times, identifying bottlenecks, and optimizing system performance
**Resource Analysis:** Monitoring CPU, memory, disk I/O, and network utilization
**Availability Issues:** Diagnosing connection problems, failover scenarios, and service interruptions  
**Consistency Problems:** Detecting corruption, integrity issues, and data anomalies
**Scalability Concerns:** Identifying capacity limits and growth-related performance degradation

### The Troubleshooting Mindset

Successful troubleshooting begins with the right mindset and approach:

1. **Systematic Methodology**: Use structured approaches rather than random exploration
2. **Data-Driven Decisions**: Base conclusions on measurable evidence rather than assumptions
3. **Root Cause Focus**: Identify underlying causes rather than treating symptoms
4. **Impact Assessment**: Understand business impact and prioritize accordingly
5. **Documentation**: Maintain detailed records of problems and solutions

## Diagnostic Tools Overview {#diagnostic-tools}

### Dynamic Management Views and Functions

SQL Server's Dynamic Management Views (DMVs) and Functions (DMFs) provide comprehensive access to system state, performance metrics, and operational data. These are essential tools for troubleshooting and performance analysis.

**Key DMV Categories:**

1. **Database and Instance Management**: System state and configuration
2. **Query Performance**: Execution plans, statistics, and resource usage
3. **Index Information**: Index usage, fragmentation, and optimization data
4. **I/O Statistics**: File-level and database-level I/O performance
5. **Wait Statistics**: Resource contention and bottleneck analysis
6. **Session Information**: Active sessions, blocking, and connection data

### System Diagnostic Stored Procedures

SQL Server includes numerous built-in stored procedures for system analysis and troubleshooting:

```sql
-- Comprehensive system information procedure
CREATE PROCEDURE sp_SystemDiagnostics
(
    @Scope nvarchar(50) = 'COMPREHENSIVE', -- COMPREHENSIVE, PERFORMANCE, CONFIGURATION, CAPACITY
    @IncludeExecutionPlans bit = 1,
    @OutputToTable bit = 1,
    @HistoryRetentionDays int = 30
)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @DiagnosticResults TABLE (
        Category nvarchar(50),
        Item nvarchar(100),
        Value nvarchar(500),
        Status nvarchar(20),
        Recommendation nvarchar(1000),
        LastChecked datetime DEFAULT GETDATE()
    );
    
    -- System Configuration Analysis
    IF @Scope = 'COMPREHENSIVE' OR @Scope = 'CONFIGURATION'
    BEGIN
        PRINT 'Collecting system configuration information...';
        
        -- SQL Server version and configuration
        INSERT INTO @DiagnosticResults (Category, Item, Value, Status, Recommendation)
        SELECT 
            'CONFIGURATION',
            'ProductVersion',
            CAST(SERVERPROPERTY('ProductVersion') AS nvarchar(50)),
            CASE WHEN CAST(SERVERPROPERTY('ProductVersion') AS nvarchar(10)) >= '15.0.0' THEN 'GOOD' ELSE 'UPGRADE_NEEDED' END,
            'Consider upgrading to SQL Server 2019 or later for latest features and performance improvements'
        UNION ALL
        SELECT 
            'CONFIGURATION',
            'ProductLevel',
            CAST(SERVERPROPERTY('ProductLevel') AS nvarchar(50)),
            CASE WHEN CAST(SERVERPROPERTY('ProductLevel') AS nvarchar(20)) LIKE '%SP%' OR CAST(SERVERPROPERTY('ProductLevel') AS nvarchar(20)) LIKE '%RTM%' THEN 'CHECK' ELSE 'GOOD' END,
            'Ensure all available service packs and updates are installed'
        UNION ALL
        SELECT 
            'CONFIGURATION',
            'InstanceName',
            CAST(SERVERPROPERTY('InstanceName') AS nvarchar(100)),
            CASE WHEN CAST(SERVERPROPERTY('InstanceName') AS nvarchar(100)) IS NULL THEN 'DEFAULT' ELSE 'NAMED' END,
            'Document named instances for proper management and access procedures';
        
        -- Server properties
        SELECT 
            @DiagnosticResults.Category,
            @DiagnosticResults.Item,
            @DiagnosticResults.Value,
            @DiagnosticResults.Status,
            @DiagnosticResults.Recommendation
        FROM @DiagnosticResults
        WHERE Category = 'CONFIGURATION'
        ORDER BY @DiagnosticResults.Item;
    END
    
    -- Performance Statistics Collection
    IF @Scope = 'COMPREHENSIVE' OR @Scope = 'PERFORMANCE'
    BEGIN
        PRINT 'Collecting performance statistics...';
        
        -- CPU and memory information
        SELECT 
            'PERFORMANCE' AS Category,
            'CPU_Count' AS Item,
            CAST(cpu_count AS nvarchar(20)) AS Value,
            CASE WHEN cpu_count >= 8 THEN 'GOOD' WHEN cpu_count >= 4 THEN 'ADEQUATE' ELSE 'LIMITED' END AS Status,
            'Number of CPU cores available to SQL Server' AS Recommendation
        FROM sys.dm_os_sys_info;
        
        SELECT 
            'PERFORMANCE' AS Category,
            'Physical_Memory_MB' AS Item,
            CAST(physical_memory_kb / 1024 AS nvarchar(20)) AS Value,
            CASE WHEN physical_memory_kb / 1024 >= 32768 THEN 'GOOD' WHEN physical_memory_kb / 1024 >= 8192 THEN 'ADEQUATE' ELSE 'LIMITED' END AS Status,
            'Total physical memory available to SQL Server' AS Recommendation
        FROM sys.dm_os_sys_info;
        
        -- Wait statistics analysis
        SELECT TOP 10
            'PERFORMANCE' AS Category,
            'WaitType' AS Item,
            wait_type AS Value,
            CASE 
                WHEN wait_type LIKE '%PAGEIOLATCH%' THEN 'IO_WARNING'
                WHEN wait_type LIKE '%WRITELOG%' THEN 'IO_WARNING'
                WHEN wait_type LIKE '%LCK%' THEN 'BLOCKING'
                WHEN wait_type LIKE '%CXPACKET%' THEN 'PARALLELISM'
                WHEN wait_type = 'SOS_SCHEDULER_YIELD' THEN 'CPU_CONTENTION'
                ELSE 'NORMAL'
            END AS Status,
            CASE 
                WHEN wait_type LIKE '%PAGEIOLATCH%' THEN 'High I/O wait times - consider faster storage or index optimization'
                WHEN wait_type LIKE '%WRITELOG%' THEN 'High transaction log I/O - consider faster storage or batch operations'
                WHEN wait_type LIKE '%LCK%' THEN 'Blocking detected - identify blocking sessions and optimize queries'
                WHEN wait_type LIKE '%CXPACKET%' THEN 'Parallel processing delays - consider adjusting MAXDOP settings'
                WHEN wait_type = 'SOS_SCHEDULER_YIELD' THEN 'CPU contention - consider query optimization or hardware upgrade'
                ELSE 'Normal wait patterns observed'
            END AS Recommendation
        FROM sys.dm_os_wait_stats
        WHERE wait_type NOT IN ('CLR_SEMAPHORE', 'LAZYWRITER_SLEEP', 'RESOURCE_QUEUE', 'SLEEP_TASK', 'SLEEP_SYSTEMTASK', 'WAITFOR')
        ORDER BY waiting_tasks_count DESC;
    end
    
    -- Database-specific diagnostics
    IF @Scope = 'COMPREHENSIVE' OR @Scope = 'CAPACITY'
    BEGIN
        PRINT 'Collecting database capacity information...';
        
        -- Database file information
        SELECT 
            'CAPACITY' AS Category,
            'DatabaseName' AS Item,
            d.name AS Value,
            CASE WHEN d.state = 0 THEN 'ONLINE' WHEN d.state = 1 THEN 'RESTORING' WHEN d.state = 2 THEN 'RECOVERING' WHEN d.state = 3 THEN 'RECOVERY_PENDING' WHEN d.state = 4 THEN 'SUSPECT' ELSE 'UNKNOWN' END AS Status,
            CASE WHEN d.state = 0 THEN 'Database is online and accessible'
                 WHEN d.state = 1 THEN 'Database is being restored'
                 WHEN d.state = 2 THEN 'Database is recovering from startup'
                 WHEN d.state = 3 THEN 'Database requires manual recovery'
                 WHEN d.state = 4 THEN 'Database is suspect - immediate attention required'
                 ELSE 'Database state requires investigation'
            END AS Recommendation
        FROM sys.databases d
        WHERE d.database_id > 4 -- Exclude system databases
        ORDER BY d.name;
        
        -- File growth and sizing analysis
        SELECT 
            'CAPACITY' AS Category,
            'FileInfo' AS Item,
            DB_NAME(database_id) + ':' + type_desc + ' - ' + name AS Value,
            CASE 
                WHEN is_percent_growth = 0 AND max_size > 0 AND CAST(size * 8 / 1024.0 / 1024.0 AS decimal(10,2)) > 0.8 * CAST(max_size * 8 / 1024.0 / 1024.0 AS decimal(10,2)) THEN 'CRITICAL'
                WHEN is_percent_growth = 0 AND max_size > 0 AND CAST(size * 8 / 1024.0 / 1024.0 AS decimal(10,2)) > 0.6 * CAST(max_size * 8 / 1024.0 / 1024.0 AS decimal(10,2)) THEN 'WARNING'
                WHEN is_percent_growth = 1 AND growth > 20 THEN 'WARNING'
                ELSE 'GOOD'
            END AS Status,
            CASE 
                WHEN is_percent_growth = 0 AND max_size > 0 AND CAST(size * 8 / 1024.0 / 1024.0 AS decimal(10,2)) > 0.8 * CAST(max_size * 8 / 1024.0 / 1024.0 AS decimal(10,2)) THEN 'Database file approaching maximum size - immediate attention required'
                WHEN is_percent_growth = 0 AND max_size > 0 AND CAST(size * 8 / 1024.0 / 1024.0 AS decimal(10,2)) > 0.6 * CAST(max_size * 8 / 1024.0 / 1024.0 AS decimal(10,2)) THEN 'Database file growth should be monitored'
                WHEN is_percent_growth = 1 AND growth > 20 THEN 'Large percentage growth may cause performance issues'
                ELSE 'File sizing appears appropriate'
            END AS Recommendation
        FROM sys.dm_db_file_space_usage
        ORDER BY DB_NAME(database_id), type_desc;
    end
    
    -- Return summary statistics
    SELECT 
        COUNT(*) AS TotalItemsChecked,
        SUM(CASE WHEN Status = 'GOOD' THEN 1 ELSE 0 END) AS GoodItems,
        SUM(CASE WHEN Status = 'WARNING' THEN 1 ELSE 0 END) AS WarningItems,
        SUM(CASE WHEN Status = 'CRITICAL' OR Status LIKE '%_CRITICAL' OR Status LIKE '%WARNING' THEN 1 ELSE 0 END) AS ItemsRequiringAttention,
        SUM(CASE WHEN Status = 'GOOD' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS HealthPercentage
    FROM @DiagnosticResults;
    
    -- Store results in history table if requested
    IF @OutputToTable = 1
    BEGIN
        IF NOT EXISTS (SELECT 1 FROM sys.tables WHERE name = 'SystemDiagnosticHistory' AND schema_id = SCHEMA_ID('dbo'))
        BEGIN
            CREATE TABLE dbo.SystemDiagnosticHistory (
                DiagnosticID int IDENTITY(1,1) PRIMARY KEY,
                DiagnosticDate datetime DEFAULT GETDATE(),
                Scope nvarchar(50),
                Category nvarchar(50),
                Item nvarchar(100),
                Value nvarchar(500),
                Status nvarchar(20),
                Recommendation nvarchar(1000),
                ExpirationDate datetime
            );
        END
        
        -- Clean up old records
        DELETE FROM dbo.SystemDiagnosticHistory 
        WHERE DiagnosticDate < DATEADD(day, -@HistoryRetentionDays, GETDATE());
        
        -- Insert current results
        INSERT INTO dbo.SystemDiagnosticHistory (
            Scope, Category, Item, Value, Status, Recommendation, ExpirationDate
        )
        SELECT 
            @Scope, Category, Item, Value, Status, Recommendation,
            DATEADD(day, @HistoryRetentionDays, GETDATE())
        FROM @DiagnosticResults;
    END
    
    PRINT 'System diagnostics completed for scope: ' + @Scope;
    
    RETURN 0;
END
GO
```

### Extended Events for Diagnostics

Extended Events provide powerful, lightweight diagnostic capabilities for monitoring SQL Server behavior and performance:

```sql
-- Extended Events diagnostic session setup
CREATE PROCEDURE sp_SetupDiagnosticExtendedEvents
(
    @SessionName nvarchar(128) = 'SQLServer_Diagnostics',
    @CreateSession bit = 1,
    @IncludeQueryPlans bit = 1,
    @IncludeWaits bit = 1,
    @IncludeErrors bit = 1,
    @MaxMemoryMB int = 256,
    @RetentionTimeMinutes int = 1440 -- 24 hours
)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL nvarchar(max);
    DECLARE @EventSessionExists bit;
    
    -- Check if session already exists
    SELECT @EventSessionExists = CASE WHEN EXISTS (
        SELECT 1 FROM sys.server_event_sessions WHERE name = @SessionName
    ) THEN 1 ELSE 0 END;
    
    IF @EventSessionExists = 1 AND @CreateSession = 1
    BEGIN
        PRINT 'Dropping existing session: ' + @SessionName;
        SET @SQL = 'DROP EVENT SESSION ' + QUOTENAME(@SessionName) + ' ON SERVER;';
        EXEC sp_executesql @SQL;
    END
    
    IF @CreateSession = 1
    BEGIN
        PRINT 'Creating Extended Events session: ' + @SessionName;
        
        -- Build the session creation script
        SET @SQL = 'CREATE EVENT SESSION ' + QUOTENAME(@SessionName) + ' ON SERVER 
        ADD EVENT sqlserver.error_reported (
            ACTION (
                sqlserver.database_id,
                sqlserver.database_name,
                sqlserver.session_id,
                sqlserver.username,
                sqlserver.client_app_name,
                sqlserver.client_hostname,
                sqlserver.sql_text,
                sqlserver.query_hash,
                sqlserver.query_plan_hash
            )
            WHERE ([severity] >= (15))
        ),';
        
        IF @IncludeQueryPlans = 1
        BEGIN
            SET @SQL = @SQL + '
        ADD EVENT sqlserver.sql_statement_completed (
            ACTION (
                sqlserver.database_id,
                sqlserver.database_name,
                sqlserver.session_id,
                sqlserver.username,
                sqlserver.client_app_name,
                sqlserver.client_hostname,
                sqlserver.sql_text,
                sqlserver.query_hash
            )
            WHERE (duration > (1000000) OR cpu_time > (100000) OR logical_reads > (10000))
        ),';
        END
        
        IF @IncludeWaits = 1
        BEGIN
            SET @SQL = @SQL + '
        ADD EVENT sqlserver.wait_completed (
            ACTION (
                sqlserver.database_id,
                sqlserver.database_name,
                sqlserver.session_id,
                sqlserver.username,
                sqlserver.client_app_name
            )
            WHERE (duration > (500000)) -- 500ms waits
        ),';
        END
        
        IF @IncludeErrors = 1
        BEGIN
            SET @SQL = @SQL + '
        ADD EVENT sqlserver.lock_deadlock (
            ACTION (
                sqlserver.database_id,
                sqlserver.database_name,
                sqlserver.session_id,
                sqlserver.username,
                sqlserver.client_app_name
            )
        ),';
        END
        
        -- Remove trailing comma
        SET @SQL = LEFT(@SQL, LEN(@SQL) - 1);
        
        -- Add remaining session options
        SET @SQL = @SQL + '
        ADD TARGET package0.ring_buffer (
            SET max_memory = ' + CAST(@MaxMemoryMB * 1024 AS nvarchar(10)) + ' -- size in KB
        )
        WITH (
            MAX_MEMORY = ' + CAST(@MaxMemoryMB AS nvarchar(10)) + ' MB,
            EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS,
            MAX_DISPATCH_LATENCY = 30 SECONDS,
            MAX_EVENT_SIZE = 0 KB,
            MEMORY_PARTITION_MODE = NONE,
            TRACK_CAUSALITY = ON,
            STARTUP_STATE = OFF
        );';
        
        EXEC sp_executesql @SQL;
        PRINT 'Extended Events session created: ' + @SessionName;
    END
    
    -- Start the session if it exists
    IF EXISTS (SELECT 1 FROM sys.server_event_sessions WHERE name = @SessionName)
    BEGIN
        PRINT 'Starting Extended Events session: ' + @SessionName;
        EXEC('ALTER EVENT SESSION ' + QUOTENAME(@SessionName) + ' ON SERVER STATE = START;');
    END
    
    -- Display current session status
    SELECT 
        name AS SessionName,
        CASE WHEN event_retention_mode_desc = 'ALLOW_SINGLE_EVENT_LOSS' THEN 'GOOD'
             WHEN event_retention_mode_desc = 'ALLOW_MULTIPLE_EVENT_LOSS' THEN 'WARNING'
             ELSE 'UNKNOWN'
        END AS RetentionMode,
        max_memory,
        CASE WHEN create_event_descriptor IS NOT NULL THEN 'CREATED' ELSE 'NOT_CREATED' END AS SessionStatus,
        CASE WHEN NULLIF(CAST(target_data AS xml) IS NOT NULL, 1) IS NOT NULL THEN 'HAS_DATA' ELSE 'NO_DATA' END AS HasData
    FROM sys.server_event_sessions ses
    LEFT JOIN sys.server_event_session_events ev ON ses.name = ev.name
    LEFT JOIN sys.server_event_session_targets tar ON ses.name = tar.name
    WHERE ses.name = @SessionName;
    
    RETURN 0;
END
GO
```

## Performance Monitoring and Analysis {#performance-monitoring}

### Query Performance Analysis

Query performance troubleshooting requires systematic analysis of execution times, resource consumption, and execution plans. SQL Server provides several tools for this analysis:

**Key Performance Analysis Components:**
1. **Query Execution Statistics**: Duration, CPU time, logical reads, writes
2. **Execution Plan Analysis**: Operators, estimated vs actual rows, warnings
3. **Index Usage Analysis**: Index effectiveness and missing indexes
4. **Resource Utilization**: Memory grants, parallelism, wait times

```sql
-- Comprehensive query performance analysis procedure
CREATE PROCEDURE sp_QueryPerformanceAnalysis
(
    @DatabaseName sysname = NULL,
    @MinExecutionCount int = 5,
    @MinDurationSeconds decimal(10,3) = 1.000,
    @IncludeExecutionPlans bit = 1,
    @OutputTopQueries int = 20,
    @AnalysisType nvarchar(20) = 'COMPREHENSIVE' -- COMPREHENSIVE, DURATION, CPU, IO, MEMORY
)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @PerformanceResults TABLE (
        QueryID int IDENTITY(1,1),
        DatabaseName sysname,
        QueryText nvarchar(max),
        ExecutionCount bigint,
        TotalDurationSeconds decimal(18,3),
        AvgDurationSeconds decimal(18,3),
        MinDurationSeconds decimal(18,3),
        MaxDurationSeconds decimal(18,3),
        TotalCpuTimeMs bigint,
        AvgCpuTimeMs bigint,
        TotalLogicalReads bigint,
        AvgLogicalReads bigint,
        TotalLogicalWrites bigint,
        AvgLogicalWrites bigint,
        TotalPhysicalReads bigint,
        AvgPhysicalReads bigint,
        QueryHash nvarchar(32),
        PlanHandle varbinary(64),
        CreationTime datetime,
        LastExecutionTime datetime,
        QueryPlanText nvarchar(max),
        ExecutionPlan nvarchar(max)
    );
    
    -- Collect query performance statistics
    IF @AnalysisType = 'COMPREHENSIVE' OR @AnalysisType = 'DURATION'
    BEGIN
        PRINT 'Collecting query performance statistics...';
        
        -- High-duration query analysis
        INSERT INTO @PerformanceResults (
            DatabaseName, QueryText, ExecutionCount, TotalDurationSeconds, AvgDurationSeconds,
            MinDurationSeconds, MaxDurationSeconds, TotalCpuTimeMs, AvgCpuTimeMs,
            TotalLogicalReads, AvgLogicalReads, TotalLogicalWrites, AvgLogicalWrites,
            TotalPhysicalReads, AvgPhysicalReads, QueryHash, PlanHandle,
            CreationTime, LastExecutionTime
        )
        SELECT TOP (@OutputTopQueries)
            DB_NAME(qs.database_id) AS DatabaseName,
            SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 
                     ((CASE qs.statement_end_offset
                       WHEN -1 THEN DATALENGTH(st.text)
                       ELSE qs.statement_end_offset
                     END - qs.statement_start_offset)/2) + 1) AS QueryText,
            qs.execution_count AS ExecutionCount,
            qs.total_elapsed_time / 1000000.0 AS TotalDurationSeconds,
            qs.total_elapsed_time / 1000000.0 / qs.execution_count AS AvgDurationSeconds,
            qs.min_elapsed_time / 1000000.0 AS MinDurationSeconds,
            qs.max_elapsed_time / 1000000.0 AS MaxDurationSeconds,
            qs.total_worker_time / 1000 AS TotalCpuTimeMs,
            qs.total_worker_time / 1000 / qs.execution_count AS AvgCpuTimeMs,
            qs.total_logical_reads AS TotalLogicalReads,
            qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
            qs.total_logical_writes AS TotalLogicalWrites,
            qs.total_logical_writes / qs.execution_count AS AvgLogicalWrites,
            qs.total_physical_reads AS TotalPhysicalReads,
            qs.total_physical_reads / qs.execution_count AS AvgPhysicalReads,
            CAST(qs.query_hash AS nvarchar(32)) AS QueryHash,
            qs.plan_handle AS PlanHandle,
            qs.creation_time AS CreationTime,
            qs.last_execution_time AS LastExecutionTime
        FROM sys.dm_exec_query_stats qs
        CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
        WHERE (@DatabaseName IS NULL OR DB_NAME(qs.database_id) = @DatabaseName)
        AND qs.execution_count >= @MinExecutionCount
        AND qs.total_elapsed_time / 1000000.0 >= @MinDurationSeconds
        ORDER BY qs.total_elapsed_time / qs.execution_count DESC;
        
        -- Get execution plans if requested
        IF @IncludeExecutionPlans = 1
        BEGIN
            UPDATE pr
            SET QueryPlanText = CAST(qp.query_plan AS nvarchar(max)),
                ExecutionPlan = CAST(qp.query_plan AS nvarchar(max))
            FROM @PerformanceResults pr
            CROSS APPLY sys.dm_exec_text_query_plan(pr.PlanHandle, NULL, NULL) qp
            WHERE pr.PlanHandle IS NOT NULL;
        END
    END
    
    -- Resource-intensive queries analysis
    IF @AnalysisType = 'COMPREHENSIVE' OR @AnalysisType = 'CPU'
    BEGIN
        SELECT TOP (@OutputTopQueries)
            'CPU_ANALYSIS' AS AnalysisType,
            DB_NAME(qs.database_id) AS DatabaseName,
            SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 
                     ((CASE qs.statement_end_offset
                       WHEN -1 THEN DATALENGTH(st.text)
                       ELSE qs.statement_end_offset
                     END - qs.statement_start_offset)/2) + 1) AS QueryText,
            qs.total_worker_time / 1000 / qs.execution_count AS AvgCpuTimeMs,
            qs.execution_count AS ExecutionCount,
            qs.total_worker_time / 1000 AS TotalCpuTimeMs,
            qs.creation_time AS CreationTime
        FROM sys.dm_exec_query_stats qs
        CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
        WHERE (@DatabaseName IS NULL OR DB_NAME(qs.database_id) = @DatabaseName)
        AND qs.execution_count >= @MinExecutionCount
        ORDER BY qs.total_worker_time / qs.execution_count DESC;
    END
    
    -- I/O intensive queries analysis
    IF @AnalysisType = 'COMPREHENSIVE' OR @AnalysisType = 'IO'
    BEGIN
        SELECT TOP (@OutputTopQueries)
            'IO_ANALYSIS' AS AnalysisType,
            DB_NAME(qs.database_id) AS DatabaseName,
            SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 
                     ((CASE qs.statement_end_offset
                       WHEN -1 THEN DATALENGTH(st.text)
                       ELSE qs.statement_end_offset
                     END - qs.statement_start_offset)/2) + 1) AS QueryText,
            qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
            qs.total_logical_writes / qs.execution_count AS AvgLogicalWrites,
            qs.total_physical_reads / qs.execution_count AS AvgPhysicalReads,
            qs.execution_count AS ExecutionCount
        FROM sys.dm_exec_query_stats qs
        CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
        WHERE (@DatabaseName IS NULL OR DB_NAME(qs.database_id) = @DatabaseName)
        AND qs.execution_count >= @MinExecutionCount
        ORDER BY (qs.total_logical_reads + qs.total_logical_writes) / qs.execution_count DESC;
    END
    
    -- Query optimization recommendations
    WITH MissingIndexRecommendations AS (
        SELECT 
            DB_NAME(mid.database_id) AS DatabaseName,
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
            (mid.user_seeks + mid.user_scans) * mid.avg_total_user_cost AS ImpactScore
        FROM sys.dm_db_missing_index_details mid
        WHERE (@DatabaseName IS NULL OR DB_NAME(mid.database_id) = @DatabaseName)
        AND (mid.user_seeks + mid.user_scans) >= 10
    )
    SELECT TOP (@OutputTopQueries)
        'MISSING_INDEXES' AS AnalysisType,
        DatabaseName,
        TableName,
        EqualityColumns,
        InequalityColumns,
        IncludedColumns,
        UserSeeks,
        UserScans,
        UserLookups,
        ImpactScore,
        'CREATE NONCLUSTERED INDEX IX_' + REPLACE(REPLACE(REPLACE(EqualityColumns + '_' + InequalityColumns, '][', '_'), '[', ''), ']', '') + 
        ' ON ' + TableName + ' (' + ISNULL(EqualityColumns, '') + ISNULL(CASE WHEN InequalityColumns IS NOT NULL THEN ',' + InequalityColumns ELSE '' END, '') + ')' +
        CASE WHEN IncludedColumns IS NOT NULL THEN ' INCLUDE (' + IncludedColumns + ')' ELSE '' END AS RecommendedIndex
    FROM MissingIndexRecommendations
    ORDER BY ImpactScore DESC;
    
    -- Return comprehensive performance report
    SELECT 
        QueryID,
        DatabaseName,
        LEFT(REPLACE(REPLACE(QueryText, CHAR(13), ' '), CHAR(10), ' '), 200) + '...' AS QueryPreview,
        ExecutionCount,
        AvgDurationSeconds,
        TotalCpuTimeMs,
        AvgLogicalReads,
        AvgLogicalWrites,
        AvgPhysicalReads,
        QueryHash,
        LastExecutionTime,
        CASE 
            WHEN AvgDurationSeconds > 10 THEN 'CRITICAL'
            WHEN AvgDurationSeconds > 3 THEN 'WARNING'
            WHEN AvgLogicalReads > 100000 THEN 'HIGH_IO'
            WHEN AvgPhysicalReads > 10000 THEN 'HIGH_PHYSICAL_IO'
            ELSE 'NORMAL'
        END AS PerformanceStatus,
        CASE 
            WHEN AvgDurationSeconds > 10 THEN 'Query execution time exceeds acceptable limits - investigate query optimization'
            WHEN AvgDurationSeconds > 3 THEN 'Query performance should be reviewed - consider index optimization'
            WHEN AvgLogicalReads > 100000 THEN 'High logical I/O suggests missing indexes or inefficient query structure'
            WHEN AvgPhysicalReads > 10000 THEN 'High physical I/O may indicate insufficient memory or missing indexes'
            ELSE 'Query performance appears acceptable'
        END AS Recommendation
    FROM @PerformanceResults
    ORDER BY AvgDurationSeconds DESC;
    
    -- Return performance summary statistics
    SELECT 
        COUNT(*) AS TotalQueriesAnalyzed,
        AVG(AvgDurationSeconds) AS AverageQueryDuration,
        MAX(AvgDurationSeconds) AS SlowestQueryDuration,
        SUM(ExecutionCount) AS TotalExecutions,
        SUM(CASE WHEN AvgDurationSeconds > 10 THEN 1 ELSE 0 END) AS CriticalQueries,
        SUM(CASE WHEN AvgDurationSeconds > 3 AND AvgDurationSeconds <= 10 THEN 1 ELSE 0 END) AS WarningQueries
    FROM @PerformanceResults;
    
    -- Store results for historical analysis
    INSERT INTO dbo.QueryPerformanceHistory (
        DatabaseName, QueryHash, ExecutionCount, AvgDurationSeconds, 
        AvgCpuTimeMs, AvgLogicalReads, AnalysisDate
    )
    SELECT 
        DatabaseName, QueryHash, ExecutionCount, AvgDurationSeconds,
        AvgCpuTimeMs, AvgLogicalReads, GETDATE()
    FROM @PerformanceResults;
    
    PRINT 'Query performance analysis completed. Analyzed ' + CAST((SELECT COUNT(*) FROM @PerformanceResults) AS nvarchar(10)) + ' queries.';
    
    RETURN 0;
END
GO
```

### Execution Plan Analysis

Execution plan analysis is crucial for understanding why queries perform poorly and identifying optimization opportunities:

```sql
-- Execution plan analysis procedure
CREATE PROCEDURE sp_ExecutionPlanAnalysis
(
    @DatabaseName sysname = NULL,
    @IncludePlanCacheAnalysis bit = 1,
    @IncludeMissingIndexes bit = 1,
    @IncludeWarnings bit = 1,
    @IncludeSpoolOperations bit = 1,
    @QueryHash nvarchar(32) = NULL,
    @PlanHandle varbinary(64) = NULL
)
AS
BEGIN
    SET NOCOUNT ON;
    
    PRINT 'Starting execution plan analysis...';
    
    -- Plan cache analysis
    IF @IncludePlanCacheAnalysis = 1
    BEGIN
        PRINT 'Analyzing plan cache...';
        
        SELECT TOP 20
            'PLAN_CACHE' AS AnalysisType,
            DB_NAME(cp.database_id) AS DatabaseName,
            cp.usecounts AS UseCount,
            cp.refcounts AS RefCount,
            cp.cacheobjtype AS CacheObjectType,
            cp.objtype AS ObjectType,
            qs.execution_count AS ExecutionCount,
            qs.total_elapsed_time / 1000000.0 / qs.execution_count AS AvgDurationSeconds,
            qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
            qs.total_physical_reads / qs.execution_count AS AvgPhysicalReads,
            qs.query_hash AS QueryHash,
            qs.creation_time AS PlanCreationTime,
            qs.last_execution_time AS LastExecutionTime,
            CAST(qp.query_plan AS nvarchar(max)) AS ExecutionPlan
        FROM sys.dm_exec_cached_plans cp
        CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
        CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
        INNER JOIN sys.dm_exec_query_stats qs ON cp.plan_handle = qs.plan_handle
        WHERE (@DatabaseName IS NULL OR DB_NAME(cp.database_id) = @DatabaseName)
        AND (@QueryHash IS NULL OR CAST(qs.query_hash AS nvarchar(32)) = @QueryHash)
        AND (@PlanHandle IS NULL OR cp.plan_handle = @PlanHandle)
        ORDER BY qs.execution_count DESC;
    END
    
    -- Missing indexes analysis
    IF @IncludeMissingIndexes = 1
    BEGIN
        PRINT 'Analyzing missing index recommendations...';
        
        WITH MissingIndexes AS (
            SELECT 
                'MISSING_INDEX' AS IssueType,
                DB_NAME(mid.database_id) AS DatabaseName,
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
                mid.avg_total_user_cost AS AvgTotalUserCost,
                mid.avg_user_impact AS AvgUserImpact,
                (mid.user_seeks + mid.user_scans) * mid.avg_total_user_cost AS ImpactScore,
                'CREATE NONCLUSTERED INDEX IX_' + 
                REPLACE(REPLACE(REPLACE(mid.equality_columns + ISNULL('_' + mid.inequality_columns, ''), '][', '_'), '[', ''), ']', '') + 
                ' ON ' + mid.statement + 
                ' (' + ISNULL(mid.equality_columns, '') + 
                CASE WHEN mid.inequality_columns IS NOT NULL THEN ',' + mid.inequality_columns ELSE '' END + ')' +
                CASE WHEN mid.included_columns IS NOT NULL THEN ' INCLUDE (' + mid.included_columns + ')' ELSE '' END AS RecommendedIndex
            FROM sys.dm_db_missing_index_details mid
            WHERE (@DatabaseName IS NULL OR DB_NAME(mid.database_id) = @DatabaseName)
            AND (mid.user_seeks + mid.user_scans) >= 5
        )
        SELECT TOP 20
            IssueType,
            DatabaseName,
            TableName,
            EqualityColumns,
            InequalityColumns,
            IncludedColumns,
            UserSeeks,
            UserScans,
            UserLookups,
            AvgTotalUserCost,
            AvgUserImpact,
            ImpactScore,
            RecommendedIndex,
            CASE 
                WHEN ImpactScore > 10000 THEN 'HIGH_PRIORITY'
                WHEN ImpactScore > 1000 THEN 'MEDIUM_PRIORITY'
                ELSE 'LOW_PRIORITY'
            END AS Priority,
            'Create suggested index to improve query performance' AS Recommendation
        FROM MissingIndexes
        ORDER BY ImpactScore DESC;
    END
    
    -- Execution plan warnings analysis
    IF @IncludeWarnings = 1
    BEGIN
        PRINT 'Analyzing execution plan warnings...';
        
        -- Note: This would require parsing XML execution plans for warnings
        -- Sample implementation for common warnings
        
        SELECT 
            'PLAN_WARNINGS' AS AnalysisType,
            'WARNING_DETECTED' AS WarningType,
            'TableScan' AS OperationType,
            'Query contains table scans - consider index optimization' AS WarningMessage,
            'HIGH' AS Severity,
            'Add appropriate indexes or rewrite query to use existing indexes' AS Recommendation;
    END
    
    -- Spool operation analysis
    IF @IncludeSpoolOperations = 1
    BEGIN
        PRINT 'Analyzing spool operations...';
        
        -- Spool operations can indicate missing indexes or inefficient queries
        SELECT 
            'SPOOL_ANALYSIS' AS AnalysisType,
            'TableSpool' AS OperationType,
            'Spool operation detected - may indicate missing index or inefficient query plan' AS AnalysisResult,
            'MEDIUM' AS Severity,
            'Review query structure and missing index recommendations' AS Recommendation;
    END
    
    -- Query execution statistics with plan information
    IF @QueryHash IS NOT NULL OR @PlanHandle IS NOT NULL
    BEGIN
        PRINT 'Getting detailed query execution information...';
        
        SELECT TOP 10
            'DETAILED_QUERY' AS AnalysisType,
            DB_NAME(qs.database_id) AS DatabaseName,
            qs.query_hash AS QueryHash,
            qs.plan_handle AS PlanHandle,
            qs.execution_count AS ExecutionCount,
            qs.total_elapsed_time / 1000000.0 / qs.execution_count AS AvgDurationSeconds,
            qs.total_worker_time / 1000 / qs.execution_count AS AvgCpuTimeMs,
            qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
            qs.total_physical_reads / qs.execution_count AS AvgPhysicalReads,
            qs.creation_time AS CreationTime,
            qs.last_execution_time AS LastExecutionTime,
            CAST(qp.query_plan AS nvarchar(max)) AS ExecutionPlan,
            SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 
                     ((CASE qs.statement_end_offset
                       WHEN -1 THEN DATALENGTH(st.text)
                       ELSE qs.statement_end_offset
                     END - qs.statement_start_offset)/2) + 1) AS QueryText
        FROM sys.dm_exec_query_stats qs
        CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
        CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
        WHERE (@QueryHash IS NULL OR CAST(qs.query_hash AS nvarchar(32)) = @QueryHash)
        AND (@PlanHandle IS NULL OR qs.plan_handle = @PlanHandle)
        ORDER BY qs.last_execution_time DESC;
    END
    
    -- Plan cache summary statistics
    SELECT 
        'PLAN_CACHE_SUMMARY' AS AnalysisType,
        COUNT(DISTINCT DB_NAME(cp.database_id)) AS DatabaseCount,
        COUNT(DISTINCT cp.plan_handle) AS TotalPlans,
        SUM(qs.execution_count) AS TotalExecutions,
        AVG(qs.total_elapsed_time / 1000000.0 / qs.execution_count) AS AvgQueryDuration,
        MAX(qs.total_elapsed_time / 1000000.0 / qs.execution_count) AS SlowestQueryDuration,
        SUM(CASE WHEN qs.total_elapsed_time / 1000000.0 / qs.execution_count > 10 THEN 1 ELSE 0 END) AS SlowQueries
    FROM sys.dm_exec_cached_plans cp
    INNER JOIN sys.dm_exec_query_stats qs ON cp.plan_handle = qs.plan_handle
    WHERE (@DatabaseName IS NULL OR DB_NAME(cp.database_id) = @DatabaseName);
    
    PRINT 'Execution plan analysis completed.';
    
    RETURN 0;
END
GO
```

## Query Performance Troubleshooting {#query-performance}

### Slow Query Investigation Methodology

When investigating slow queries, follow a systematic approach:

1. **Problem Identification**: Gather execution statistics and identify the specific queries causing issues
2. **Plan Analysis**: Examine execution plans for inefficiencies, missing indexes, and warnings
3. **Resource Analysis**: Check CPU, memory, and I/O utilization for the query
4. **Index Evaluation**: Review existing indexes and identify missing index recommendations
5. **Statistical Analysis**: Verify statistics freshness and accuracy
6. **Optimization**: Apply appropriate fixes and measure impact

### Advanced Query Performance Troubleshooting

```sql
-- Comprehensive slow query troubleshooting procedure
CREATE PROCEDURE sp_SlowQueryTroubleshooting
(
    @DatabaseName sysname = NULL,
    @QueryHash nvarchar(32) = NULL,
    @MinExecutionTime decimal(10,3) = 1.000,
    @IncludeDetailedAnalysis bit = 1,
    @IncludeBlockingAnalysis bit = 1,
    @IncludeWaitAnalysis bit = 1,
    @GenerateRecommendations bit = 1
)
AS
BEGIN
    SET NOCOUNT ON;
    
    PRINT 'Starting comprehensive slow query troubleshooting...';
    
    -- Step 1: Identify problematic queries
    PRINT 'Step 1: Identifying slow queries...';
    
    WITH QueryPerformance AS (
        SELECT TOP 50
            qs.query_hash AS QueryHash,
            DB_NAME(qs.database_id) AS DatabaseName,
            SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 
                     ((CASE qs.statement_end_offset
                       WHEN -1 THEN DATALENGTH(st.text)
                       ELSE qs.statement_end_offset
                     END - qs.statement_start_offset)/2) + 1) AS QueryText,
            qs.execution_count AS ExecutionCount,
            qs.total_elapsed_time / 1000000.0 AS TotalDurationSeconds,
            qs.total_elapsed_time / 1000000.0 / qs.execution_count AS AvgDurationSeconds,
            qs.min_elapsed_time / 1000000.0 AS MinDurationSeconds,
            qs.max_elapsed_time / 1000000.0 AS MaxDurationSeconds,
            qs.total_worker_time / 1000 AS TotalCpuTimeMs,
            qs.total_worker_time / 1000 / qs.execution_count AS AvgCpuTimeMs,
            qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
            qs.total_physical_reads / qs.execution_count AS AvgPhysicalReads,
            qs.total_logical_writes / qs.execution_count AS AvgLogicalWrites,
            qs.plan_handle AS PlanHandle,
            qs.creation_time AS CreationTime,
            qs.last_execution_time AS LastExecutionTime,
            DATEDIFF(day, qs.last_execution_time, GETDATE()) AS DaysSinceLastExecution,
            -- Performance categorization
            CASE 
                WHEN qs.total_elapsed_time / 1000000.0 / qs.execution_count > 30 THEN 'CRITICAL'
                WHEN qs.total_elapsed_time / 1000000.0 / qs.execution_count > 10 THEN 'HIGH'
                WHEN qs.total_elapsed_time / 1000000.0 / qs.execution_count > 3 THEN 'MEDIUM'
                WHEN qs.total_elapsed_time / 1000000.0 / qs.execution_count > 1 THEN 'LOW'
                ELSE 'NORMAL'
            END AS PerformanceCategory
        FROM sys.dm_exec_query_stats qs
        CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
        WHERE (@DatabaseName IS NULL OR DB_NAME(qs.database_id) = @DatabaseName)
        AND (@QueryHash IS NULL OR CAST(qs.query_hash AS nvarchar(32)) = @QueryHash)
        AND qs.execution_count >= 5
        AND qs.total_elapsed_time / 1000000.0 / qs.execution_count >= @MinExecutionTime
        ORDER BY qs.total_elapsed_time / 1000000.0 / qs.execution_count DESC
    )
    SELECT 
        QueryHash,
        DatabaseName,
        LEFT(REPLACE(REPLACE(QueryText, CHAR(13), ' '), CHAR(10), ' '), 300) + '...' AS QueryPreview,
        ExecutionCount,
        AvgDurationSeconds,
        AvgCpuTimeMs,
        AvgLogicalReads,
        AvgPhysicalReads,
        PerformanceCategory,
        DaysSinceLastExecution,
        LastExecutionTime,
        'Analyze execution plan and missing indexes' AS NextAction
    FROM QueryPerformance;
    
    -- Step 2: Detailed analysis for critical queries
    IF @IncludeDetailedAnalysis = 1
    BEGIN
        PRINT 'Step 2: Performing detailed analysis of critical queries...';
        
        -- For each slow query, get detailed execution information
        DECLARE @AnalysisQuery nvarchar(max);
        DECLARE @CurrentQueryHash nvarchar(32);
        
        -- Cursor to analyze each slow query
        DECLARE query_cursor CURSOR FOR
        SELECT DISTINCT CAST(query_hash AS nvarchar(32))
        FROM sys.dm_exec_query_stats qs
        WHERE (@DatabaseName IS NULL OR DB_NAME(qs.database_id) = @DatabaseName)
        AND (@QueryHash IS NULL OR CAST(qs.query_hash AS nvarchar(32)) = @QueryHash)
        AND qs.execution_count >= 5
        AND qs.total_elapsed_time / 1000000.0 / qs.execution_count >= 3.0
        ORDER BY CAST(query_hash AS nvarchar(32));
        
        OPEN query_cursor;
        FETCH NEXT FROM query_cursor INTO @CurrentQueryHash;
        
        WHILE @@FETCH_STATUS = 0
        BEGIN
            PRINT 'Analyzing query hash: ' + @CurrentQueryHash;
            
            -- Get execution plan details
            SELECT TOP 5
                'EXECUTION_PLAN_DETAIL' AS AnalysisType,
                @CurrentQueryHash AS QueryHash,
                DB_NAME(qs.database_id) AS DatabaseName,
                qs.execution_count AS ExecutionCount,
                qs.total_elapsed_time / 1000000.0 / qs.execution_count AS AvgDurationSeconds,
                qs.total_worker_time / 1000 / qs.execution_count AS AvgCpuTimeMs,
                qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
                qs.total_physical_reads / qs.execution_count AS AvgPhysicalReads,
                CAST(qp.query_plan AS nvarchar(max)) AS ExecutionPlan
            FROM sys.dm_exec_query_stats qs
            CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
            CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
            WHERE CAST(qs.query_hash AS nvarchar(32)) = @CurrentQueryHash
            ORDER BY qs.last_execution_time DESC;
            
            FETCH NEXT FROM query_cursor INTO @CurrentQueryHash;
        END
        
        CLOSE query_cursor;
        DEALLOCATE query_cursor;
    END
    
    -- Step 3: Blocking analysis
    IF @IncludeBlockingAnalysis = 1
    BEGIN
        PRINT 'Step 3: Analyzing blocking and waits...';
        
        -- Current blocking analysis
        SELECT 
            'BLOCKING_ANALYSIS' AS AnalysisType,
            blocking_session.session_id AS BlockingSessionID,
            blocked_session.session_id AS BlockedSessionID,
            blocking_session.login_name AS BlockingUser,
            blocked_session.login_name AS BlockedUser,
            blocking_session.program_name AS BlockingProgram,
            blocked_session.program_name AS BlockedProgram,
            blocked_session.wait_type AS WaitType,
            blocked_session.wait_time AS WaitTime,
            blocked_session.last_wait_type AS LastWaitType,
            blocking_session.cpu_time AS BlockingCpuTime,
            blocked_session.cpu_time AS BlockedCpuTime,
            CASE WHEN blocking_session.database_id = blocked_session.database_id 
                 THEN DB_NAME(blocking_session.database_id) 
                 ELSE 'MULTIPLE' 
            END AS DatabaseName,
            SUBSTRING(blocked_session.text, 1, 200) AS BlockedQuery,
            SUBSTRING(blocking_session.text, 1, 200) AS BlockingQuery
        FROM sys.dm_exec_requests blocked_session
        JOIN sys.dm_exec_requests blocking_session ON blocked_session.blocking_session_id = blocking_session.session_id
        CROSS APPLY sys.dm_exec_sql_text(blocked_session.sql_handle) blocked_session
        CROSS APPLY sys.dm_exec_sql_text(blocking_session.sql_handle) blocking_session
        WHERE blocked_session.blocking_session_id > 0;
    END
    
    -- Step 4: Wait statistics analysis
    IF @IncludeWaitAnalysis = 1
    BEGIN
        PRINT 'Step 4: Analyzing wait statistics...';
        
        SELECT TOP 20
            'WAIT_STATISTICS' AS AnalysisType,
            wait_type AS WaitType,
            waiting_tasks_count AS WaitingTasksCount,
            wait_time_ms / 1000.0 AS TotalWaitTimeSeconds,
            signal_wait_time_ms / 1000.0 AS SignalWaitTimeSeconds,
            wait_time_ms / 1000.0 / waiting_tasks_count AS AvgWaitTimeSeconds,
            (wait_time_ms - signal_wait_time_ms) / 1000.0 / waiting_tasks_count AS ResourceWaitTimeSeconds,
            CASE 
                WHEN wait_type LIKE '%PAGEIOLATCH%' THEN 'DISK_IO'
                WHEN wait_type LIKE '%WRITELOG%' THEN 'TRANSACTION_LOG_IO'
                WHEN wait_type LIKE '%LCK%' THEN 'LOCKING'
                WHEN wait_type LIKE '%CXPACKET%' THEN 'PARALLELISM'
                WHEN wait_type = 'SOS_SCHEDULER_YIELD' THEN 'CPU'
                WHEN wait_type LIKE '%LATCH%' THEN 'LATCH'
                WHEN wait_type LIKE '%SUSPEND%' THEN 'SUSPENSION'
                ELSE 'OTHER'
            END AS WaitCategory,
            'Investigate high wait times for ' + wait_type AS Recommendation
        FROM sys.dm_os_wait_stats
        WHERE wait_type NOT IN ('CLR_SEMAPHORE', 'LAZYWRITER_SLEEP', 'RESOURCE_QUEUE', 'SLEEP_TASK', 'SLEEP_SYSTEMTASK', 'WAITFOR')
        AND waiting_tasks_count > 0
        ORDER BY wait_time_ms DESC;
    END
    
    -- Step 5: Generate optimization recommendations
    IF @GenerateRecommendations = 1
    BEGIN
        PRINT 'Step 5: Generating optimization recommendations...';
        
        WITH OptimizationRecommendations AS (
            -- Missing index recommendations
            SELECT 
                'MISSING_INDEX' AS RecommendationType,
                DB_NAME(mid.database_id) AS DatabaseName,
                mid.statement AS TableName,
                'CREATE NONCLUSTERED INDEX IX_' + REPLACE(REPLACE(REPLACE(mid.equality_columns + ISNULL('_' + mid.inequality_columns, ''), '][', '_'), '[', ''), ']', '') + 
                ' ON ' + mid.statement + ' (' + ISNULL(mid.equality_columns, '') + 
                CASE WHEN mid.inequality_columns IS NOT NULL THEN ',' + mid.inequality_columns ELSE '' END + ')' +
                CASE WHEN mid.included_columns IS NOT NULL THEN ' INCLUDE (' + mid.included_columns + ')' ELSE '' END AS RecommendationText,
                (mid.user_seeks + mid.user_scans) * mid.avg_total_user_cost AS ImpactScore,
                'HIGH' AS Priority,
                'Create missing index to improve query performance' AS Action
            FROM sys.dm_db_missing_index_details mid
            WHERE (@DatabaseName IS NULL OR DB_NAME(mid.database_id) = @DatabaseName)
            
            UNION ALL
            
            -- Statistics update recommendations
            SELECT 
                'OUTDATED_STATISTICS' AS RecommendationType,
                DB_NAME(s.object_id) AS DatabaseName,
                OBJECT_NAME(s.object_id) AS TableName,
                'UPDATE STATISTICS ' + QUOTENAME(OBJECT_SCHEMA_NAME(s.object_id)) + '.' + QUOTENAME(OBJECT_NAME(s.object_id)) +
                CASE WHEN s.stats_id != 1 THEN ' ' + QUOTENAME(s.name) ELSE '' END + 
                ' WITH FULLSCAN;' AS RecommendationText,
                ISNULL(sp.modification_counter, 0) AS ImpactScore,
                CASE WHEN ISNULL(sp.modification_counter, 0) > 1000 THEN 'HIGH' 
                     WHEN ISNULL(sp.modification_counter, 0) > 100 THEN 'MEDIUM' 
                     ELSE 'LOW' 
                END AS Priority,
                'Update statistics to improve query optimization' AS Action
            FROM sys.stats s
            CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
            WHERE (@DatabaseName IS NULL OR DB_NAME(s.object_id) = @DatabaseName)
            AND s.auto_created = 1
            AND ISNULL(sp.modification_counter, 0) > 0
            AND DATEDIFF(day, sp.last_updated, GETDATE()) > 7
            
            UNION ALL
            
            -- Index fragmentation recommendations
            SELECT 
                'INDEX_FRAGMENTATION' AS RecommendationType,
                DB_NAME(ips.database_id) AS DatabaseName,
                OBJECT_NAME(ips.object_id) + '.' + i.name AS IndexName,
                CASE 
                    WHEN ips.avg_fragmentation_in_percent > 40 THEN 'ALTER INDEX ' + QUOTENAME(i.name) + ' ON ' + QUOTENAME(SCHEMA_NAME(t.schema_id)) + '.' + QUOTENAME(OBJECT_NAME(ips.object_id)) + ' REBUILD WITH (ONLINE = ON);'
                    WHEN ips.avg_fragmentation_in_percent > 10 THEN 'ALTER INDEX ' + QUOTENAME(i.name) + ' ON ' + QUOTENAME(SCHEMA_NAME(t.schema_id)) + '.' + QUOTENAME(OBJECT_NAME(ips.object_id)) + ' REORGANIZE;'
                    ELSE 'No action needed'
                END AS RecommendationText,
                ips.avg_fragmentation_in_percent AS ImpactScore,
                CASE WHEN ips.avg_fragmentation_in_percent > 40 THEN 'HIGH' 
                     WHEN ips.avg_fragmentation_in_percent > 20 THEN 'MEDIUM' 
                     ELSE 'LOW' 
                END AS Priority,
                'Address index fragmentation to improve query performance' AS Action
            FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
            INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
            INNER JOIN sys.tables t ON i.object_id = t.object_id
            WHERE (@DatabaseName IS NULL OR DB_NAME(ips.database_id) = @DatabaseName)
            AND i.type > 0
            AND ips.index_level = 0
            AND ips.avg_fragmentation_in_percent > 10
        )
        SELECT TOP 20
            RecommendationType,
            DatabaseName,
            TableName,
            LEFT(RecommendationText, 200) AS RecommendationText,
            ImpactScore,
            Priority,
            Action
        FROM OptimizationRecommendations
        ORDER BY 
            CASE Priority WHEN 'HIGH' THEN 1 WHEN 'MEDIUM' THEN 2 WHEN 'LOW' THEN 3 ELSE 4 END,
            ImpactScore DESC;
    END
    
    -- Return troubleshooting summary
    SELECT 
        'TROUBLESHOOTING_SUMMARY' AS SummaryType,
        @DatabaseName AS TargetDatabase,
        GETDATE() AS AnalysisDate,
        (SELECT COUNT(*) FROM sys.dm_exec_query_stats qs WHERE (@DatabaseName IS NULL OR DB_NAME(qs.database_id) = @DatabaseName)) AS TotalQueriesInCache,
        (SELECT COUNT(*) FROM sys.dm_exec_query_stats qs WHERE (@DatabaseName IS NULL OR DB_NAME(qs.database_id) = @DatabaseName) AND qs.total_elapsed_time / 1000000.0 / qs.execution_count > 10) AS CriticalQueries,
        (SELECT COUNT(*) FROM sys.dm_exec_requests WHERE blocking_session_id > 0) AS BlockedSessions,
        (SELECT COUNT(DISTINCT wait_type) FROM sys.dm_os_wait_stats WHERE waiting_tasks_count > 0) AS ActiveWaitTypes,
        'Troubleshooting analysis completed successfully' AS Status;
    
    PRINT 'Slow query troubleshooting completed.';
    
    RETURN 0;
END
GO
```

## System Resource Analysis {#system-resources}

### CPU Utilization Analysis

High CPU utilization can indicate inefficient queries, insufficient hardware, or configuration issues. Systematic CPU analysis helps identify root causes:

```sql
-- CPU utilization analysis procedure
CREATE PROCEDURE sp_CpuUtilizationAnalysis
(
    @AnalysisPeriodMinutes int = 60,
    @IncludeDetailedBreakdown bit = 1,
    @IncludeQueryLevelAnalysis bit = 1,
    @GenerateRecommendations bit = 1
)
AS
BEGIN
    SET NOCOUNT ON;
    
    PRINT 'Starting CPU utilization analysis...';
    
    -- Current CPU utilization snapshot
    SELECT 
        'CPU_SNAPSHOT' AS AnalysisType,
        sqlserver_start_time AS SqlServerStartTime,
        DATEDIFF(minute, sqlserver_start_time, GETDATE()) AS SqlServerUptimeMinutes,
        cpu_count AS CpuCount,
        hyperthread_ratio AS HyperthreadRatio,
        (physical_memory_kb / 1024) AS PhysicalMemoryMB,
        (virtual_memory_kb / 1024) AS VirtualMemoryMB,
        committed_kb / 1024 AS CommittedMemoryMB,
        committed_target_kb / 1024 AS CommittedTargetMemoryMB
    FROM sys.dm_os_sys_info;
    
    -- Historical CPU utilization (if available through Extended Events or Performance Monitor)
    SELECT 
        'CPU_HISTORICAL' AS AnalysisType,
        'Real-time CPU monitoring requires Extended Events or Performance Monitor integration' AS Note,
        'Use Extended Events session for continuous CPU tracking' AS Recommendation;
    
    -- Top CPU-consuming queries
    IF @IncludeQueryLevelAnalysis = 1
    BEGIN
        PRINT 'Analyzing top CPU-consuming queries...';
        
        SELECT TOP 20
            'TOP_CPU_QUERIES' AS AnalysisType,
            DB_NAME(qs.database_id) AS DatabaseName,
            SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 
                     ((CASE qs.statement_end_offset
                       WHEN -1 THEN DATALENGTH(st.text)
                       ELSE qs.statement_end_offset
                     END - qs.statement_start_offset)/2) + 1) AS QueryPreview,
            qs.execution_count AS ExecutionCount,
            qs.total_worker_time / 1000 AS TotalCpuTimeMs,
            qs.total_worker_time / 1000 / qs.execution_count AS AvgCpuTimeMs,
            qs.total_elapsed_time / 1000000.0 / qs.execution_count AS AvgDurationSeconds,
            qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
            qs.creation_time AS CreationTime,
            qs.last_execution_time AS LastExecutionTime,
            CAST(qs.query_hash AS nvarchar(32)) AS QueryHash
        FROM sys.dm_exec_query_stats qs
        CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
        ORDER BY qs.total_worker_time / qs.execution_count DESC;
    END
    
    -- CPU wait statistics
    IF @IncludeDetailedBreakdown = 1
    BEGIN
        PRINT 'Analyzing CPU-related wait statistics...';
        
        SELECT TOP 20
            'CPU_WAIT_ANALYSIS' AS AnalysisType,
            wait_type AS WaitType,
            waiting_tasks_count AS WaitingTasksCount,
            wait_time_ms / 1000.0 AS TotalWaitTimeSeconds,
            wait_time_ms / 1000.0 / waiting_tasks_count AS AvgWaitTimeSeconds,
            CASE 
                WHEN wait_type = 'SOS_SCHEDULER_YIELD' THEN 'Scheduler yield - CPU contention'
                WHEN wait_type LIKE '%CXPACKET%' THEN 'Parallel processing wait'
                WHEN wait_type LIKE '%CXCONSUMER%' THEN 'Parallel consumer wait'
                WHEN wait_type LIKE '%THREADPOOL%' THEN 'Worker thread pool exhaustion'
                WHEN wait_type = 'BACKUPIO' THEN 'Backup I/O wait'
                ELSE 'Other wait type'
            END AS WaitDescription,
            CASE 
                WHEN wait_type = 'SOS_SCHEDULER_YIELD' THEN 'Query optimization, index tuning'
                WHEN wait_type LIKE '%CXPACKET%' THEN 'Consider MAXDOP adjustment'
                WHEN wait_type LIKE '%THREADPOOL%' THEN 'Increase worker threads or optimize queries'
                WHEN wait_type = 'BACKUPIO' THEN 'Backup optimization or storage performance'
                ELSE 'Further investigation needed'
            END AS Recommendation
        FROM sys.dm_os_wait_stats
        WHERE wait_type IN ('SOS_SCHEDULER_YIELD', 'CXPACKET', 'CXCONSUMER', 'THREADPOOL', 'BACKUPIO')
        AND waiting_tasks_count > 0
        ORDER BY wait_time_ms DESC;
    END
    
    -- Process-level CPU utilization
    SELECT TOP 10
        'PROCESS_CPU_USAGE' AS AnalysisType,
        session_id AS SessionId,
        login_name AS LoginName,
        program_name AS ProgramName,
        host_name AS HostName,
        cpu_time AS CpuTimeMs,
        total_scheduled_time AS TotalScheduledTimeMs,
        total_elapsed_time AS TotalElapsedTimeMs,
        (cpu_time * 100.0 / NULLIF(total_elapsed_time, 0)) AS CpuUtilizationPercent,
        status,
        wait_type,
        wait_time AS WaitTimeMs,
        SUBSTRING(sql_text.text, 1, 100) AS QueryPreview
    FROM sys.dm_exec_sessions s
    LEFT JOIN sys.dm_exec_requests r ON s.session_id = r.session_id
    LEFT JOIN sys.dm_exec_sql_text(r.sql_handle) sql_text ON r.sql_handle IS NOT NULL
    WHERE s.session_id != @@SPID
    AND s.is_user_process = 1
    AND s.cpu_time > 0
    ORDER BY s.cpu_time DESC;
    
    -- Parallel execution analysis
    SELECT 
        'PARALLEL_EXECUTION_ANALYSIS' AS AnalysisType,
        qs.query_hash AS QueryHash,
        qs.execution_count AS ExecutionCount,
        qs.max_dop AS MaxDopUsed,
        qs.min_dop AS MinDopUsed,
        qs.avg_dop AS AvgDopUsed,
        qs.total_elapsed_time / 1000000.0 / qs.execution_count AS AvgDurationSeconds,
        qs.total_worker_time / 1000 / qs.execution_count AS AvgCpuTimeMs,
        qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
        CASE 
            WHEN qs.avg_dop > 16 THEN 'HIGH_PARALLELISM'
            WHEN qs.avg_dop > 8 THEN 'MODERATE_PARALLELISM'
            ELSE 'LOW_PARALLELISM'
        END AS ParallelismLevel,
        CASE 
            WHEN qs.avg_dop > 16 THEN 'Consider reducing MAXDOP or optimizing query'
            WHEN qs.avg_dop > 8 THEN 'Monitor parallel execution impact'
            ELSE 'Parallelism appears appropriate'
        END AS Recommendation
    FROM sys.dm_exec_query_stats qs
    WHERE qs.max_dop > 1
    AND qs.execution_count > 5
    ORDER BY qs.total_worker_time / qs.execution_count DESC;
    
    -- Generate CPU optimization recommendations
    IF @GenerateRecommendations = 1
    BEGIN
        PRINT 'Generating CPU optimization recommendations...';
        
        WITH CpuRecommendations AS (
            -- High parallelism queries
            SELECT 
                'HIGH_PARALLELISM' AS RecommendationType,
                DB_NAME(qs.database_id) AS DatabaseName,
                'Query ' + CAST(qs.query_hash AS nvarchar(32)) AS AffectedObject,
                'Query uses high degree of parallelism (avg: ' + CAST(qs.avg_dop AS nvarchar(10)) + ')' AS Issue,
                'Consider using OPTION (MAXDOP 8) or optimize query structure' AS Recommendation,
                'MEDIUM' AS Priority
            FROM sys.dm_exec_query_stats qs
            WHERE qs.avg_dop > 16
            AND qs.execution_count > 10
            
            UNION ALL
            
            -- CPU-intensive queries
            SELECT 
                'CPU_INTENSIVE_QUERIES' AS RecommendationType,
                DB_NAME(qs.database_id) AS DatabaseName,
                'Query ' + CAST(qs.query_hash AS nvarchar(32)) AS AffectedObject,
                'High CPU time per execution: ' + CAST(qs.total_worker_time / 1000 / qs.execution_count AS nvarchar(10)) + 'ms' AS Issue,
                'Optimize query execution plan, add indexes, or rewrite query' AS Recommendation,
                CASE WHEN qs.total_worker_time / 1000 / qs.execution_count > 10000 THEN 'HIGH'
                     WHEN qs.total_worker_time / 1000 / qs.execution_count > 1000 THEN 'MEDIUM'
                     ELSE 'LOW'
                END AS Priority
            FROM sys.dm_exec_query_stats qs
            WHERE qs.total_worker_time / 1000 / qs.execution_count > 1000
            AND qs.execution_count > 5
            
            UNION ALL
            
            -- Schema optimization recommendations
            SELECT 
                'SCHEMA_OPTIMIZATION' AS RecommendationType,
                'SYSTEM' AS DatabaseName,
                'Server configuration' AS AffectedObject,
                'MAXDOP setting may need adjustment based on CPU cores' AS Issue,
                'Review and adjust MAXDOP setting for optimal parallel performance' AS Recommendation,
                'HIGH' AS Priority
            FROM sys.dm_os_sys_info
            WHERE cpu_count > 8
        )
        SELECT 
            RecommendationType,
            DatabaseName,
            AffectedObject,
            Issue,
            Recommendation,
            Priority,
            'CPU optimization recommendation generated' AS ActionRequired
        FROM CpuRecommendations
        ORDER BY 
            CASE Priority WHEN 'HIGH' THEN 1 WHEN 'MEDIUM' THEN 2 WHEN 'LOW' THEN 3 ELSE 4 END;
    END
    
    PRINT 'CPU utilization analysis completed.';
    
    RETURN 0;
END
GO
```

### Memory Analysis

Memory-related performance issues often manifest as slow queries, high page life expectancy, or buffer pool pressure. Comprehensive memory analysis helps identify optimization opportunities:

```sql
-- Memory utilization analysis procedure
CREATE PROCEDURE sp_MemoryUtilizationAnalysis
(
    @IncludeBufferPoolAnalysis bit = 1,
    @IncludePlanCacheAnalysis bit = 1,
    @IncludeMemoryGrantsAnalysis bit = 1,
    @GenerateRecommendations bit = 1
)
AS
BEGIN
    SET NOCOUNT ON;
    
    PRINT 'Starting memory utilization analysis...';
    
    -- Overall memory configuration and usage
    SELECT 
        'MEMORY_CONFIGURATION' AS AnalysisType,
        osi.physical_memory_kb / 1024 AS PhysicalMemoryMB,
        osi.virtual_memory_kb / 1024 AS VirtualMemoryMB,
        osi.committed_kb / 1024 AS CommittedMemoryMB,
        osi.committed_target_kb / 1024 AS CommittedTargetMB,
        osi.accessible_memory_kb / 1024 AS AccessibleMemoryMB,
        osi.total_virtual_address_space_kb / 1024 AS TotalVirtualAddressSpaceMB,
        CASE 
            WHEN osi.committed_kb > osi.committed_target_kb * 1.1 THEN 'COMMIT_PRESSURE'
            WHEN osi.committed_kb < osi.committed_target_kb * 0.9 THEN 'MEMORY_AVAILABLE'
            ELSE 'MEMORY_NORMAL'
        END AS MemoryPressureStatus,
        CASE 
            WHEN osi.committed_kb > osi.committed_target_kb * 1.1 THEN 'SQL Server has committed more memory than target - monitor for memory pressure'
            WHEN osi.committed_kb < osi.committed_target_kb * 0.9 THEN 'SQL Server has committed less memory than target - additional memory available'
            ELSE 'Memory allocation appears normal'
        END AS MemoryStatusDescription
    FROM sys.dm_os_sys_info osi;
    
    -- Buffer pool analysis
    IF @IncludeBufferPoolAnalysis = 1
    BEGIN
        PRINT 'Analyzing buffer pool usage...';
        
        -- Buffer pool pages by database
        SELECT TOP 20
            'BUFFER_POOL_ANALYSIS' AS AnalysisType,
            DB_NAME(database_id) AS DatabaseName,
            file_id AS FileId,
            page_id AS PageId,
            page_type_desc AS PageType,
            row_count AS RowCount,
            free_space_in_bytes AS FreeSpaceBytes,
            is_modified AS IsModified,
            (free_space_in_bytes * 100.0 / 8192) AS FreeSpaceKB
        FROM sys.dm_os_buffer_descriptors
        WHERE database_id > 4
        ORDER BY free_space_in_bytes DESC;
        
        -- Buffer pool pressure indicators
        SELECT 
            'BUFFER_POOL_PRESSURE' AS AnalysisType,
            'PageLifeExpectancy' AS MetricName,
            osm.counter_value AS MetricValue,
            CASE 
                WHEN osm.counter_value < 300 THEN 'CRITICAL'
                WHEN osm.counter_value < 600 THEN 'WARNING'
                ELSE 'NORMAL'
            END AS Status,
            CASE 
                WHEN osm.counter_value < 300 THEN 'Page life expectancy is very low - immediate memory pressure'
                WHEN osm.counter_value < 600 THEN 'Page life expectancy is low - monitor memory usage'
                ELSE 'Page life expectancy is within acceptable range'
            END AS Description
        FROM sys.dm_os_performance_counters osm
        WHERE osm.object_name = 'SQLServer:Buffer Manager'
        AND osm.counter_name = 'Page life expectancy';
        
        -- Lazy writes analysis
        SELECT 
            'LAZY_WRITES_ANALYSIS' AS AnalysisType,
            'Lazy Writes/sec' AS MetricName,
            osm.counter_value AS LazyWritesPerSecond,
            CASE 
                WHEN osm.counter_value > 20 THEN 'HIGH'
                WHEN osm.counter_value > 10 THEN 'MEDIUM'
                ELSE 'LOW'
            END AS ActivityLevel,
            CASE 
                WHEN osm.counter_value > 20 THEN 'High lazy write activity indicates memory pressure'
                WHEN osm.counter_value > 10 THEN 'Moderate lazy write activity - monitor closely'
                ELSE 'Low lazy write activity is normal'
            END AS Description
        FROM sys.dm_os_performance_counters osm
        WHERE osm.object_name = 'SQLServer:Buffer Manager'
        AND osm.counter_name = 'Lazy writes/sec';
    END
    
    -- Plan cache analysis
    IF @IncludePlanCacheAnalysis = 1
    BEGIN
        PRINT 'Analyzing plan cache memory usage...';
        
        -- Plan cache size and distribution
        SELECT 
            'PLAN_CACHE_USAGE' AS AnalysisType,
            cp.cacheobjtype AS CacheObjectType,
            cp.objtype AS ObjectType,
            COUNT(*) AS ObjectCount,
            SUM(cp.size_in_bytes) / 1024 / 1024 AS SizeMB,
            AVG(cp.size_in_bytes) / 1024 AS AvgSizeKB,
            SUM(cp.usecounts) AS TotalUseCount,
            AVG(cp.usecounts) AS AvgUseCount
        FROM sys.dm_exec_cached_plans cp
        GROUP BY cp.cacheobjtype, cp.objtype
        ORDER BY SUM(cp.size_in_bytes) DESC;
        
        -- Adhoc queries consuming plan cache
        SELECT TOP 20
            'ADHOC_QUERY_ANALYSIS' AS AnalysisType,
            ST.text AS QueryText,
            cp.cacheobjtype AS CacheType,
            cp.objtype AS ObjectType,
            cp.size_in_bytes / 1024 AS SizeKB,
            cp.usecounts AS UseCount,
            CASE 
                WHEN cp.usecounts = 1 AND LEN(ST.text) > 500 THEN 'LONG_ADHOC_QUERY'
                WHEN cp.usecounts = 1 THEN 'ADHOC_QUERY'
                WHEN cp.usecounts < 5 THEN 'RARE_USE_QUERY'
                ELSE 'FREQUENTLY_USED_QUERY'
            END AS QueryCategory,
            CASE 
                WHEN cp.usecounts = 1 AND LEN(ST.text) > 500 THEN 'Consider using stored procedures or parameterized queries'
                WHEN cp.usecounts = 1 THEN 'Consider parameterizing this query'
                WHEN cp.usecounts < 5 THEN 'Monitor plan cache size impact'
                ELSE 'Query plan is frequently reused - performance likely good'
            END AS Recommendation
        FROM sys.dm_exec_cached_plans cp
        CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) ST
        WHERE cp.objtype = 'Adhoc'
        ORDER BY cp.size_in_bytes DESC;
    END
    
    -- Memory grants analysis
    IF @IncludeMemoryGrantsAnalysis = 1
    BEGIN
        PRINT 'Analyzing memory grants...';
        
        -- Queries waiting for memory grants
        SELECT TOP 20
            'MEMORY_GRANT_ANALYSIS' AS AnalysisType,
            session_id AS SessionId,
            request_id AS RequestId,
            database_id AS DatabaseId,
            user_id AS UserId,
            memory_requested_kb / 1024 AS MemoryRequestedMB,
            required_memory_kb / 1024 AS RequiredMemoryMB,
            max_used_memory_kb / 1024 AS MaxUsedMemoryMB,
            grant_time AS GrantTime,
            wait_time_ms AS WaitTimeMs,
            CASE 
                WHEN required_memory_kb > 1024 * 1024 THEN 'LARGE_MEMORY_GRANT'
                WHEN memory_requested_kb > 512 * 1024 THEN 'MEDIUM_MEMORY_GRANT'
                ELSE 'SMALL_MEMORY_GRANT'
            END AS GrantSize,
            CASE 
                WHEN required_memory_kb > 1024 * 1024 THEN 'Large memory grant may indicate hash operations or sorts'
                WHEN memory_requested_kb > 512 * 1024 THEN 'Medium memory grant - monitor for memory pressure'
                ELSE 'Small memory grant - typical for simple queries'
            END AS Analysis
        FROM sys.dm_exec_query_memory_grants
        WHERE session_id != @@SPID
        ORDER BY memory_requested_kb DESC;
        
        -- Memory grants over time
        SELECT 
            'MEMORY_GRANT_TRENDS' AS AnalysisType,
            COUNT(*) AS TotalGrants,
            AVG(memory_requested_kb / 1024) AS AvgGrantSizeMB,
            MAX(memory_requested_kb / 1024) AS MaxGrantSizeMB,
            SUM(CASE WHEN memory_requested_kb > 1024 * 1024 THEN 1 ELSE 0 END) AS LargeGrants,
            SUM(CASE WHEN required_memory_kb > 1024 * 1024 THEN 1 ELSE 0 END) AS RequiredLargeGrants
        FROM sys.dm_exec_query_memory_grants
        WHERE session_id != @@SPID;
    END
    
    -- SQL Server memory usage breakdown
    SELECT 
        'MEMORY_BREAKDOWN' AS AnalysisType,
        osm.instance_name AS MemoryComponent,
        osm.cntr_value / 1024 AS SizeMB,
        CASE 
            WHEN osm.instance_name = 'Total Server Memory (KB)' THEN 'Total memory allocated by SQL Server'
            WHEN osm.instance_name = 'Target Server Memory (KB)' THEN 'Target memory allocation'
            WHEN osm.instance_name = 'Buffer Pool Memory (KB)' THEN 'Memory used by buffer pool'
            WHEN osm.instance_name = 'Connection Memory (KB)' THEN 'Memory used for connections'
            WHEN osm.instance_name = 'Cursor memory usage (KB)' THEN 'Memory used by cursors'
            WHEN osm.instance_name = 'Database Cache Memory (KB)' THEN 'Memory used for database pages'
            WHEN osm.instance_name = 'Granted Workspace Memory (KB)' THEN 'Memory granted to queries'
            WHEN osm.instance_name = 'Lock Memory (KB)' THEN 'Memory used for locks'
            WHEN osm.instance_name = 'Optimizer Memory (KB)' THEN 'Memory used for query optimization'
            ELSE 'Other memory component'
        END AS ComponentDescription
    FROM sys.dm_os_performance_counters osm
    WHERE osm.object_name = 'SQLServer:Memory Manager'
    AND osm.cntr_value > 0
    ORDER BY osm.cntr_value DESC;
    
    -- Generate memory optimization recommendations
    IF @GenerateRecommendations = 1
    BEGIN
        PRINT 'Generating memory optimization recommendations...';
        
        WITH MemoryRecommendations AS (
            -- Page life expectancy recommendations
            SELECT 
                'PAGE_LIFE_EXPECTANCY' AS RecommendationType,
                'SYSTEM' AS DatabaseName,
                'Buffer Pool' AS AffectedObject,
                'Page Life Expectancy below recommended threshold' AS Issue,
                'Increase SQL Server memory allocation or optimize query patterns' AS Recommendation,
                'HIGH' AS Priority
            FROM sys.dm_os_performance_counters osm
            WHERE osm.object_name = 'SQLServer:Buffer Manager'
            AND osm.counter_name = 'Page life expectancy'
            AND osm.counter_value < 300
            
            UNION ALL
            
            -- Plan cache optimization
            SELECT 
                'PLAN_CACHE_OPTIMIZATION' AS RecommendationType,
                'SYSTEM' AS DatabaseName,
                'Plan Cache' AS AffectedObject,
                'Large number of single-use adhoc queries' AS Issue,
                'Implement stored procedures or query parameterization' AS Recommendation,
                'MEDIUM' AS Priority
            FROM (
                SELECT COUNT(*) AS AdhocCount
                FROM sys.dm_exec_cached_plans
                WHERE objtype = 'Adhoc' AND usecounts = 1
            ) adhoc_stats
            WHERE adhoc_stats.AdhocCount > 100
            
            UNION ALL
            
            -- Memory pressure detection
            SELECT 
                'MEMORY_PRESSURE' AS RecommendationType,
                'SYSTEM' AS DatabaseName,
                'Server Memory' AS AffectedObject,
                'High lazy write activity indicates memory pressure' AS Issue,
                'Increase SQL Server max server memory or optimize queries' AS Recommendation,
                'HIGH' AS Priority
            FROM sys.dm_os_performance_counters osm
            WHERE osm.object_name = 'SQLServer:Buffer Manager'
            AND osm.counter_name = 'Lazy writes/sec'
            AND osm.counter_value > 20
        )
        SELECT 
            RecommendationType,
            DatabaseName,
            AffectedObject,
            Issue,
            Recommendation,
            Priority,
            'Memory optimization recommendation' AS ActionRequired
        FROM MemoryRecommendations
        ORDER BY 
            CASE Priority WHEN 'HIGH' THEN 1 WHEN 'MEDIUM' THEN 2 WHEN 'LOW' THEN 3 ELSE 4 END;
    END
    
    PRINT 'Memory utilization analysis completed.';
    
    RETURN 0;
END
GO
```

## Blocking and Deadlock Analysis {#blocking-deadlocks}

### Blocking Investigation Methodology

Blocking occurs when one session holds a lock that another session is waiting for. Understanding blocking patterns and implementing resolution strategies is crucial for maintaining database performance and availability.

**Blocking Analysis Components:**
1. **Current Blocking Detection**: Identify active blocking relationships
2. **Blocking History Analysis**: Track blocking patterns over time
3. **Deadlock Detection**: Identify and analyze deadlocks
4. **Lock Type Analysis**: Understand different types of locks and their impact
5. **Resolution Strategies**: Implement appropriate blocking resolution approaches

```sql
-- Comprehensive blocking and deadlock analysis procedure
CREATE PROCEDURE sp_BlockingAndDeadlockAnalysis
(
    @IncludeCurrentBlocking bit = 1,
    @IncludeBlockingHistory bit = 1,
    @IncludeDeadlockAnalysis bit = 1,
    @IncludeLockAnalysis bit = 1,
    @GenerateRecommendations bit = 1
)
AS
BEGIN
    SET NOCOUNT ON;
    
    PRINT 'Starting comprehensive blocking and deadlock analysis...';
    
    -- Current blocking analysis
    IF @IncludeCurrentBlocking = 1
    BEGIN
        PRINT 'Analyzing current blocking relationships...';
        
        WITH BlockingHierarchy AS (
            -- Identify the root blockers (sessions that are blocking others but not blocked themselves)
            SELECT 
                session_id AS BlockingSession,
                'ROOT_BLOCKER' AS BlockerLevel,
                0 AS BlockingLevel
            FROM sys.dm_exec_requests
            WHERE blocking_session_id = 0
            AND session_id != @@SPID
            
            UNION ALL
            
            -- Identify blocked sessions
            SELECT 
                r.blocking_session_id AS BlockingSession,
                'BLOCKED_SESSION' AS BlockerLevel,
                1 AS BlockingLevel
            FROM sys.dm_exec_requests r
            WHERE r.blocking_session_id > 0
        )
        SELECT TOP 50
            'CURRENT_BLOCKING' AS AnalysisType,
            blocker.session_id AS BlockingSessionId,
            blocker.login_name AS BlockingUser,
            blocker.program_name AS BlockingProgram,
            blocker.host_name AS BlockingHost,
            blocker.cpu_time AS BlockingCpuTime,
            blocker.total_scheduled_time AS BlockingScheduledTime,
            blocker.total_elapsed_time AS BlockingElapsedTime,
            blocker.percent_complete AS BlockingPercentComplete,
            blocker.database_id AS BlockingDatabaseId,
            CASE WHEN blocker.database_id > 0 THEN DB_NAME(blocker.database_id) ELSE NULL END AS BlockingDatabaseName,
            blocker.wait_type AS BlockingWaitType,
            blocker.wait_time AS BlockingWaitTime,
            blocked.session_id AS BlockedSessionId,
            blocked.login_name AS BlockedUser,
            blocked.program_name AS BlockedProgram,
            blocked.host_name AS BlockedHost,
            blocked.cpu_time AS BlockedCpuTime,
            blocked.total_scheduled_time AS BlockedScheduledTime,
            blocked.total_elapsed_time AS BlockedElapsedTime,
            blocked.percent_complete AS BlockedPercentComplete,
            blocked.database_id AS BlockedDatabaseId,
            CASE WHEN blocked.database_id > 0 THEN DB_NAME(blocked.database_id) ELSE NULL END AS BlockedDatabaseName,
            blocked.wait_type AS BlockedWaitType,
            blocked.wait_time AS BlockedWaitTime,
            DATEDIFF(second, blocked.start_time, GETDATE()) AS BlockedDurationSeconds,
            SUBSTRING(blocking_sql.text, 1, 200) AS BlockingQueryPreview,
            SUBSTRING(blocked_sql.text, 1, 200) AS BlockedQueryPreview,
            CASE 
                WHEN blocked.wait_time > 300000 THEN 'CRITICAL_BLOCKING'
                WHEN blocked.wait_time > 60000 THEN 'HIGH_BLOCKING'
                WHEN blocked.wait_time > 10000 THEN 'MODERATE_BLOCKING'
                ELSE 'MINOR_BLOCKING'
            END AS BlockingSeverity,
            CASE 
                WHEN blocked.wait_time > 300000 THEN 'Session has been blocked for over 5 minutes - immediate intervention required'
                WHEN blocked.wait_time > 60000 THEN 'Session has been blocked for over 1 minute - intervention recommended'
                WHEN blocked.wait_time > 10000 THEN 'Session is blocked - monitor situation'
                ELSE 'Minor blocking detected'
            END AS BlockingAnalysis
        FROM sys.dm_exec_requests blocked
        INNER JOIN sys.dm_exec_requests blocker ON blocked.blocking_session_id = blocker.session_id
        LEFT JOIN sys.dm_exec_sql_text(blocker.sql_handle) blocking_sql ON blocker.sql_handle IS NOT NULL
        LEFT JOIN sys.dm_exec_sql_text(blocked.sql_handle) blocked_sql ON blocked.sql_handle IS NOT NULL
        WHERE blocked.session_id != @@SPID
        ORDER BY blocked.wait_time DESC;
        
        -- Blocking chain analysis
        SELECT 
            'BLOCKING_CHAIN_ANALYSIS' AS AnalysisType,
            'Blocking Chain Hierarchy' AS Description,
            COUNT(DISTINCT r.blocking_session_id) AS RootBlockers,
            COUNT(DISTINCT r.session_id) AS BlockedSessions,
            MAX(DATEDIFF(second, r.start_time, GETDATE())) AS LongestBlockingDuration,
            SUM(DATEDIFF(second, r.start_time, GETDATE())) AS TotalBlockingTime,
            CASE 
                WHEN MAX(DATEDIFF(second, r.start_time, GETDATE())) > 300 THEN 'CRITICAL'
                WHEN MAX(DATEDIFF(second, r.start_time, GETDATE())) > 60 THEN 'WARNING'
                ELSE 'NORMAL'
            END AS BlockingStatus
        FROM sys.dm_exec_requests r
        WHERE r.blocking_session_id > 0
        AND r.session_id != @@SPID;
    END
    
    -- Lock analysis
    IF @IncludeLockAnalysis = 1
    BEGIN
        PRINT 'Analyzing current lock patterns...';
        
        SELECT TOP 50
            'LOCK_ANALYSIS' AS AnalysisType,
            l.resource_type AS ResourceType,
            l.resource_description AS ResourceDescription,
            l.request_mode AS RequestMode,
            l.request_type AS RequestType,
            l.request_status AS RequestStatus,
            l.request_owner_type AS RequestOwnerType,
            s.session_id AS SessionId,
            s.login_name AS LoginName,
            s.program_name AS ProgramName,
            s.host_name AS HostName,
            s.status AS SessionStatus,
            s.cpu_time AS CpuTime,
            s.total_scheduled_time AS ScheduledTime,
            s.total_elapsed_time AS ElapsedTime,
            CASE 
                WHEN l.resource_type = 'DATABASE' THEN 'Database-level lock'
                WHEN l.resource_type = 'TABLE' THEN 'Table-level lock'
                WHEN l.resource_type = 'PAGE' THEN 'Page-level lock'
                WHEN l.resource_type = 'KEY' THEN 'Row-level lock'
                WHEN l.resource_type = 'RID' THEN 'Row identifier lock'
                WHEN l.resource_type = 'HOBT' THEN 'Heap or B-tree lock'
                ELSE 'Other resource type'
            END AS LockDescription,
            CASE 
                WHEN l.request_mode = 'S' THEN 'Shared - read operations'
                WHEN l.request_mode = 'X' THEN 'Exclusive - write operations'
                WHEN l.request_mode = 'U' THEN 'Update - intent to update'
                WHEN l.request_mode = 'IS' THEN 'Intent Shared - table-level intent'
                WHEN l.request_mode = 'IU' THEN 'Intent Update - table-level intent'
                WHEN l.request_mode = 'IX' THEN 'Intent Exclusive - table-level intent'
                WHEN l.request_mode = 'SIX' THEN 'Shared with Intent Exclusive'
                WHEN l.request_mode = 'SCH-S' THEN 'Schema stability'
                WHEN l.request_mode = 'SCH-M' THEN 'Schema modification'
                ELSE 'Other lock type'
            END AS LockModeDescription
        FROM sys.dm_tran_locks l
        INNER JOIN sys.dm_exec_sessions s ON l.request_session_id = s.session_id
        WHERE s.session_id != @@SPID
        AND s.is_user_process = 1
        ORDER BY l.request_time DESC;
        
        -- Lock escalation analysis
        SELECT 
            'LOCK_ESCALATION_ANALYSIS' AS AnalysisType,
            OBJECT_NAME(l.resource_associated_entity_id) AS TableName,
            SCHEMA_NAME(o.schema_id) AS SchemaName,
            l.resource_type AS ResourceType,
            COUNT(*) AS LockCount,
            MIN(l.request_time) AS FirstLockTime,
            MAX(l.request_time) AS LastLockTime,
            DATEDIFF(second, MIN(l.request_time), MAX(l.request_time)) AS LockDurationSeconds,
            CASE 
                WHEN COUNT(*) > 5000 THEN 'HIGH_ESCALATION_RISK'
                WHEN COUNT(*) > 1000 THEN 'MEDIUM_ESCALATION_RISK'
                ELSE 'NORMAL'
            END AS EscalationRisk,
            CASE 
                WHEN COUNT(*) > 5000 THEN 'High number of row locks may escalate to table locks'
                WHEN COUNT(*) > 1000 THEN 'Moderate number of row locks - monitor for escalation'
                ELSE 'Lock count appears normal'
            END AS EscalationAnalysis
        FROM sys.dm_tran_locks l
        INNER JOIN sys.objects o ON l.resource_associated_entity_id = o.object_id
        WHERE l.resource_type = 'KEY'
        AND o.type = 'U' -- User tables only
        GROUP BY OBJECT_NAME(l.resource_associated_entity_id), SCHEMA_NAME(o.schema_id), l.resource_type
        HAVING COUNT(*) > 100
        ORDER BY COUNT(*) DESC;
    end
    
    -- Deadlock analysis
    IF @IncludeDeadlockAnalysis = 1
    BEGIN
        PRINT 'Analyzing deadlock information...';
        
        -- Current deadlock information (requires Extended Events)
        SELECT 
            'DEADLOCK_ANALYSIS' AS AnalysisType,
            'Extended Events required for deadlock detection' AS Note,
            'Enable deadlock trace flag or Extended Events session' AS Recommendation;
        
        -- Historical deadlock information from system health session
        SELECT TOP 10
            'DEADLOCK_HISTORY' AS AnalysisType,
            'SYSTEM_HEALTH_SESSION' AS Source,
            event_data.value('(event[@name=''xml_deadlock_report'']/@timestamp)[1]', 'datetime') AS EventTime,
            event_data.value('(event[@name=''xml_deadlock_report'']/deadlock/victim-list/victim/@id)[1]', 'varchar(50)') AS VictimProcessId,
            event_data.value('(event[@name=''xml_deadlock_report'']/deadlock/process-list/process/@spid)[1]', 'int') AS VictimSpid,
            event_data.value('(event[@name=''xml_deadlock_report'']/deadlock/process-list/process/@loginname)[1]', 'varchar(100)') AS VictimLogin,
            event_data.value('(event[@name=''xml_deadlock_report'']/deadlock/process-list/process/@hostname)[1]', 'varchar(100)') AS VictimHost,
            'Deadlock detected - see XML data for details' AS Description
        FROM sys.fn_xe_file_target_read_file('system_health*.xel', NULL, NULL, NULL) AS target_file
        CROSS APPLY (SELECT CAST(target_file.event_data AS xml)) AS event_data
        WHERE target_file.event_data LIKE '%xml_deadlock_report%'
        ORDER BY event_data.value('(event[@name=''xml_deadlock_report'']/@timestamp)[1]', 'datetime') DESC;
        
        -- Deadlock victim analysis
        SELECT 
            'DEADLOCK_VICTIM_ANALYSIS' AS AnalysisType,
            COUNT(*) AS DeadlockCount,
            COUNT(DISTINCT event_data.value('(event[@name=''xml_deadlock_report'']/deadlock/process-list/process/@spid)[1]', 'int')) AS UniqueVictimSpids,
            COUNT(DISTINCT event_data.value('(event[@name=''xml_deadlock_report'']/deadlock/process-list/process/@hostname)[1]', 'varchar(100)')) AS UniqueVictimHosts,
            AVG(CAST(event_data.value('(event[@name=''xml_deadlock_report'']/deadlock/process-list/process/@spid)[1]', 'int') AS float)) AS AvgVictimSpid,
            'Deadlock patterns detected in system health session' AS Analysis
        FROM sys.fn_xe_file_target_read_file('system_health*.xel', NULL, NULL, NULL) AS target_file
        CROSS APPLY (SELECT CAST(target_file.event_data AS xml)) AS event_data
        WHERE target_file.event_data LIKE '%xml_deadlock_report%'
        GROUP BY CAST(0 AS bit); -- Dummy group by to get overall statistics
    end
    
    -- Generate blocking and deadlock recommendations
    IF @GenerateRecommendations = 1
    BEGIN
        PRINT 'Generating blocking and deadlock recommendations...';
        
        WITH BlockingRecommendations AS (
            -- Current blocking detection
            SELECT 
                'CURRENT_BLOCKING' AS RecommendationType,
                CASE WHEN blocked.database_id > 0 THEN DB_NAME(blocked.database_id) ELSE 'MULTIPLE_DATABASES' END AS DatabaseName,
                'Session ' + CAST(blocker.session_id AS varchar(10)) + ' blocking session ' + CAST(blocked.session_id AS varchar(10)) AS AffectedObject,
                'Active blocking detected with duration ' + CAST(DATEDIFF(second, blocked.start_time, GETDATE()) AS varchar(10)) + ' seconds' AS Issue,
                'Identify blocking session and consider KILL command if necessary' AS Recommendation,
                CASE 
                    WHEN DATEDIFF(second, blocked.start_time, GETDATE()) > 300 THEN 'HIGH'
                    WHEN DATEDIFF(second, blocked.start_time, GETDATE()) > 60 THEN 'MEDIUM'
                    ELSE 'LOW'
                END AS Priority
            FROM sys.dm_exec_requests blocked
            INNER JOIN sys.dm_exec_requests blocker ON blocked.blocking_session_id = blocker.session_id
            WHERE blocked.session_id != @@SPID
            AND DATEDIFF(second, blocked.start_time, GETDATE()) > 30
            
            UNION ALL
            
            -- Lock escalation recommendations
            SELECT 
                'LOCK_ESCALATION' AS RecommendationType,
                DB_NAME(o.object_id) AS DatabaseName,
                SCHEMA_NAME(o.schema_id) + '.' + OBJECT_NAME(l.resource_associated_entity_id) AS AffectedObject,
                'High number of row locks (' + CAST(COUNT(*) AS varchar(10)) + ') on table may escalate to table lock' AS Issue,
                'Consider adding appropriate indexes or implementing row versioning' AS Recommendation,
                'MEDIUM' AS Priority
            FROM sys.dm_tran_locks l
            INNER JOIN sys.objects o ON l.resource_associated_entity_id = o.object_id
            WHERE l.resource_type = 'KEY'
            AND o.type = 'U'
            GROUP BY DB_NAME(o.object_id), SCHEMA_NAME(o.schema_id), OBJECT_NAME(l.resource_associated_entity_id)
            HAVING COUNT(*) > 5000
            
            UNION ALL
            
            -- Query optimization recommendations
            SELECT 
                'QUERY_OPTIMIZATION' AS RecommendationType,
                'SYSTEM' AS DatabaseName,
                'Blocking and deadlocks' AS AffectedObject,
                'High blocking activity indicates potential query optimization opportunities' AS Issue,
                'Review query execution plans, add appropriate indexes, and implement proper transaction management' AS Recommendation,
                'HIGH' AS Priority
        )
        SELECT TOP 20
            RecommendationType,
            DatabaseName,
            AffectedObject,
            Issue,
            Recommendation,
            Priority,
            'Blocking/deadlock optimization required' AS ActionRequired
        FROM BlockingRecommendations
        ORDER BY 
            CASE Priority WHEN 'HIGH' THEN 1 WHEN 'MEDIUM' THEN 2 WHEN 'LOW' THEN 3 ELSE 4 END;
    end
    
    -- Return analysis summary
    SELECT 
        'ANALYSIS_SUMMARY' AS SummaryType,
        @IncludeCurrentBlocking AS CurrentBlockingAnalyzed,
        @IncludeBlockingHistory AS BlockingHistoryAnalyzed,
        @IncludeDeadlockAnalysis AS DeadlockAnalysisIncluded,
        @IncludeLockAnalysis AS LockAnalysisIncluded,
        (SELECT COUNT(*) FROM sys.dm_exec_requests WHERE blocking_session_id > 0) AS CurrentBlockedSessions,
        (SELECT COUNT(*) FROM sys.dm_tran_locks WHERE request_session_id != @@SPID) AS ActiveLocks,
        GETDATE() AS AnalysisCompleted,
        'Blocking and deadlock analysis completed successfully' AS Status;
    
    PRINT 'Blocking and deadlock analysis completed.';
    
    RETURN 0;
END
GO
```

## Consistency Issues {#consistency-issues}

### Database Consistency Troubleshooting

Database consistency issues can manifest as corruption errors, integrity check failures, or unexpected behavior. Systematic consistency analysis helps identify and resolve these critical issues.

```sql
-- Database consistency troubleshooting procedure
CREATE PROCEDURE sp_DatabaseConsistencyTroubleshooting
(
    @DatabaseName sysname = NULL,
    @PerformIntegrityCheck bit = 0,
    @CheckSystemTables bit = 1,
    @AnalyzeCorruptionPatterns bit = 1,
    @IncludePageVerification bit = 1,
    @GenerateResolutionPlan bit = 1
)
AS
BEGIN
    SET NOCOUNT ON;
    
    PRINT 'Starting database consistency troubleshooting...';
    
    -- Database integrity overview
    IF @DatabaseName IS NOT NULL
    BEGIN
        SELECT 
            'DATABASE_INTEGRITY' AS AnalysisType,
            d.name AS DatabaseName,
            d.state_desc AS DatabaseState,
            d.user_access_desc AS UserAccess,
            d.compatibility_level AS CompatibilityLevel,
            d.recovery_model_desc AS RecoveryModel,
            d.is_auto_create_stats_on AS AutoCreateStats,
            d.is_auto_update_stats_on AS AutoUpdateStats,
            d.is_auto_close AS AutoClose,
            d.is_auto_shrink AS AutoShrink,
            CASE 
                WHEN d.state = 0 THEN 'Database is online and accessible'
                WHEN d.state = 1 THEN 'Database is in restore state'
                WHEN d.state = 2 THEN 'Database is recovering'
                WHEN d.state = 3 THEN 'Database recovery pending'
                WHEN d.state = 4 THEN 'Database is suspect'
                ELSE 'Unknown database state'
            END AS StateDescription,
            CASE 
                WHEN d.state = 4 THEN 'CRITICAL: Database is suspect - immediate attention required'
                WHEN d.state > 0 THEN 'WARNING: Database not in normal state'
                ELSE 'GOOD: Database appears healthy'
            END AS HealthStatus
        FROM sys.databases d
        WHERE d.name = @DatabaseName;
    END
    
    -- System table consistency analysis
    IF @CheckSystemTables = 1
    BEGIN
        PRINT 'Checking system table consistency...';
        
        -- System table integrity
        SELECT 
            'SYSTEM_TABLE_INTEGRITY' AS AnalysisType,
            OBJECT_SCHEMA_NAME(object_id) + '.' + OBJECT_NAME(object_id) AS TableName,
            type_desc AS TableType,
            is_replicated AS IsReplicated,
            is_merge_published AS IsMergePublished,
            is_published AS IsPublished,
            is_subscribed AS IsSubscribed,
            is_tracked_by_cdc AS IsTrackedByCDC,
            CASE 
                WHEN is_tracked_by_cdc = 1 AND NOT EXISTS (SELECT 1 FROM cdc.change_tables WHERE object_id = system_table_integrity.object_id) THEN 'CDC_TRACKING_MISMATCH'
                WHEN is_replicated = 1 AND NOT EXISTS (SELECT 1 FROM distribution.dbo.MSreplication_subscriptions WHERE article_id = system_table_integrity.object_id) THEN 'REPLICATION_MISMATCH'
                ELSE 'SYSTEM_TABLE_NORMAL'
            END AS ConsistencyStatus,
            CASE 
                WHEN is_tracked_by_cdc = 1 AND NOT EXISTS (SELECT 1 FROM cdc.change_tables WHERE object_id = system_table_integrity.object_id) THEN 'CDC tracking enabled but change table not found'
                WHEN is_replicated = 1 AND NOT EXISTS (SELECT 1 FROM distribution.dbo.MSreplication_subscriptions WHERE article_id = system_table_integrity.object_id) THEN 'Replication flag set but subscription not found'
                ELSE 'System table consistency appears normal'
            END AS ConsistencyIssue
        FROM sys.tables system_table_integrity
        WHERE (@DatabaseName IS NULL OR DB_NAME(DB_ID()) = @DatabaseName)
        AND (is_replicated = 1 OR is_merge_published = 1 OR is_published = 1 OR is_subscribed = 1 OR is_tracked_by_cdc = 1);
        
        -- Check for orphaned users
        SELECT 
            'ORPHANED_USERS' AS AnalysisType,
            d.name AS DatabaseName,
            dp.name AS UserName,
            dp.type_desc AS UserType,
            dp.authentication_type_desc AS AuthenticationType,
            'User exists in database but not in server' AS Issue,
            'Create server login for orphaned user or drop database user' AS Recommendation
        FROM sys.databases d
        INNER JOIN sys.database_principals dp ON d.database_id = dp.database_id
        LEFT JOIN sys.server_principals sp ON dp.name = sp.name
        WHERE (@DatabaseName IS NULL OR d.name = @DatabaseName)
        AND dp.type_desc IN ('SQL_USER', 'WINDOWS_USER', 'WINDOWS_GROUP')
        AND dp.authentication_type_desc = 'INSTANCE'
        AND sp.name IS NULL
        AND dp.name != 'dbo';
    END
    
    -- Page verification analysis
    IF @IncludePageVerification = 1
    BEGIN
        PRINT 'Performing page verification analysis...';
        
        -- Note: This would require extensive checks using DBCC PAGE and other low-level operations
        SELECT 
            'PAGE_VERIFICATION' AS AnalysisType,
            'Page-level verification requires specialized tools' AS Note,
            'Use DBCC CHECKDB, DBCC PAGE, and Extended Events for detailed page analysis' AS Recommendation;
    END
    
    -- Transaction log consistency
    SELECT 
        'TRANSACTION_LOG_CONSISTENCY' AS AnalysisType,
        d.name AS DatabaseName,
        df.type_desc AS FileType,
        df.name AS FileName,
        df.physical_name AS FilePath,
        df.size * 8 / 1024 AS FileSizeMB,
        CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 8 / 1024 AS UsedSpaceMB,
        (df.size - CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint)) * 8 / 1024 AS FreeSpaceMB,
        CAST((CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 100.0 / df.size) AS decimal(5,2)) AS PercentUsed,
        CASE 
            WHEN df.type = 2 THEN -- Log file
                CASE 
                    WHEN CAST((CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 100.0 / df.size) AS decimal(5,2)) > 90 THEN 'CRITICAL'
                    WHEN CAST((CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 100.0 / df.size) AS decimal(5,2)) > 70 THEN 'WARNING'
                    ELSE 'NORMAL'
                END
            ELSE 'NOT_APPLICABLE'
        END AS LogHealthStatus,
        CASE 
            WHEN df.type = 2 AND CAST((CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 100.0 / df.size) AS decimal(5,2)) > 90 THEN 'Transaction log is nearly full - immediate action required'
            WHEN df.type = 2 AND CAST((CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 100.0 / df.size) AS decimal(5,2)) > 70 THEN 'Transaction log usage is high - monitor closely'
            WHEN df.type = 2 THEN 'Transaction log space usage appears normal'
            ELSE 'N/A for data files'
        END AS LogAnalysis
    FROM sys.databases d
    CROSS APPLY sys.database_files df
    WHERE (@DatabaseName IS NULL OR d.name = @DatabaseName)
    AND d.database_id > 4
    ORDER BY d.name, df.type_desc;
    
    -- Consistency check results
    IF @PerformIntegrityCheck = 1
    BEGIN
        PRINT 'Running database integrity check...';
        
        -- Note: In production, you would use DBCC CHECKDB or sp_executesql to run integrity checks
        SELECT 
            'INTEGRITY_CHECK' AS AnalysisType,
            @DatabaseName AS DatabaseName,
            'Integrity check initiated' AS Status,
            'Use DBCC CHECKDB for comprehensive integrity verification' AS NextStep;
    END
    
    -- Index consistency analysis
    SELECT TOP 20
        'INDEX_CONSISTENCY' AS AnalysisType,
        DB_NAME(ips.database_id) AS DatabaseName,
        OBJECT_NAME(ips.object_id) AS TableName,
        i.name AS IndexName,
        i.type_desc AS IndexType,
        ips.avg_fragmentation_in_percent AS FragmentationPercent,
        ips.page_count AS PageCount,
        ips.record_count AS RecordCount,
        ips.index_level AS IndexLevel,
        CASE 
            WHEN ips.avg_fragmentation_in_percent > 40 THEN 'CRITICAL'
            WHEN ips.avg_fragmentation_in_percent > 20 THEN 'WARNING'
            WHEN ips.avg_fragmentation_in_percent > 10 THEN 'MONITOR'
            ELSE 'NORMAL'
        END AS FragmentationStatus,
        CASE 
            WHEN ips.avg_fragmentation_in_percent > 40 THEN 'Severe fragmentation - immediate rebuild required'
            WHEN ips.avg_fragmentation_in_percent > 20 THEN 'High fragmentation - rebuild recommended'
            WHEN ips.avg_fragmentation_in_percent > 10 THEN 'Moderate fragmentation - monitor and reorganize if needed'
            ELSE 'Fragmentation within acceptable range'
        END AS FragmentationAnalysis
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
    INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE (@DatabaseName IS NULL OR DB_NAME(ips.database_id) = @DatabaseName)
    AND i.type > 0
    AND ips.index_level = 0
    ORDER BY ips.avg_fragmentation_in_percent DESC;
    
    -- Corruption pattern analysis
    IF @AnalyzeCorruptionPatterns = 1
    BEGIN
        PRINT 'Analyzing potential corruption patterns...';
        
        -- Check for common corruption indicators
        SELECT 
            'CORRUPTION_ANALYSIS' AS AnalysisType,
            'Corruption pattern analysis requires Extended Events and error log monitoring' AS Note,
            'Enable corruption detection through Extended Events and monitor error logs' AS Recommendation;
        
        -- Check for consistency issues in backup history
        SELECT TOP 10
            'BACKUP_CONSISTENCY' AS AnalysisType,
            d.name AS DatabaseName,
            bs.backup_start_date AS BackupDate,
            bs.type_desc AS BackupType,
            bs.backup_size / 1024 / 1024 AS BackupSizeMB,
            bs.compressed_backup_size / 1024 / 1024 AS CompressedSizeMB,
            bs.is_damaged AS IsDamaged,
            bs.is_copy_only AS IsCopyOnly,
            CASE 
                WHEN bs.is_damaged = 1 THEN 'CORRUPTED_BACKUP'
                WHEN bs.backup_size / bs.compressed_backup_size > 2 THEN 'COMPRESSION_ANOMALY'
                WHEN bs.backup_size < 1024 * 1024 THEN 'UNUSUALLY_SMALL_BACKUP'
                ELSE 'BACKUP_NORMAL'
            END AS BackupStatus,
            CASE 
                WHEN bs.is_damaged = 1 THEN 'Backup marked as damaged - investigate integrity'
                WHEN bs.backup_size / bs.compressed_backup_size > 2 THEN 'Unusual compression ratio - verify backup integrity'
                WHEN bs.backup_size < 1024 * 1024 THEN 'Unusually small backup - check for completeness'
                ELSE 'Backup consistency appears normal'
            END AS BackupAnalysis
        FROM msdb.dbo.backupset bs
        INNER JOIN sys.databases d ON bs.database_name = d.name
        WHERE (@DatabaseName IS NULL OR d.name = @DatabaseName)
        AND bs.backup_start_date >= DATEADD(day, -7, GETDATE())
        ORDER BY bs.backup_start_date DESC;
    end
    
    -- Generate resolution recommendations
    IF @GenerateResolutionPlan = 1
    BEGIN
        PRINT 'Generating consistency resolution recommendations...';
        
        WITH ConsistencyRecommendations AS (
            -- Database state issues
            SELECT 
                'DATABASE_STATE' AS RecommendationType,
                d.name AS DatabaseName,
                'Database ' + d.name AS AffectedObject,
                'Database is in ' + LOWER(d.state_desc) + ' state' AS Issue,
                CASE 
                    WHEN d.state = 1 THEN 'Complete database restore process'
                    WHEN d.state = 2 THEN 'Allow recovery to complete or restore from backup if recovery fails'
                    WHEN d.state = 3 THEN 'Manual recovery intervention required'
                    WHEN d.state = 4 THEN 'CRITICAL: Database marked as suspect - restore from backup immediately'
                    ELSE 'Investigate database state issue'
                END AS Recommendation,
                CASE 
                    WHEN d.state = 4 THEN 'CRITICAL'
                    WHEN d.state > 0 THEN 'HIGH'
                    ELSE 'LOW'
                END AS Priority
            FROM sys.databases d
            WHERE (@DatabaseName IS NULL OR d.name = @DatabaseName)
            AND d.state > 0
            
            UNION ALL
            
            -- Transaction log issues
            SELECT 
                'TRANSACTION_LOG' AS RecommendationType,
                d.name AS DatabaseName,
                'Transaction Log - ' + df.name AS AffectedObject,
                'Transaction log is ' + CAST(CAST((CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 100.0 / df.size) AS decimal(5,2)) AS varchar(10)) + '% full' AS Issue,
                'Back up transaction log and/or increase log file size' AS Recommendation,
                CASE 
                    WHEN CAST((CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 100.0 / df.size) AS decimal(5,2)) > 90 THEN 'CRITICAL'
                    WHEN CAST((CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 100.0 / df.size) AS decimal(5,2)) > 70 THEN 'HIGH'
                    ELSE 'MEDIUM'
                END AS Priority
            FROM sys.databases d
            CROSS APPLY sys.database_files df
            WHERE (@DatabaseName IS NULL OR d.name = @DatabaseName)
            AND df.type = 2 -- Log file
            AND CAST((CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 100.0 / df.size) AS decimal(5,2)) > 70
            
            UNION ALL
            
            -- Index fragmentation issues
            SELECT 
                'INDEX_FRAGMENTATION' AS RecommendationType,
                DB_NAME(ips.database_id) AS DatabaseName,
                OBJECT_NAME(ips.object_id) + '.' + i.name AS AffectedObject,
                'Index fragmentation at ' + CAST(ips.avg_fragmentation_in_percent AS varchar(10)) + '%' AS Issue,
                CASE 
                    WHEN ips.avg_fragmentation_in_percent > 40 THEN 'Rebuild index with ONLINE = ON option'
                    WHEN ips.avg_fragmentation_in_percent > 10 THEN 'Reorganize index or schedule rebuild'
                    ELSE 'Monitor fragmentation levels'
                END AS Recommendation,
                CASE 
                    WHEN ips.avg_fragmentation_in_percent > 40 THEN 'HIGH'
                    WHEN ips.avg_fragmentation_in_percent > 20 THEN 'MEDIUM'
                    ELSE 'LOW'
                END AS Priority
            FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
            INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
            WHERE (@DatabaseName IS NULL OR DB_NAME(ips.database_id) = @DatabaseName)
            AND i.type > 0
            AND ips.index_level = 0
            AND ips.avg_fragmentation_in_percent > 10
        )
        SELECT 
            RecommendationType,
            DatabaseName,
            AffectedObject,
            Issue,
            Recommendation,
            Priority,
            'Database consistency resolution required' AS ActionRequired
        FROM ConsistencyRecommendations
        ORDER BY 
            CASE Priority WHEN 'CRITICAL' THEN 1 WHEN 'HIGH' THEN 2 WHEN 'MEDIUM' THEN 3 WHEN 'LOW' THEN 4 ELSE 5 END;
    end
    
    -- Return troubleshooting summary
    SELECT 
        'CONSISTENCY_SUMMARY' AS SummaryType,
        @DatabaseName AS TargetDatabase,
        @CheckSystemTables AS SystemTablesChecked,
        @AnalyzeCorruptionPatterns AS CorruptionPatternsAnalyzed,
        @IncludePageVerification AS PageVerificationIncluded,
        @PerformIntegrityCheck AS IntegrityCheckPerformed,
        CASE WHEN EXISTS (SELECT 1 FROM sys.databases WHERE name = @DatabaseName AND state > 0) 
             THEN 'ISSUES_DETECTED' 
             ELSE 'CONSISTENCY_NORMAL' 
        END AS OverallStatus,
        GETDATE() AS AnalysisCompleted;
    
    PRINT 'Database consistency troubleshooting completed.';
    
    RETURN 0;
END
GO
```

## Advanced Diagnostic Techniques {#advanced-techniques}

### Extended Events for Advanced Diagnostics

Extended Events provide comprehensive diagnostic capabilities for monitoring SQL Server behavior, performance, and troubleshooting complex issues.

```sql
-- Extended Events diagnostic management procedure
CREATE PROCED sp_ExtendedEventsDiagnosticManagement
(
    @SessionName nvarchar(128) = 'Advanced_Diagnostics',
    @Operation nvarchar(20) = 'CREATE', -- CREATE, START, STOP, DROP, ANALYZE
    @EventCategories nvarchar(500) = 'ALL', -- ALL, PERFORMANCE, ERRORS, BLOCKING, QUERY_EXECUTION
    @TargetType nvarchar(20) = 'RING_BUFFER', -- RING_BUFFER, EVENT_FILE, ASYNC_FILE_TARGET
    @MaxMemoryMB int = 512,
    @RetentionHours int = 24
)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL nvarchar(max);
    DECLARE @EventList nvarchar(max);
    
    -- Create event session based on categories
    IF @Operation = 'CREATE'
    BEGIN
        PRINT 'Creating Extended Events session: ' + @SessionName;
        
        -- Build event list based on categories
        IF @EventCategories = 'ALL' OR @EventCategories LIKE '%PERFORMANCE%'
        BEGIN
            SET @EventList = @EventList + '
        ADD EVENT sqlserver.sql_statement_completed (
            ACTION (
                sqlserver.database_id,
                sqlserver.database_name,
                sqlserver.session_id,
                sqlserver.username,
                sqlserver.client_app_name,
                sqlserver.client_hostname,
                sqlserver.sql_text,
                sqlserver.query_hash,
                sqlserver.query_plan_hash,
                sqlserver.collect_system_time
            )
            WHERE (duration > (1000000) OR cpu_time > (100000) OR logical_reads > (10000))
        ),
        ADD EVENT sqlserver.rpc_completed (
            ACTION (
                sqlserver.database_id,
                sqlserver.database_name,
                sqlserver.session_id,
                sqlserver.username,
                sqlserver.client_app_name,
                sqlserver.client_hostname,
                sqlserver.sql_text,
                sqlserver.collect_system_time
            )
            WHERE (duration > (1000000) OR cpu_time > (100000))
        ),';
        END
        
        IF @EventCategories = 'ALL' OR @EventCategories LIKE '%ERRORS%'
        BEGIN
            SET @EventList = @EventList + '
        ADD EVENT sqlserver.error_reported (
            ACTION (
                sqlserver.database_id,
                sqlserver.database_name,
                sqlserver.session_id,
                sqlserver.username,
                sqlserver.client_app_name,
                sqlserver.client_hostname,
                sqlserver.sql_text,
                sqlserver.query_hash,
                sqlserver.collect_system_time
            )
            WHERE ([severity] >= (15))
        ),
        ADD EVENT sqlserver.sql_assembly_load (
            ACTION (
                sqlserver.database_id,
                sqlserver.database_name,
                sqlserver.session_id,
                sqlserver.username,
                sqlserver.client_app_name
            )
        ),';
        END
        
        IF @EventCategories = 'ALL' OR @EventCategories LIKE '%BLOCKING%'
        BEGIN
            SET @EventList = @EventList + '
        ADD EVENT sqlserver.lock_deadlock (
            ACTION (
                sqlserver.database_id,
                sqlserver.database_name,
                sqlserver.session_id,
                sqlserver.username,
                sqlserver.client_app_name,
                sqlserver.client_hostname,
                sqlserver.sql_text,
                sqlserver.collect_system_time
            )
        ),
        ADD EVENT sqlserver.lock_escalation (
            ACTION (
                sqlserver.database_id,
                sqlserver.database_name,
                sqlserver.session_id,
                sqlserver.username,
                sqlserver.client_app_name
            )
        ),';
        END
        
        IF @EventCategories = 'ALL' OR @EventCategories LIKE '%QUERY_EXECUTION%'
        BEGIN
            SET @EventList = @EventList + '
        ADD EVENT sqlserver.sql_batch_completed (
            ACTION (
                sqlserver.database_id,
                sqlserver.database_name,
                sqlserver.session_id,
                sqlserver.username,
                sqlserver.client_app_name,
                sqlserver.client_hostname,
                sqlserver.sql_text,
                sqlserver.collect_system_time
            )
            WHERE (duration > (500000) OR cpu_time > (50000))
        ),';
        END
        
        -- Remove trailing comma
        SET @EventList = LEFT(@EventList, LEN(@EventList) - 1);
        
        -- Build complete session creation script
        SET @SQL = 'CREATE EVENT SESSION ' + QUOTENAME(@SessionName) + ' ON SERVER ' + @EventList + '
        ADD TARGET ' + CASE 
            WHEN @TargetType = 'RING_BUFFER' THEN 'package0.ring_buffer'
            WHEN @TargetType = 'EVENT_FILE' THEN 'package0.event_file'
            ELSE 'package0.ring_buffer'
        END + ' (
            SET max_memory = ' + CAST(@MaxMemoryMB * 1024 AS nvarchar(10)) + ' -- size in KB
        )
        WITH (
            MAX_MEMORY = ' + CAST(@MaxMemoryMB AS nvarchar(10)) + ' MB,
            EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS,
            MAX_DISPATCH_LATENCY = 30 SECONDS,
            MAX_EVENT_SIZE = 0 KB,
            MEMORY_PARTITION_MODE = NONE,
            TRACK_CAUSALITY = ON,
            STARTUP_STATE = OFF
        );';
        
        -- Execute session creation
        BEGIN TRY
            EXEC sp_executesql @SQL;
            PRINT 'Extended Events session created successfully: ' + @SessionName;
        END TRY
        BEGIN CATCH
            PRINT 'Error creating Extended Events session: ' + ERROR_MESSAGE();
        END CATCH
    END
    
    -- Start session
    IF @Operation = 'START'
    BEGIN
        PRINT 'Starting Extended Events session: ' + @SessionName;
        BEGIN TRY
            EXEC('ALTER EVENT SESSION ' + QUOTENAME(@SessionName) + ' ON SERVER STATE = START;');
            PRINT 'Extended Events session started successfully: ' + @SessionName;
        END TRY
        BEGIN CATCH
            PRINT 'Error starting Extended Events session: ' + ERROR_MESSAGE();
        END CATCH
    END
    
    -- Stop session
    IF @Operation = 'STOP'
    BEGIN
        PRINT 'Stopping Extended Events session: ' + @SessionName;
        BEGIN TRY
            EXEC('ALTER EVENT SESSION ' + QUOTENAME(@SessionName) + ' ON SERVER STATE = STOP;');
            PRINT 'Extended Events session stopped successfully: ' + @SessionName;
        END TRY
        BEGIN CATCH
            PRINT 'Error stopping Extended Events session: ' + ERROR_MESSAGE();
        END CATCH
    END
    
    -- Drop session
    IF @Operation = 'DROP'
    BEGIN
        PRINT 'Dropping Extended Events session: ' + @SessionName;
        BEGIN TRY
            EXEC('DROP EVENT SESSION ' + QUOTENAME(@SessionName) + ' ON SERVER;');
            PRINT 'Extended Events session dropped successfully: ' + @SessionName;
        END TRY
        BEGIN CATCH
            PRINT 'Error dropping Extended Events session: ' + ERROR_MESSAGE();
        END CATCH
    END
    
    -- Analyze session data
    IF @Operation = 'ANALYZE'
    BEGIN
        PRINT 'Analyzing Extended Events session data: ' + @SessionName;
        
        -- Query session data based on target type
        IF @TargetType = 'RING_BUFFER'
        BEGIN
            SELECT 
                'RING_BUFFER_DATA' AS AnalysisType,
                event_data.value('(@name)[1]', 'varchar(100)') AS EventName,
                event_data.value('(@timestamp)[1]', 'datetime') AS EventTime,
                event_data.value('(action[@name=''database_id'']/value)[1]', 'int') AS DatabaseId,
                event_data.value('(action[@name=''database_name'']/value)[1]', 'varchar(128)') AS DatabaseName,
                event_data.value('(action[@name=''session_id'']/value)[1]', 'int') AS SessionId,
                event_data.value('(action[@name=''username'']/value)[1]', 'varchar(128)') AS Username,
                event_data.value('(action[@name=''client_app_name'']/value)[1]', 'varchar(128)') AS ClientApp,
                event_data.value('(action[@name=''client_hostname'']/value)[1]', 'varchar(128)') AS ClientHost,
                event_data.value('(action[@name=''sql_text'']/value)[1]', 'nvarchar(max)') AS SqlText,
                CASE 
                    WHEN event_data.value('(@name)[1]', 'varchar(100)') = 'sql_statement_completed' THEN 'STATEMENT_EXECUTION'
                    WHEN event_data.value('(@name)[1]', 'varchar(100)') = 'error_reported' THEN 'ERROR_EVENT'
                    WHEN event_data.value('(@name)[1]', 'varchar(100)') = 'lock_deadlock' THEN 'DEADLOCK_EVENT'
                    WHEN event_data.value('(@name)[1]', 'varchar(100)') = 'sql_batch_completed' THEN 'BATCH_EXECUTION'
                    ELSE 'OTHER_EVENT'
                END AS EventCategory,
                CASE 
                    WHEN event_data.value('(@name)[1]', 'varchar(100)') = 'sql_statement_completed' AND 
                         event_data.value('(data[@name=''duration'']/value)[1]', 'bigint') > 1000000 THEN 'SLOW_STATEMENT'
                    WHEN event_data.value('(@name)[1]', 'varchar(100)') = 'error_reported' THEN 'ERROR_EVENT'
                    WHEN event_data.value('(@name)[1]', 'varchar(100)') = 'lock_deadlock' THEN 'DEADLOCK_DETECTED'
                    ELSE 'NORMAL_EVENT'
                END AS EventSeverity,
                event_data.value('(data[@name=''duration'']/value)[1]', 'bigint') / 1000.0 AS DurationMs,
                event_data.value('(data[@name=''cpu_time'']/value)[1]', 'bigint') AS CpuTimeMs,
                event_data.value('(data[@name=''logical_reads'']/value)[1]', 'bigint') AS LogicalReads
            FROM (
                SELECT CAST(target_data AS xml) AS event_data
                FROM sys.dm_xe_session_targets xst
                INNER JOIN sys.dm_xe_sessions xs ON xs.address = xst.event_session_address
                WHERE xs.name = @SessionName
                AND xst.target_name = 'ring_buffer'
            ) AS x
            CROSS APPLY x.event_data.nodes('//event') AS x2(event_data)
            ORDER BY event_data.value('(@timestamp)[1]', 'datetime') DESC;
        END
        
        -- Event file analysis
        IF @TargetType = 'EVENT_FILE'
        BEGIN
            SELECT 
                'EVENT_FILE_DATA' AS AnalysisType,
                file_name,
                file_size,
                file_modification_date,
                'Extended Events event file data requires file system access' AS Note
            FROM sys.fn_xe_file_target_read_file(@SessionName + '*.xel', NULL, NULL, NULL);
        END
    END
    
    -- Return session status
    SELECT 
        'SESSION_STATUS' AS StatusType,
        name AS SessionName,
        create_event_descriptor IS NOT NULL AS IsCreated,
        CASE 
            WHEN NULLIF(CAST(target_data AS xml), 1) IS NOT NULL THEN 'HAS_DATA'
            ELSE 'NO_DATA'
        END AS DataStatus,
        CASE 
            WHEN event_retention_mode_desc = 'ALLOW_SINGLE_EVENT_LOSS' THEN 'OPTIMAL'
            WHEN event_retention_mode_desc = 'ALLOW_MULTIPLE_EVENT_LOSS' THEN 'REDUCED'
            ELSE 'UNKNOWN'
        END AS RetentionMode,
        max_memory
    FROM sys.server_event_sessions ses
    LEFT JOIN sys.server_event_session_events ev ON ses.name = ev.name
    LEFT JOIN sys.server_event_session_targets tar ON ses.name = tar.name
    WHERE ses.name = @SessionName;
    
    RETURN 0;
END
GO
```

### Query Store for Performance Analysis

Query Store provides historical performance data for queries, enabling better performance analysis and troubleshooting:

```sql
-- Query Store analysis procedure
CREATE PROCEDURE sp_QueryStoreAnalysis
(
    @DatabaseName sysname = NULL,
    @AnalysisType nvarchar(20) = 'COMPREHENSIVE', -- COMPREHENSIVE, REGRESSION, FORCING, STATISTICS
    @MinExecutionCount int = 100,
    @PerformanceDropThreshold decimal(5,2) = 50.00, -- 50% performance drop
    @DaysToAnalyze int = 7,
    @ForceExecutionPlan bit = 0
)
AS
BEGIN
    SET NOCOUNT ON;
    
    PRINT 'Starting Query Store analysis...';
    
    -- Check if Query Store is enabled
    SELECT TOP 1
        'QUERY_STORE_STATUS' AS AnalysisType,
        DB_NAME(database_id) AS DatabaseName,
        current_query_cost AS CurrentQueryCost,
        total_execution_count AS TotalExecutionCount,
        avg_query_cost AS AvgQueryCost,
        last_execution_time AS LastExecutionTime,
        'Query Store analysis' AS Note
    FROM sys.query_store_runtime_stats
    WHERE (@DatabaseName IS NULL OR DB_NAME(database_id) = @DatabaseName);
    
    -- Query performance regression analysis
    IF @AnalysisType = 'COMPREHENSIVE' OR @AnalysisType = 'REGRESSION'
    BEGIN
        PRINT 'Analyzing query performance regressions...';
        
        WITH QueryPerformanceRegression AS (
            SELECT 
                qsqt.query_id,
                qsqt.object_id,
                qsqt.object_name,
                qsrsi.plan_id,
                qsrsi.avg_duration / 1000.0 AS AvgDurationSeconds,
                qsrsi.avg_cpu_time / 1000.0 AS AvgCpuTimeSeconds,
                qsrsi.avg_logical_io_reads AS AvgLogicalReads,
                qsrsi.avg_physical_io_reads AS AvgPhysicalReads,
                qsrsi.count_executions AS ExecutionCount,
                qsrsi.first_execution_time AS FirstExecutionTime,
                qsrsi.last_execution_time AS LastExecutionTime,
                -- Compare current vs historical performance
                LAG(qsrsi.avg_duration / 1000.0) OVER (PARTITION BY qsqt.query_id ORDER BY qsrsi.last_execution_time) AS PreviousAvgDurationSeconds,
                CASE 
                    WHEN LAG(qsrsi.avg_duration / 1000.0) OVER (PARTITION BY qsqt.query_id ORDER BY qsrsi.last_execution_time) > 0
                    THEN ((qsrsi.avg_duration / 1000.0) - LAG(qsrsi.avg_duration / 1000.0) OVER (PARTITION BY qsqt.query_id ORDER BY qsrsi.last_execution_time)) / 
                         LAG(qsrsi.avg_duration / 1000.0) OVER (PARTITION BY qsqt.query_id ORDER BY qsrsi.last_execution_time) * 100
                    ELSE 0
                END AS PerformanceChangePercent
            FROM sys.query_store_query_text qsqt
            INNER JOIN sys.query_store_query qsq ON qsqt.query_text_id = qsq.query_text_id
            INNER JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
            INNER JOIN sys.query_store_runtime_stats qsrsi ON qsp.plan_id = qsrsi.plan_id
            WHERE (@DatabaseName IS NULL OR DB_NAME(qsq.database_id) = @DatabaseName)
            AND qsrsi.count_executions >= @MinExecutionCount
            AND qsrsi.last_execution_time >= DATEADD(day, -@DaysToAnalyze, GETDATE())
        ),
        RegressionDetection AS (
            SELECT *,
                CASE 
                    WHEN ExecutionCount > 1000 AND PerformanceChangePercent > @PerformanceDropThreshold THEN 'HIGH_REGRESSION'
                    WHEN ExecutionCount > 100 AND PerformanceChangePercent > @PerformanceDropThreshold THEN 'MODERATE_REGRESSION'
                    WHEN PerformanceChangePercent > 25 THEN 'PERFORMANCE_VARIANCE'
                    ELSE 'STABLE_PERFORMANCE'
                END AS RegressionCategory
            FROM QueryPerformanceRegression
        )
        SELECT TOP 20
            'PERFORMANCE_REGRESSION' AS AnalysisType,
            query_id,
            ISNULL(object_name, 'Ad-hoc Query') AS ObjectName,
            plan_id,
            ExecutionCount,
            AvgDurationSeconds,
            PreviousAvgDurationSeconds,
            PerformanceChangePercent,
            RegressionCategory,
            CASE 
                WHEN ExecutionCount > 1000 AND PerformanceChangePercent > @PerformanceDropThreshold THEN 'Critical performance regression detected'
                WHEN ExecutionCount > 100 AND PerformanceChangePercent > @PerformanceDropThreshold THEN 'Moderate performance regression - investigate'
                WHEN PerformanceChangePercent > 25 THEN 'Performance variance detected'
                ELSE 'Performance appears stable'
            END AS AnalysisResult,
            CASE 
                WHEN ExecutionCount > 1000 AND PerformanceChangePercent > @PerformanceDropThreshold THEN 'Force good plan or optimize query immediately'
                WHEN ExecutionCount > 100 AND PerformanceChangePercent > @PerformanceDropThreshold THEN 'Review execution plan and optimize'
                WHEN PerformanceChangePercent > 25 THEN 'Monitor performance trends'
                ELSE 'No action required'
            END AS Recommendation
        FROM RegressionDetection
        WHERE RegressionCategory != 'STABLE_PERFORMANCE'
        ORDER BY ExecutionCount DESC, PerformanceChangePercent DESC;
    END
    
    -- Query plan forcing analysis
    IF @AnalysisType = 'COMPREHENSIVE' OR @AnalysisType = 'FORCING'
    BEGIN
        PRINT 'Analyzing query plan forcing opportunities...';
        
        -- Queries suitable for plan forcing
        SELECT TOP 20
            'PLAN_FORCING_ANALYSIS' AS AnalysisType,
            qsqt.query_text_id,
            qsq.query_id,
            qsp.plan_id,
            qsp.is_forced_plan AS IsForcedPlan,
            qsp.is_parallel_plan AS IsParallelPlan,
            qsp.is_trivial_plan AS IsTrivialPlan,
            qsp.is_compile_recommended AS IsCompileRecommended,
            qsp.engine_version AS EngineVersion,
            qsp.compatibility_level AS CompatibilityLevel,
            qsp.count_compiles AS CompileCount,
            qsrsi.avg_duration / 1000.0 AS AvgDurationSeconds,
            qsrsi.avg_logical_io_reads AS AvgLogicalReads,
            qsrsi.avg_physical_io_reads AS AvgPhysicalReads,
            qsrsi.count_executions AS ExecutionCount,
            LEFT(qsqt.query_sql_text, 100) + '...' AS QueryPreview,
            CASE 
                WHEN qsp.is_forced_plan = 1 THEN 'PLAN_ALREADY_FORCED'
                WHEN qsrsi.count_executions > 1000 AND qsp.engine_version = qsp.compatibility_level THEN 'GOOD_CANDIDATE_FOR_FORCING'
                WHEN qsrsi.count_executions > 100 AND qsp.engine_version = qsp.compatibility_level THEN 'MODERATE_CANDIDATE'
                ELSE 'POOR_CANDIDATE'
            END AS ForcingRecommendation,
            CASE 
                WHEN qsp.is_forced_plan = 1 THEN 'Query plan is already being forced'
                WHEN qsrsi.count_executions > 1000 AND qsp.engine_version = qsp.compatibility_level THEN 'Consider forcing this plan for stable performance'
                WHEN qsrsi.count_executions > 100 AND qsp.engine_version = qsp.compatibility_level THEN 'Monitor plan stability before forcing'
                ELSE 'Not suitable for plan forcing'
            END AS ForcingAnalysis
        FROM sys.query_store_query_text qsqt
        INNER JOIN sys.query_store_query qsq ON qsqt.query_text_id = qsq.query_text_id
        INNER JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
        INNER JOIN sys.query_store_runtime_stats qsrsi ON qsp.plan_id = qsrsi.plan_id
        WHERE (@DatabaseName IS NULL OR DB_NAME(qsq.database_id) = @DatabaseName)
        AND qsrsi.count_executions >= @MinExecutionCount
        AND qsrsi.last_execution_time >= DATEADD(day, -@DaysToAnalyze, GETDATE())
        ORDER BY qsrsi.avg_duration DESC, qsrsi.count_executions DESC;
    END
    
    -- Query statistics trends
    IF @AnalysisType = 'COMPREHENSIVE' OR @AnalysisType = 'STATISTICS'
    BEGIN
        PRINT 'Analyzing query statistics trends...';
        
        -- Query usage trends over time
        WITH QueryTrends AS (
            SELECT 
                qsqt.query_text_id,
                qsq.query_id,
                DATEADD(hour, DATEDIFF(hour, '19000101', qsrsi.last_execution_time), '19000101') AS ExecutionHour,
                COUNT(*) AS PlanCount,
                SUM(qsrsi.count_executions) AS TotalExecutions,
                AVG(qsrsi.avg_duration / 1000.0) AS AvgDurationSeconds,
                AVG(qsrsi.avg_logical_io_reads) AS AvgLogicalReads,
                MIN(qsrsi.first_execution_time) AS FirstExecution,
                MAX(qsrsi.last_execution_time) AS LastExecution
            FROM sys.query_store_query_text qsqt
            INNER JOIN sys.query_store_query qsq ON qsqt.query_text_id = qsq.query_text_id
            INNER JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
            INNER JOIN sys.query_store_runtime_stats qsrsi ON qsp.plan_id = qsrsi.plan_id
            WHERE (@DatabaseName IS NULL OR DB_NAME(qsq.database_id) = @DatabaseName)
            AND qsrsi.last_execution_time >= DATEADD(day, -@DaysToAnalyze, GETDATE())
            GROUP BY qsqt.query_text_id, qsq.query_id, 
                     DATEADD(hour, DATEDIFF(hour, '19000101', qsrsi.last_execution_time), '19000101')
        ),
        TrendAnalysis AS (
            SELECT *,
                LAG(TotalExecutions) OVER (PARTITION BY query_id ORDER BY ExecutionHour) AS PreviousExecutions,
                LAG(AvgDurationSeconds) OVER (PARTITION BY query_id ORDER BY ExecutionHour) AS PreviousDuration,
                CASE 
                    WHEN LAG(TotalExecutions) OVER (PARTITION BY query_id ORDER BY ExecutionHour) > 0
                    THEN (TotalExecutions - LAG(TotalExecutions) OVER (PARTITION BY query_id ORDER BY ExecutionHour)) / 
                         LAG(TotalExecutions) OVER (PARTITION BY query_id ORDER BY ExecutionHour) * 100
                    ELSE 0
                END AS ExecutionChangePercent,
                CASE 
                    WHEN LAG(AvgDurationSeconds) OVER (PARTITION BY query_id ORDER BY ExecutionHour) > 0
                    THEN (AvgDurationSeconds - LAG(AvgDurationSeconds) OVER (PARTITION BY query_id ORDER BY ExecutionHour)) / 
                         LAG(AvgDurationSeconds) OVER (PARTITION BY query_id ORDER BY ExecutionHour) * 100
                    ELSE 0
                END AS DurationChangePercent
            FROM QueryTrends
        )
        SELECT TOP 20
            'QUERY_STATISTICS_TRENDS' AS AnalysisType,
            query_id,
            ExecutionHour,
            TotalExecutions,
            PreviousExecutions,
            ExecutionChangePercent,
            AvgDurationSeconds,
            PreviousDuration,
            DurationChangePercent,
            CASE 
                WHEN ExecutionChangePercent > 100 THEN 'INCREASED_USAGE'
                WHEN ExecutionChangePercent < -50 THEN 'DECREASED_USAGE'
                WHEN DurationChangePercent > 50 THEN 'PERFORMANCE_DEGRADATION'
                WHEN DurationChangePercent < -50 THEN 'PERFORMANCE_IMPROVEMENT'
                ELSE 'STABLE_TRENDS'
            END AS TrendCategory,
            CASE 
                WHEN ExecutionChangePercent > 100 THEN 'Query usage has increased significantly - monitor for performance impact'
                WHEN ExecutionChangePercent < -50 THEN 'Query usage has decreased significantly'
                WHEN DurationChangePercent > 50 THEN 'Query performance has degraded - investigate cause'
                WHEN DurationChangePercent < -50 THEN 'Query performance has improved'
                ELSE 'Usage and performance trends appear stable'
            END AS TrendAnalysis
        FROM TrendAnalysis
        ORDER BY ABS(ExecutionChangePercent) DESC, ABS(DurationChangePercent) DESC;
    end
    
    -- High resource usage queries
    SELECT TOP 20
        'HIGH_RESOURCE_QUERIES' AS AnalysisType,
        qsqt.query_text_id,
        qsq.query_id,
        qsp.plan_id,
        SUM(qsrsi.count_executions) AS TotalExecutions,
        AVG(qsrsi.avg_duration / 1000.0) AS AvgDurationSeconds,
        MAX(qsrsi.avg_duration / 1000.0) AS MaxDurationSeconds,
        AVG(qsrsi.avg_logical_io_reads) AS AvgLogicalReads,
        MAX(qsrsi.avg_logical_io_reads) AS MaxLogicalReads,
        AVG(qsrsi.avg_physical_io_reads) AS AvgPhysicalReads,
        MAX(qsrsi.avg_physical_io_reads) AS MaxPhysicalReads,
        AVG(qsrsi.avg_logical_io_writes) AS AvgLogicalWrites,
        SUM(qsrsi.count_executions * qsrsi.avg_duration) / SUM(qsrsi.count_executions) / 1000.0 AS WeightedAvgDuration,
        'High resource consumption query' AS Classification,
        'Consider optimization or indexing improvements' AS Recommendation
    FROM sys.query_store_query_text qsqt
    INNER JOIN sys.query_store_query qsq ON qsqt.query_text_id = qsq.query_text_id
    INNER JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
    INNER JOIN sys.query_store_runtime_stats qsrsi ON qsp.plan_id = qsrsi.plan_id
    WHERE (@DatabaseName IS NULL OR DB_NAME(qsq.database_id) = @DatabaseName)
    AND qsrsi.last_execution_time >= DATEADD(day, -@DaysToAnalyze, GETDATE())
    GROUP BY qsqt.query_text_id, qsq.query_id, qsp.plan_id
    ORDER BY WeightedAvgDuration DESC;
    
    PRINT 'Query Store analysis completed.';
    
    RETURN 0;
END
GO
```

## Practical Troubleshooting Scenarios {#scenarios}

### Scenario 1: Sudden Performance Degradation

**Problem:** A production database that previously performed well suddenly experiences significant slowdowns with queries taking much longer to complete.

**Investigation Process:**

```sql
-- Step 1: Check recent changes and current load
EXEC sp_WhoIsActive @GetFullInnerText = 1, @GetTransactionInfo = 1, @GetWaitStats = 1;

-- Step 2: Identify resource bottlenecks
EXEC sp_CpuUtilizationAnalysis @IncludeQueryLevelAnalysis = 1, @GenerateRecommendations = 1;

-- Step 3: Check for blocking issues
EXEC sp_BlockingAndDeadlockAnalysis @IncludeCurrentBlocking = 1, @IncludeLockAnalysis = 1;

-- Step 4: Analyze query performance changes
EXEC sp_QueryStoreAnalysis @AnalysisType = 'REGRESSION', @PerformanceDropThreshold = 30.0;
```

**Likely Causes and Solutions:**
1. **Statistics Outdated**: Update statistics for frequently changed tables
2. **Index Fragmentation**: Rebuild highly fragmented indexes
3. **Parameter Sniffing**: Use OPTION (RECOMPILE) or create stored procedures
4. **Locking Issues**: Identify and resolve blocking queries
5. **Resource Contention**: Monitor CPU, memory, and I/O utilization

### Scenario 2: High Transaction Log Usage

**Problem:** Transaction log file is growing rapidly and approaching disk space limits.

**Investigation:**

```sql
-- Check transaction log usage and growth patterns
SELECT 
    d.name AS DatabaseName,
    df.name AS LogFileName,
    df.size * 8 / 1024 AS CurrentSizeMB,
    CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 8 / 1024 AS UsedSpaceMB,
    (df.size - CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint)) * 8 / 1024 AS FreeSpaceMB,
    CAST((CAST(FILEPROPERTY(df.name, 'SpaceUsed') AS bigint) * 100.0 / df.size) AS decimal(5,2)) AS PercentUsed,
    df.growth,
    df.max_size
FROM sys.databases d
CROSS APPLY sys.database_files df
WHERE df.type = 2 -- Log files only
ORDER BY d.name;

-- Check for long-running transactions
SELECT 
    session_id,
    database_id,
    transaction_begin_time,
    DATEDIFF(minute, transaction_begin_time, GETDATE()) AS TransactionDurationMinutes,
    CASE transaction_type
        WHEN 1 THEN 'Read/Write Transaction'
        WHEN 2 THEN 'Read-Only Transaction'
        WHEN 3 THEN 'System Transaction'
        WHEN 4 THEN 'Distributed Transaction'
        ELSE 'Unknown'
    END AS TransactionType,
    CASE transaction_state
        WHEN 0 THEN 'Not Initialized'
        WHEN 1 THEN 'Not Started'
        WHEN 2 THEN 'Active'
        WHEN 3 THEN 'Ended (Read-Only)'
        WHEN 4 THEN 'Commit Initiated'
        WHEN 5 THEN 'Prepared'
        WHEN 6 THEN 'Committed'
        WHEN 7 THEN 'Rolling Back'
        WHEN 8 THEN 'Rolled Back'
        ELSE 'Unknown'
    END AS TransactionState
FROM sys.dm_tran_active_transactions
WHERE database_id = DB_ID('YourDatabaseName')
ORDER BY transaction_begin_time;
```

**Solutions:**
1. **Backup Transaction Log**: Regular log backups in FULL recovery model
2. **Check for Long Transactions**: Identify and commit/rollback long-running transactions
3. **Use BULK_LOGGED Recovery**: For bulk operations
4. **Shrink Log File**: If space is needed immediately (temporary solution)
5. **Optimize Transactions**: Break large transactions into smaller batches

### Scenario 3: Deadlock Issues

**Problem:** Application reports frequent deadlock errors affecting user experience.

**Investigation:**

```sql
-- Enable deadlock trace flag
DBCC TRACEON (1222, -1);

-- Check current blocking
EXEC sp_BlockingAndDeadlockAnalysis @IncludeDeadlockAnalysis = 1, @IncludeLockAnalysis = 1;

-- Analyze deadlock patterns from system health session
SELECT TOP 10
    event_data.value('(event[@name=''xml_deadlock_report'']/@timestamp)[1]', 'datetime') AS EventTime,
    event_data.value('(event[@name=''xml_deadlock_report'']/deadlock/victim-list/victim/@id)[1]', 'varchar(50)') AS VictimProcessId,
    event_data.value('(event[@name=''xml_deadlock_report'']/deadlock/process-list/process/@spid)[1]', 'int') AS VictimSpid,
    event_data.value('(event[@name=''xml_deadlock_report'']/deadlock/process-list/process/@loginname)[1]', 'varchar(100)') AS VictimLogin,
    'Deadlock details available in XML event data' AS Analysis
FROM sys.fn_xe_file_target_read_file('system_health*.xel', NULL, NULL, NULL) AS target_file
CROSS APPLY (SELECT CAST(target_file.event_data AS xml)) AS event_data
WHERE target_file.event_data LIKE '%xml_deadlock_report%'
ORDER BY event_data.value('(event[@name=''xml_deadlock_report'']/@timestamp)[1]', 'datetime') DESC;
```

**Solutions:**
1. **Analyze Deadlock Graph**: Review deadlock XML to identify conflicting queries
2. **Optimize Query Order**: Ensure consistent query ordering across transactions
3. **Use Appropriate Isolation Levels**: Consider READ COMMITTED SNAPSHOT
4. **Add Indexes**: Reduce lock duration by optimizing queries
5. **Implement Retry Logic**: Application-level deadlock retry mechanism

### Scenario 4: Backup Failures

**Problem:** Automated backup jobs are failing, putting data at risk.

**Investigation:**

```sql
-- Check recent backup history for failures
SELECT TOP 20
    d.name AS DatabaseName,
    bs.type_desc AS BackupType,
    bs.backup_start_date AS BackupStartTime,
    bs.backup_finish_date AS BackupEndTime,
    DATEDIFF(minute, bs.backup_start_date, bs.backup_finish_date) AS DurationMinutes,
    bs.backup_size / 1024 / 1024 AS BackupSizeMB,
    bs.compressed_backup_size / 1024 / 1024 AS CompressedSizeMB,
    CASE 
        WHEN bs.backup_finish_date IS NULL THEN 'FAILED'
        WHEN bs.backup_finish_date > bs.backup_start_date THEN 'SUCCESS'
        ELSE 'UNKNOWN'
    END AS BackupStatus,
    CASE 
        WHEN bs.backup_finish_date IS NULL THEN 'Backup did not complete'
        WHEN DATEDIFF(minute, bs.backup_start_date, bs.backup_finish_date) > 120 THEN 'UNUSUALLY_LONG'
        WHEN DATEDIFF(minute, bs.backup_start_date, bs.backup_finish_date) < 1 THEN 'UNUSUALLY_SHORT'
        ELSE 'NORMAL'
    END AS BackupAnalysis
FROM msdb.dbo.backupset bs
INNER JOIN sys.databases d ON bs.database_name = d.name
WHERE bs.backup_start_date >= DATEADD(day, -7, GETDATE())
ORDER BY bs.backup_start_date DESC;

-- Check backup device information
SELECT 
    bs.backup_set_id,
    bs.database_name,
    bs.backup_start_date,
    bs.backup_finish_date,
    bs.backup_size,
    bs.compressed_backup_size,
    bmf.physical_device_name AS BackupFilePath,
    CASE 
        WHEN bmf.physical_device_name LIKE 'NUL%' THEN 'NULL_DEVICE'
        WHEN bmf.physical_device_name LIKE '\\%' THEN 'NETWORK_SHARE'
        WHEN bmf.physical_device_name LIKE '%:%' THEN 'LOCAL_DISK'
        ELSE 'OTHER'
    END AS DeviceType,
    CASE 
        WHEN bmf.physical_device_name LIKE 'NUL%' THEN 'Using null device - backup not actually created'
        WHEN bmf.physical_device_name LIKE '\\%' THEN 'Network backup - check network connectivity'
        WHEN bmf.physical_device_name LIKE '%:%' THEN 'Local disk backup'
        ELSE 'Other backup device type'
    END AS DeviceAnalysis
FROM msdb.dbo.backupset bs
INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.backup_start_date >= DATEADD(day, -3, GETDATE())
ORDER BY bs.backup_start_date DESC;

-- Check for backup job failures
SELECT TOP 20
    j.name AS JobName,
    jh.run_date,
    jh.run_time,
    jh.run_status,
    jh.run_duration,
    jh.message,
    CASE 
        WHEN jh.run_status = 0 THEN 'Failed'
        WHEN jh.run_status = 1 THEN 'Succeeded'
        WHEN jh.run_status = 2 THEN 'Retry'
        WHEN jh.run_status = 3 THEN 'Canceled'
        ELSE 'Unknown'
    END AS JobStatus
FROM msdb.dbo.sysjobhistory jh
INNER JOIN msdb.dbo.sysjobs j ON jh.job_id = j.job_id
WHERE jh.run_date >= CONVERT(int, CONVERT(varchar(8), DATEADD(day, -7, GETDATE()), 112))
AND j.name LIKE '%backup%'
ORDER BY jh.run_date DESC, jh.run_time DESC;
```

**Solutions:**
1. **Check Disk Space**: Ensure sufficient space for backup files
2. **Verify Network Access**: For network-based backups
3. **Review Backup Paths**: Confirm backup locations exist and are accessible
4. **Check Permissions**: Verify service account has backup permissions
5. **Test Backup Restoration**: Periodically test backup integrity

This comprehensive week 12 curriculum provides DBAs with advanced troubleshooting skills and diagnostic techniques needed to quickly identify, analyze, and resolve SQL Server issues. By mastering these concepts and practical scenarios, you'll be able to maintain optimal database performance and availability in any environment.
