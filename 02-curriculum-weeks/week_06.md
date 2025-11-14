# Week 6: Monitoring and Performance

## Learning Objectives

By the end of this week, students will be able to:

1. **Implement Comprehensive Monitoring**: Set up monitoring for SQL Server components including databases, queries, and system resources
2. **Analyze Performance Metrics**: Interpret SQL Server performance data to identify bottlenecks and optimization opportunities
3. **Optimize Query Performance**: Identify and resolve slow-running queries using execution plans and indexing strategies
4. **Manage Indexes Effectively**: Create, maintain, and optimize indexes for improved query performance
5. **Implement Proactive Maintenance**: Establish regular maintenance routines for optimal database health
6. **Use Advanced Monitoring Tools**: Leverage Extended Events, Performance Dashboard, and other monitoring capabilities

## Theoretical Content

### 1. SQL Server Monitoring Architecture

#### 1.1 Monitoring Layers Overview

SQL Server monitoring operates at multiple layers to provide comprehensive visibility:

**Infrastructure Layer**:
- **CPU Utilization**: Processor usage, context switching, interrupts
- **Memory Usage**: Available memory, page file usage, memory pressure
- **Disk I/O**: Read/write operations, queue depths, disk latency
- **Network**: Bandwidth utilization, connection counts, latency

**SQL Server Engine Layer**:
- **Database Engine**: Instance health, connections, transactions
- **Query Performance**: Execution plans, waits, resource consumption
- **Buffer Pool**: Memory utilization, page life expectancy, buffer cache hit ratio
- **Transaction Log**: Log growth, backup status, transaction activity

**Application Layer**:
- **Application Response Time**: End-to-end performance from user perspective
- **Query Patterns**: Most frequently executed queries, slow queries
- **Connection Patterns**: Connection pooling, login/logout rates
- **Error Rates**: Failed connections, timeout rates, application errors

#### 1.2 Dynamic Management Views and Functions

SQL Server provides Dynamic Management Views (DMVs) and Dynamic Management Functions (DMFs) for comprehensive monitoring:

**Server-Level DMVs**:
```sql
-- Monitor current sessions and requests
SELECT 
    session_id,
    login_time,
    host_name,
    program_name,
    login_name,
    status,
    total_scheduled_time,
    total_elapsed_time,
    last_request_start_time,
    last_request_end_time,
    reads,
    writes,
    logical_reads,
    cpu_time
FROM sys.dm_exec_sessions
WHERE is_user_process = 1;

-- Monitor active requests
SELECT 
    r.session_id,
    s.login_name,
    s.host_name,
    r.status,
    r.command,
    r.cpu_time,
    r.total_elapsed_time,
    r.reads,
    r.writes,
    r.logical_reads,
    r.blocking_session_id,
    r.wait_type,
    r.wait_time,
    r.last_wait_type,
    SUBSTRING(st.text, (r.statement_start_offset/2)+1,
        ((CASE r.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE r.statement_end_offset
        END - r.statement_start_offset)/2) + 1) AS statement_text
FROM sys.dm_exec_requests r
LEFT JOIN sys.dm_exec_sessions s ON r.session_id = s.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.session_id <> @@SPID;
```

**Database-Level DMVs**:
```sql
-- Monitor database file I/O
SELECT 
    DB_NAME(database_id) AS database_name,
    file_id,
    io_stall_read_ms / NULLIF(num_of_reads, 0) AS avg_read_latency,
    io_stall_write_ms / NULLIF(num_of_writes, 0) AS avg_write_latency,
    io_stall / (num_of_reads + num_of_writes) AS avg_io_latency,
    num_of_reads,
    num_of_writes,
    io_stall_read_ms,
    io_stall_write_ms,
    size_on_disk_bytes / 1024/1024 AS size_on_disk_mb
FROM sys.dm_io_virtual_file_stats(NULL, NULL)
ORDER BY avg_io_latency DESC;

-- Monitor index fragmentation
SELECT 
    OBJECT_NAME(ips.object_id) AS table_name,
    i.name AS index_name,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.record_count,
    ips.avg_page_space_used_in_percent,
    CASE 
        WHEN ips.avg_fragmentation_in_percent < 10 THEN 'No Action'
        WHEN ips.avg_fragmentation_in_percent < 30 THEN 'Reorganize'
        ELSE 'Rebuild'
    END AS recommended_action
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 5
AND ips.page_count > 100
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

**Query Performance DMVs**:
```sql
-- Find top resource-consuming queries
SELECT TOP 20
    qs.execution_count,
    qs.total_elapsed_time / 1000 as total_time_ms,
    (qs.total_elapsed_time / qs.execution_count) / 1000 as avg_time_ms,
    qs.total_logical_reads / 1024 as total_logical_reads_kb,
    qs.total_physical_reads / 1024 as total_physical_reads_kb,
    qs.total_worker_time / 1000 as total_cpu_time_ms,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS StatementText
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.last_execution_time > DATEADD(day, -1, GETDATE())
ORDER BY qs.total_elapsed_time DESC;

-- Identify missing indexes
SELECT 
    mid.statement,
    migs.avg_user_impact,
    migs.avg_total_user_cost,
    migs.user_seeks,
    migs.user_scans,
    migs.user_lookups,
    migs.last_user_seek,
    'CREATE INDEX IX_' + REPLACE(REPLACE(REPLACE(mid.statement, '[', ''), ']', ''), '.', '_') + 
    '_' + CAST(migs.index_handle AS VARCHAR(10)) +
    ' ON ' + REPLACE(mid.statement, '.', '.') +
    ' (' + ISNULL(mid.equality_columns, '') +
    CASE WHEN mid.equality_columns IS NOT NULL AND mid.inequality_columns IS NOT NULL THEN ', ' ELSE '' END +
    ISNULL(mid.inequality_columns, '') + ')' +
    CASE WHEN mid.included_columns IS NOT NULL THEN ' INCLUDE (' + mid.included_columns + ')' ELSE '' END +
    ' WITH (ONLINE = ON);' AS create_index_statement
FROM sys.dm_db_missing_index_details mid
INNER JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE migs.avg_user_impact > 10
ORDER BY migs.avg_total_user_cost DESC;
```

#### 1.3 Performance Counters and Wait Statistics

**System Performance Counters**:
```sql
-- Monitor buffer cache usage
SELECT 
    OBJECT_NAME,
    counter_name,
    cntr_value,
    CASE 
        WHEN counter_name = 'Buffer cache hit ratio' AND cntr_value < 95 THEN 'Poor Performance'
        WHEN counter_name = 'Buffer cache hit ratio' AND cntr_value < 98 THEN 'Acceptable Performance'
        WHEN counter_name = 'Buffer cache hit ratio' THEN 'Good Performance'
        ELSE 'Monitor'
    END AS performance_status
FROM sys.dm_os_performance_counters
WHERE counter_name IN (
    'Buffer cache hit ratio',
    'Page life expectancy',
    'Free list stalls/sec',
    'Page reads/sec',
    'Page writes/sec',
    'Lazy writes/sec',
    'Checkpoint pages/sec',
    'Database pages'
)
AND OBJECT_NAME = 'MSSQL$INSTANCE:Buffer Manager';

-- Monitor transaction log usage
SELECT 
    DB_NAME(database_id) AS database_name,
    name AS log_file_name,
    (size * 8.0) / 1024 AS size_mb,
    (FILEPROPERTY(name, 'SpaceUsed') * 8.0) / 1024 AS used_mb,
    (CAST(FILEPROPERTY(name, 'SpaceUsed') AS FLOAT) / size * 100) AS percent_used,
    CASE 
        WHEN (CAST(FILEPROPERTY(name, 'SpaceUsed') AS FLOAT) / size * 100) > 90 THEN 'Critical'
        WHEN (CAST(FILEPROPERTY(name, 'SpaceUsed') AS FLOAT) / size * 100) > 80 THEN 'Warning'
        ELSE 'Normal'
    END AS log_status
FROM sys.database_files
WHERE type_desc = 'LOG';
```

**Wait Statistics Analysis**:
```sql
-- Analyze wait statistics
SELECT 
    wait_type,
    waiting_tasks_count,
    wait_time_ms / 1000.0 AS wait_time_seconds,
    max_wait_time_ms / 1000.0 AS max_wait_time_seconds,
    signal_wait_time_ms / 1000.0 AS signal_wait_time_seconds,
    wait_time_ms / NULLIF(waiting_tasks_count, 0) AS avg_wait_time_ms,
    CASE 
        WHEN wait_type LIKE '%PAGEIOLATCH%' THEN 'I/O Bottleneck'
        WHEN wait_type LIKE '%LCK%' THEN 'Locking Issues'
        WHEN wait_type LIKE '%WRITELOG%' THEN 'Log I/O Bottleneck'
        WHEN wait_type LIKE '%CXPACKET%' THEN 'Parallelism Issues'
        WHEN wait_type LIKE '%SOS_SCHEDULER_YIELD%' THEN 'CPU Pressure'
        WHEN wait_type LIKE '%RESOURCE_SEMAPHORE%' THEN 'Memory Pressure'
        ELSE 'Other'
    END AS wait_category,
    CASE 
        WHEN wait_time_ms / NULLIF(waiting_tasks_count, 0) > 1000 THEN 'High Impact'
        WHEN wait_time_ms / NULLIF(waiting_tasks_count, 0) > 100 THEN 'Medium Impact'
        ELSE 'Low Impact'
    END AS impact_level
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'CLR_SEMAPHORE', 'LAZYWRITER_SLEEP', 'RESOURCE_QUEUE', 
    'SLEEP_TASK', 'SLEEP_SYSTEMTASK', 'SQLTRACE_BUFFER_FLUSH',
    'WAITFOR', 'LOGMGR_QUEUE', 'CHECKPOINT_QUEUE', 'REQUEST_FOR_DEADLOCK_SEARCH',
    'XE_TIMER_EVENT', 'XE_DISPATCHER_WAIT', 'FT_IFTS_SCHEDULER_IDLE_WAIT',
    'SQLTRACE_FILE_BUFFER', 'CLR_MANUAL_EVENT', 'CLR_AUTO_EVENT'
)
ORDER BY wait_time_ms DESC;
```

### 2. Query Performance Optimization

#### 2.1 Query Analysis and Execution Plans

**Execution Plan Analysis**:
```sql
-- Enable actual execution plan in SSMS
-- Tools > Options > Query Execution > Advanced
-- SET STATISTICS IO ON;
-- SET STATISTICS TIME ON;
-- Include Actual Execution Plan (Ctrl+M)

-- Analyze execution plan for specific query
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Example query analysis
SELECT 
    c.CustomerID,
    c.FirstName,
    c.LastName,
    c.Email,
    o.OrderID,
    o.OrderDate,
    od.ProductID,
    p.ProductName,
    od.Quantity,
    od.UnitPrice,
    (od.Quantity * od.UnitPrice) AS LineTotal
FROM Customers c
INNER JOIN Orders o ON c.CustomerID = o.CustomerID
INNER JOIN OrderDetails od ON o.OrderID = od.OrderID
INNER JOIN Products p ON od.ProductID = p.ProductID
WHERE o.OrderDate >= DATEADD(MONTH, -3, GETDATE())
AND c.CustomerID IN (SELECT TOP 100 CustomerID FROM Customers WHERE IsActive = 1)
ORDER BY o.OrderDate DESC;

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
```

**Query Store for Performance Analysis**:
```sql
-- Enable Query Store for performance tracking
ALTER DATABASE PerformanceDB
SET QUERY_STORE = ON
(
    OPERATION_MODE = READ_WRITE,
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    INTERVAL_LENGTH_MINUTES = 60,
    MAX_STORAGE_SIZE_MB = 1024,
    QUERY_CAPTURE_MODE = AUTO,
    SIZE_BASED_CLEANUP_MODE = AUTO
);

-- Find queries with worst average execution time
SELECT 
    qsq.query_id,
    qsqt.query_sql_text,
    qsq.avg_duration / 1000.0 AS avg_duration_ms,
    qsq.avg_cpu_time / 1000.0 AS avg_cpu_time_ms,
    qsq.avg_logical_io_reads,
    qsq.avg_logical_io_writes,
    qsq.avg_physical_io_reads,
    qsq.count_executions,
    qsq.first_execution_time,
    qsq.last_execution_time,
    CASE 
        WHEN qsq.avg_duration / 1000.0 > 5000 THEN 'Critical'
        WHEN qsq.avg_duration / 1000.0 > 1000 THEN 'High'
        WHEN qsq.avg_duration / 1000.0 > 100 THEN 'Medium'
        ELSE 'Low'
    END AS performance_impact
FROM sys.query_store_query qsq
INNER JOIN sys.query_store_query_text qsqt ON qsq.query_text_id = qsqt.query_text_id
WHERE qsq.last_execution_time > DATEADD(DAY, -7, GETDATE())
AND qsq.count_executions > 10
ORDER BY qsq.avg_duration DESC;

-- Find queries with most plan changes
SELECT 
    qsq.query_id,
    qsqt.query_sql_text,
    qsp.plan_id,
    qsp.query_plan,
    qsp.avg_compile_duration / 1000.0 AS avg_compile_duration_ms,
    qsp.last_compile_start_time,
    qsp.plan_count,
    'Plan Count: ' + CAST(qsp.plan_count AS VARCHAR(10)) AS plan_analysis
FROM sys.query_store_query qsq
INNER JOIN sys.query_store_query_text qsqt ON qsq.query_text_id = qsqt.query_text_id
INNER JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
WHERE qsp.plan_count > 3
ORDER BY qsp.plan_count DESC;
```

**Missing Index Analysis**:
```sql
-- Comprehensive missing index analysis
SELECT 
    'Missing Indexes' AS analysis_type,
    mid.statement AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.avg_user_impact,
    migs.avg_total_user_cost,
    migs.user_seeks,
    migs.user_scans,
    migs.user_lookups,
    migs.last_user_seek,
    'CREATE INDEX IX_' + 
    REPLACE(REPLACE(REPLACE(mid.statement, '[', ''), ']', ''), '.', '_') + '_' + 
    CAST(migs.index_handle AS VARCHAR(10)) + ' ON ' + 
    mid.statement + 
    ' (' + 
    ISNULL(mid.equality_columns, '') + 
    CASE WHEN mid.equality_columns IS NOT NULL AND mid.inequality_columns IS NOT NULL THEN ', ' ELSE '' END +
    ISNULL(mid.inequality_columns, '') + ')' +
    CASE WHEN mid.included_columns IS NOT NULL THEN ' INCLUDE (' + mid.included_columns + ')' ELSE '' END +
    ' WITH (ONLINE = ON, FILLFACTOR = 90);' AS create_index_ddl,
    CASE 
        WHEN migs.avg_user_impact > 70 THEN 'High Impact - Create Index'
        WHEN migs.avg_user_impact > 30 THEN 'Medium Impact - Consider Index'
        ELSE 'Low Impact - Monitor'
    END AS recommendation
FROM sys.dm_db_missing_index_details mid
INNER JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE migs.avg_user_impact > 10
AND migs.user_seeks + migs.user_scans > 100
ORDER BY migs.avg_total_user_cost DESC;
```

#### 2.2 Index Management and Optimization

**Index Fragmentation Analysis**:
```sql
-- Comprehensive index analysis
SELECT 
    OBJECT_SCHEMA_NAME(ips.object_id) AS schema_name,
    OBJECT_NAME(ips.object_id) AS table_name,
    i.name AS index_name,
    i.type_desc AS index_type,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.record_count,
    ips.avg_page_space_used_in_percent,
    ips.index_level,
    ips.leaf_level_insert_count,
    ips.leaf_level_update_count,
    ips.leaf_level_delete_count,
    CASE 
        WHEN ips.avg_fragmentation_in_percent < 10 THEN 'No Action'
        WHEN ips.avg_fragmentation_in_percent < 30 THEN 'Reorganize Index'
        WHEN ips.page_count > 1000 THEN 'Rebuild Index (Large)'
        ELSE 'Rebuild Index'
    END AS recommended_action,
    CASE 
        WHEN ips.avg_fragmentation_in_percent < 10 THEN 0
        WHEN ips.avg_fragmentation_in_percent < 30 THEN 1
        ELSE 2
    END AS maintenance_priority
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 5
AND ips.page_count > 10
ORDER BY ips.avg_fragmentation_in_percent DESC, ips.page_count DESC;

-- Index usage statistics
SELECT 
    OBJECT_SCHEMA_NAME(ius.object_id) AS schema_name,
    OBJECT_NAME(ius.object_id) AS table_name,
    i.name AS index_name,
    ius.user_seeks,
    ius.user_scans,
    ius.user_lookups,
    ius.user_updates,
    ius.last_user_seek,
    ius.last_user_scan,
    ius.last_user_lookup,
    CASE 
        WHEN ius.user_seeks + ius.user_scans + ius.user_lookups = 0 THEN 'Unused Index'
        WHEN ius.user_updates > (ius.user_seeks + ius.user_scans + ius.user_lookups) * 10 THEN 'High Maintenance Index'
        WHEN ius.user_seeks + ius.user_scans + ius.user_lookups < 10 THEN 'Rarely Used Index'
        ELSE 'Active Index'
    END AS usage_category,
    'DROP INDEX ' + i.name + ' ON ' + OBJECT_SCHEMA_NAME(ius.object_id) + '.' + OBJECT_NAME(ius.object_id) + ';' AS drop_statement
FROM sys.dm_db_index_usage_stats ius
INNER JOIN sys.indexes i ON ius.object_id = i.object_id AND ius.index_id = i.index_id
WHERE ius.database_id = DB_ID()
AND i.name NOT LIKE 'PK_%'
AND OBJECT_NAME(ius.object_id) NOT LIKE 'sys%'
ORDER BY (ius.user_seeks + ius.user_scans + ius.user_lookups) ASC;
```

**Index Maintenance Procedures**:
```sql
-- Automated index maintenance procedure
CREATE PROCEDURE sp_IndexMaintenance
    @DatabaseName NVARCHAR(128) = NULL,
    @RebuildThreshold DECIMAL(5,2) = 30.0,  -- Rebuild if fragmentation > 30%
    @ReorganizeThreshold DECIMAL(5,2) = 10.0, -- Reorganize if fragmentation > 10%
    @MinPageCount INT = 100,  -- Only maintain indexes with > 100 pages
    @OnlineRebuild BIT = 1,   -- Use ONLINE = ON for rebuilds
    @OutputResults BIT = 1    -- Return maintenance results
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @IndexName NVARCHAR(128);
    DECLARE @SchemaName NVARCHAR(128);
    DECLARE @TableName NVARCHAR(128);
    DECLARE @Fragmentation DECIMAL(5,2);
    DECLARE @Action NVARCHAR(20);
    DECLARE @StartTime DATETIME2;
    DECLARE @EndTime DATETIME2;
    DECLARE @Duration INT;
    
    -- Create table to store results
    CREATE TABLE #IndexMaintenanceResults (
        SchemaName NVARCHAR(128),
        TableName NVARCHAR(128),
        IndexName NVARCHAR(128),
        Action NVARCHAR(20),
        Fragmentation DECIMAL(5,2),
        StartTime DATETIME2,
        EndTime DATETIME2,
        DurationSeconds INT,
        Status NVARCHAR(20),
        ErrorMessage NVARCHAR(MAX)
    );
    
    -- Cursor through indexes needing maintenance
    DECLARE maintenance_cursor CURSOR FOR
    SELECT 
        OBJECT_SCHEMA_NAME(ips.object_id) AS schema_name,
        OBJECT_NAME(ips.object_id) AS table_name,
        i.name AS index_name,
        ips.avg_fragmentation_in_percent
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
    INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE ips.avg_fragmentation_in_percent > @ReorganizeThreshold
    AND ips.page_count > @MinPageCount
    AND i.name NOT LIKE 'PK_%'  -- Skip primary key indexes
    AND i.name NOT LIKE '_WA_%'  -- Skip worktable indexes
    ORDER BY ips.avg_fragmentation_in_percent DESC;
    
    OPEN maintenance_cursor;
    FETCH NEXT FROM maintenance_cursor INTO @SchemaName, @TableName, @IndexName, @Fragmentation;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @StartTime = GETDATE();
        SET @Action = CASE 
            WHEN @Fragmentation >= @RebuildThreshold THEN 'REBUILD'
            ELSE 'REORGANIZE'
        END;
        
        BEGIN TRY
            -- Generate maintenance SQL
            IF @Action = 'REBUILD'
            BEGIN
                SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @SchemaName + '.' + @TableName;
                IF @OnlineRebuild = 1
                    SET @SQL = @SQL + ' REBUILD WITH (ONLINE = ON, FILLFACTOR = 90)';
                ELSE
                    SET @SQL = @SQL + ' REBUILD WITH (FILLFACTOR = 90)';
            END
            ELSE
            BEGIN
                SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @SchemaName + '.' + @TableName + ' REORGANIZE';
            END
            
            -- Execute maintenance
            EXEC sp_executesql @SQL;
            
            SET @EndTime = GETDATE();
            SET @Duration = DATEDIFF(SECOND, @StartTime, @EndTime);
            
            -- Log successful maintenance
            INSERT INTO #IndexMaintenanceResults
            VALUES (@SchemaName, @TableName, @IndexName, @Action, @Fragmentation, @StartTime, @EndTime, @Duration, 'SUCCESS', NULL);
            
        END TRY
        BEGIN CATCH
            SET @EndTime = GETDATE();
            SET @Duration = DATEDIFF(SECOND, @StartTime, @EndTime);
            
            -- Log failed maintenance
            INSERT INTO #IndexMaintenanceResults
            VALUES (@SchemaName, @TableName, @IndexName, @Action, @Fragmentation, @StartTime, @EndTime, @Duration, 'FAILED', ERROR_MESSAGE());
        END CATCH
        
        FETCH NEXT FROM maintenance_cursor INTO @SchemaName, @TableName, @IndexName, @Fragmentation;
    END
    
    CLOSE maintenance_cursor;
    DEALLOCATE maintenance_cursor;
    
    -- Update statistics on all tables
    SET @SQL = 'EXEC sp_updatestats';
    EXEC sp_executesql @SQL;
    
    -- Return results if requested
    IF @OutputResults = 1
    BEGIN
        SELECT * FROM #IndexMaintenanceResults ORDER BY StartTime;
        
        -- Summary statistics
        SELECT 
            'Index Maintenance Summary' AS summary_type,
            COUNT(*) AS total_indexes_maintained,
            SUM(CASE WHEN Status = 'SUCCESS' THEN 1 ELSE 0 END) AS successful_maintenance,
            SUM(CASE WHEN Status = 'FAILED' THEN 1 ELSE 0 END) AS failed_maintenance,
            SUM(CASE WHEN Action = 'REBUILD' THEN 1 ELSE 0 END) AS indexes_rebuilt,
            SUM(CASE WHEN Action = 'REORGANIZE' THEN 1 ELSE 0 END) AS indexes_reorganized,
            AVG(DurationSeconds) AS avg_duration_seconds,
            SUM(DurationSeconds) AS total_duration_seconds
        FROM #IndexMaintenanceResults;
    END
    
    DROP TABLE #IndexMaintenanceResults;
END;
```

#### 2.3 Query Optimization Techniques

**Parameter Sniffing Solutions**:
```sql
-- Solution 1: OPTION (OPTIMIZE FOR UNKNOWN)
SELECT 
    c.CustomerID,
    c.FirstName,
    c.LastName,
    COUNT(o.OrderID) AS OrderCount,
    SUM(o.TotalAmount) AS TotalSpent
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
WHERE c.Region = @Region
GROUP BY c.CustomerID, c.FirstName, c.LastName
ORDER BY TotalSpent DESC
OPTION (OPTIMIZE FOR UNKNOWN);

-- Solution 2: Plan Guide for specific query pattern
CREATE TABLE #CustomerOrders (Region NVARCHAR(50));
INSERT INTO #CustomerOrders VALUES ('North');

-- Create plan guide
EXEC sp_create_plan_guide 
    @name = N'CustomerOrders_Plan_Guide',
    @stmt = N'SELECT 
    c.CustomerID,
    c.FirstName,
    c.LastName,
    COUNT(o.OrderID) AS OrderCount,
    SUM(o.TotalAmount) AS TotalSpent
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
WHERE c.Region = @Region
GROUP BY c.CustomerID, c.FirstName, c.LastName
ORDER BY TotalSpent DESC',
    @type = N'SQL',
    @module_or_batch = NULL,
    @params = N'@Region NVARCHAR(50)',
    @hints = N'OPTION (OPTIMIZE FOR UNKNOWN)';

-- Solution 3: Recompile option
SELECT 
    c.CustomerID,
    c.FirstName,
    c.LastName,
    COUNT(o.OrderID) AS OrderCount,
    SUM(o.TotalAmount) AS TotalSpent
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
WHERE c.Region = @Region
GROUP BY c.CustomerID, c.FirstName, c.LastName
ORDER BY TotalSpent DESC
OPTION (RECOMPILE);
```

**Statistics Management**:
```sql
-- Auto-update statistics monitoring
SELECT 
    OBJECT_SCHEMA_NAME(sp.object_id) AS schema_name,
    OBJECT_NAME(sp.object_id) AS table_name,
    sp.name AS statistics_name,
    sp.stats_id,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    CAST(sp.rows_sampled * 100.0 / NULLIF(sp.rows, 0) AS DECIMAL(5,2)) AS sample_percent,
    CASE 
        WHEN sp.rows_sampled * 100.0 / NULLIF(sp.rows, 0) < 10 THEN 'Low Sample'
        WHEN sp.rows_sampled * 100.0 / NULLIF(sp.rows, 0) < 50 THEN 'Medium Sample'
        ELSE 'Good Sample'
    END AS sample_quality,
    CASE 
        WHEN sp.last_updated < DATEADD(DAY, -7, GETDATE()) THEN 'Stale Statistics'
        WHEN sp.last_updated < DATEADD(DAY, -1, GETDATE()) THEN 'Recent'
        ELSE 'Current'
    END AS statistics_age
FROM sys.stats sp
WHERE sp.object_id IN (SELECT object_id FROM sys.objects WHERE type = 'U')
AND sp.name NOT LIKE '_WA%'  -- Exclude auto-created statistics
ORDER BY sp.last_updated ASC;

-- Manual statistics update for critical tables
CREATE PROCEDURE sp_UpdateStatistics
    @TableName NVARCHAR(128) = NULL,
    @SamplePercent INT = NULL  -- NULL for auto, or specific percentage
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX);
    
    IF @TableName IS NOT NULL
    BEGIN
        -- Update statistics for specific table
        IF @SamplePercent IS NOT NULL
            SET @SQL = 'UPDATE STATISTICS ' + @TableName + ' WITH SAMPLE ' + CAST(@SamplePercent AS VARCHAR(10)) + ' PERCENT';
        ELSE
            SET @SQL = 'UPDATE STATISTICS ' + @TableName + ' WITH FULLSCAN';
        
        EXEC sp_executesql @SQL;
        
        PRINT 'Statistics updated for table: ' + @TableName;
    END
    ELSE
    BEGIN
        -- Update statistics for all user tables
        DECLARE stats_cursor CURSOR FOR
        SELECT 'UPDATE STATISTICS ' + OBJECT_SCHEMA_NAME(object_id) + '.' + name + ' WITH FULLSCAN' AS update_statement
        FROM sys.objects
        WHERE type = 'U'
        ORDER BY name;
        
        OPEN stats_cursor;
        DECLARE @UpdateStatement NVARCHAR(MAX);
        
        FETCH NEXT FROM stats_cursor INTO @UpdateStatement;
        
        WHILE @@FETCH_STATUS = 0
        BEGIN
            EXEC sp_executesql @UpdateStatement;
            PRINT 'Statistics updated: ' + REPLACE(@UpdateStatement, 'UPDATE STATISTICS ', '');
            
            FETCH NEXT FROM stats_cursor INTO @UpdateStatement;
        END
        
        CLOSE stats_cursor;
        DEALLOCATE stats_cursor;
    END
END;
```

### 3. System Performance Monitoring

#### 3.1 Resource Utilization Monitoring

**CPU Performance Monitoring**:
```sql
-- CPU usage over time
SELECT 
    DATEADD(MINUTE, -(datepart(MINUTE, create_date) % 15), create_date) AS bucket_time,
    COUNT(*) AS session_count,
    AVG(cpu_time) AS avg_cpu_ms,
    SUM(cpu_time) AS total_cpu_ms,
    MAX(cpu_time) AS max_cpu_ms,
    AVG(total_scheduled_time) AS avg_scheduled_ms
FROM sys.dm_exec_sessions
WHERE is_user_process = 1
AND create_date >= DATEADD(HOUR, -24, GETDATE())
GROUP BY DATEADD(MINUTE, -(datepart(MINUTE, create_date) % 15), create_date)
ORDER BY bucket_time DESC;

-- Find CPU-intensive queries
SELECT TOP 20
    qs.execution_count,
    qs.total_worker_time / 1000.0 AS total_cpu_time_ms,
    qs.avg_worker_time / 1000.0 AS avg_cpu_time_ms,
    qs.total_elapsed_time / 1000.0 AS total_elapsed_time_ms,
    qs.avg_elapsed_time / 1000.0 AS avg_elapsed_time_ms,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS statement_text,
    'Query consumes significant CPU time' AS cpu_analysis
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.total_worker_time > 1000000  -- More than 1000 seconds total CPU time
AND qs.last_execution_time > DATEADD(HOUR, -1, GETDATE())
ORDER BY qs.total_worker_time DESC;

-- Monitor CPU pressure indicators
SELECT 
    'CPU Pressure Analysis' AS analysis_type,
    CASE 
        WHEN SQLProcessUtilization < 80 THEN 'Normal CPU Usage'
        WHEN SQLProcessUtilization < 95 THEN 'High CPU Usage - Monitor'
        ELSE 'Critical CPU Usage - Immediate Action Required'
    END AS cpu_status,
    SQLProcessUtilization,
    SystemIdle,
    100 - SystemIdle - SQLProcessUtilization AS OtherProcesses,
    sample_time
FROM (
    SELECT 
        record.value('(./Record/@id)[1]', 'int') AS record_id,
        record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
        record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SQLProcessUtilization)[1]', 'int') AS SQLProcessUtilization,
        record.value('(./Record/@id)[1]', 'int') AS sample_time
    FROM (
        SELECT TOP 1 CAST(record AS XML) AS record
        FROM sys.dm_os_ring_buffers
        WHERE ring_buffer_type = 'RING_BUFFER_SCHEDULER_MONITOR'
        ORDER BY timestamp DESC
    ) AS x
    CROSS APPLY x.record.nodes('/Record') AS r(record)
) AS y
ORDER BY sample_time DESC;
```

**Memory Pressure Analysis**:
```sql
-- Buffer pool usage analysis
SELECT 
    'Buffer Pool Analysis' AS analysis_type,
    DB_NAME(database_id) AS database_name,
    file_id,
    page_id,
    page_type,
    page_level,
    row_count,
    free_space_in_bytes,
    is_modified,
    CASE 
        WHEN free_space_in_bytes > 0 THEN 'Page has free space'
        ELSE 'Page is full'
    END AS page_status
FROM sys.dm_db_database_page_allocations(DB_ID(), NULL, NULL, NULL, 'DETAILED')
WHERE page_type IN (1, 2, 3)  -- Data, Index, Text pages
AND free_space_in_bytes > 8192  -- More than 8KB free space
ORDER BY free_space_in_bytes DESC;

-- Memory clerk analysis
SELECT 
    type,
    SUM(single_pages_kb + multi_pages_kb + virtual_memory_committed_kb + shared_memory_committed_kb) / 1024.0 AS total_memory_mb,
    SUM(single_pages_kb) / 1024.0 AS single_pages_mb,
    SUM(multi_pages_kb) / 1024.0 AS multi_pages_mb,
    SUM(virtual_memory_committed_kb) / 1024.0 AS virtual_memory_mb,
    SUM(shared_memory_committed_kb) / 1024.0 AS shared_memory_mb,
    COUNT(*) AS page_count
FROM sys.dm_os_memory_clerks
WHERE single_pages_kb + multi_pages_kb + virtual_memory_committed_kb + shared_memory_committed_kb > 1024  -- More than 1MB
GROUP BY type
ORDER BY total_memory_mb DESC;

-- Page life expectancy analysis
SELECT 
    OBJECT_NAME(object_id) AS table_name,
    index_id,
    page_id,
    row_count,
    free_space_in_bytes,
    CASE 
        WHEN free_space_in_bytes > 8192 THEN 'Good page'
        WHEN free_space_in_bytes > 0 THEN 'Average page'
        ELSE 'Full page'
    END AS page_condition
FROM sys.dm_db_database_page_allocations(DB_ID(), NULL, NULL, NULL, 'LIMITED')
WHERE page_type = 1  -- Data pages only
ORDER BY free_space_in_bytes DESC;

-- Memory grants analysis
SELECT 
    mg.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    mg.granted_memory_kb / 1024.0 AS granted_memory_mb,
    mg.required_memory_kb / 1024.0 AS required_memory_mb,
    mg.used_memory_kb / 1024.0 AS used_memory_mb,
    mg.max_used_memory_kb / 1024.0 AS max_used_memory_mb,
    mg.grant_time,
    mg.wait_time_ms,
    mg.query_cost,
    SUBSTRING(st.text, (r.statement_start_offset/2)+1,
        ((CASE r.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE r.statement_end_offset
        END - r.statement_start_offset)/2) + 1) AS statement_text
FROM sys.dm_exec_query_memory_grants mg
LEFT JOIN sys.dm_exec_sessions s ON mg.session_id = s.session_id
LEFT JOIN sys.dm_exec_requests r ON mg.session_id = r.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE mg.granted_memory_kb > 10240  -- More than 10MB granted
ORDER BY mg.granted_memory_kb DESC;
```

**I/O Performance Monitoring**:
```sql
-- I/O latency analysis by database file
SELECT 
    DB_NAME(mf.database_id) AS database_name,
    mf.name AS file_name,
    mf.physical_name,
    vfs.io_stall_read_ms / NULLIF(vfs.num_of_reads, 0) AS avg_read_latency_ms,
    vfs.io_stall_write_ms / NULLIF(vfs.num_of_writes, 0) AS avg_write_latency_ms,
    vfs.io_stall / NULLIF(vfs.num_of_reads + vfs.num_of_writes, 0) AS avg_io_latency_ms,
    vfs.num_of_reads,
    vfs.num_of_writes,
    vfs.num_of_bytes_read / 1024/1024 AS mb_read,
    vfs.num_of_bytes_written / 1024/1024 AS mb_written,
    vfs.io_stall_read_ms,
    vfs.io_stall_write_ms,
    vfs.io_stall,
    CASE 
        WHEN vfs.io_stall / NULLIF(vfs.num_of_reads + vfs.num_of_writes, 0) > 50 THEN 'High I/O Latency'
        WHEN vfs.io_stall / NULLIF(vfs.num_of_reads + vfs.num_of_writes, 0) > 20 THEN 'Medium I/O Latency'
        ELSE 'Good I/O Performance'
    END AS io_performance_status
FROM sys.master_files mf
INNER JOIN sys.dm_io_virtual_file_stats(NULL, NULL) vfs ON mf.database_id = vfs.database_id AND mf.file_id = vfs.file_id
WHERE vfs.io_stall / NULLIF(vfs.num_of_reads + vfs.num_of_writes, 0) > 10  -- More than 10ms average latency
ORDER BY avg_io_latency_ms DESC;

-- I/O queue depth analysis
SELECT 
    database_id,
    file_id,
    io_stall_read_ms / NULLIF(num_of_reads, 0) AS avg_read_latency,
    io_stall_write_ms / NULLIF(num_of_writes, 0) AS avg_write_latency,
    io_stall / (num_of_reads + num_of_writes) AS avg_io_latency,
    num_of_reads,
    num_of_writes,
    size_on_disk_bytes / 1024/1024 AS file_size_mb,
    CASE 
        WHEN io_stall / (num_of_reads + num_of_writes) > 50 THEN 'Critical I/O'
        WHEN io_stall / (num_of_reads + num_of_writes) > 20 THEN 'High I/O'
        WHEN io_stall / (num_of_reads + num_of_writes) > 10 THEN 'Medium I/O'
        ELSE 'Good I/O'
    END AS io_status
FROM sys.dm_io_virtual_file_stats(NULL, NULL)
WHERE num_of_reads + num_of_writes > 1000  -- Active files only
ORDER BY avg_io_latency DESC;

-- TempDB usage analysis
SELECT 
    session_id,
    user_objects_alloc_page_count / 128.0 AS user_objects_alloc_mb,
    user_objects_dealloc_page_count / 128.0 AS user_objects_dealloc_mb,
    internal_objects_alloc_page_count / 128.0 AS internal_objects_alloc_mb,
    internal_objects_dealloc_page_count / 128.0 AS internal_objects_dealloc_mb,
    total_allocated_page_count / 128.0 AS total_allocated_mb,
    s.login_name,
    s.host_name,
    s.program_name
FROM sys.dm_db_task_space_usage tsu
INNER JOIN sys.dm_exec_sessions s ON tsu.session_id = s.session_id
WHERE total_allocated_page_count > 0
AND s.is_user_process = 1
ORDER BY total_allocated_page_count DESC;
```

#### 3.2 Blocking and Deadlock Analysis

**Blocking Detection**:
```sql
-- Current blocking analysis
SELECT 
    blocked.session_id AS blocked_session_id,
    blocked.login_name AS blocked_user,
    blocked.host_name AS blocked_host,
    blocking.session_id AS blocking_session_id,
    blocking.login_name AS blocking_user,
    blocking.host_name AS blocking_host,
    blocked.total_elapsed_time / 1000.0 AS blocked_duration_seconds,
    blocking.total_elapsed_time / 1000.0 AS blocking_duration_seconds,
    blocked.last_request_start_time AS blocked_start_time,
    blocking.last_request_start_time AS blocking_start_time,
    CASE blocked.wait_type
        WHEN 'LCK_M_U' THEN 'Update lock wait'
        WHEN 'LCK_M_S' THEN 'Shared lock wait'
        WHEN 'LCK_M_X' THEN 'Exclusive lock wait'
        WHEN 'LCK_M_IS' THEN 'Intent shared lock wait'
        WHEN 'LCK_M_IX' THEN 'Intent exclusive lock wait'
        WHEN 'LCK_M_SIX' THEN 'Shared intent exclusive lock wait'
        ELSE blocked.wait_type
    END AS wait_type,
    blocked.wait_time / 1000.0 AS wait_duration_seconds,
    SUBSTRING(blocked.statement_text, 1, 200) AS blocked_statement,
    SUBSTRING(blocking.statement_text, 1, 200) AS blocking_statement,
    CASE 
        WHEN blocked.wait_time > 30000 THEN 'Critical Blocking'
        WHEN blocked.wait_time > 10000 THEN 'High Blocking'
        WHEN blocked.wait_time > 5000 THEN 'Medium Blocking'
        ELSE 'Low Blocking'
    END AS blocking_severity
FROM sys.dm_exec_requests blocked
INNER JOIN sys.dm_exec_requests blocking ON blocked.blocking_session_id = blocking.session_id
OUTER APPLY (
    SELECT SUBSTRING(st.text, (r.statement_start_offset/2)+1,
        ((CASE r.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE r.statement_end_offset
        END - r.statement_start_offset)/2) + 1) AS statement_text
    FROM sys.dm_exec_sql_text(r.sql_handle) st
) blocked
OUTER APPLY (
    SELECT SUBSTRING(st.text, (r.statement_start_offset/2)+1,
        ((CASE r.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE r.statement_end_offset
        END - r.statement_start_offset)/2) + 1) AS statement_text
    FROM sys.dm_exec_sql_text(r.sql_handle) st
) blocking
WHERE blocked.session_id <> blocking.session_id
ORDER BY blocked.wait_time DESC;

-- Lock information by object
SELECT 
    OBJECT_SCHEMA_NAME(l.resource_associated_entity_id) AS schema_name,
    OBJECT_NAME(l.resource_associated_entity_id) AS table_name,
    l.resource_type,
    l.request_mode,
    l.request_status,
    l.request_session_id,
    s.login_name AS requesting_user,
    s.host_name AS requesting_host,
    l.lock_owner_address,
    CASE l.resource_type
        WHEN 'OBJECT' THEN 'Table lock'
        WHEN 'PAGE' THEN 'Page lock'
        WHEN 'KEY' THEN 'Row lock'
        WHEN 'RID' THEN 'Row identifier lock'
        WHEN 'EXTENT' THEN 'Extent lock'
        WHEN 'DATABASE' THEN 'Database lock'
        ELSE l.resource_type
    END AS lock_description
FROM sys.dm_tran_locks l
LEFT JOIN sys.dm_exec_sessions s ON l.request_session_id = s.session_id
WHERE l.resource_associated_entity_id > 0
AND s.is_user_process = 1
ORDER BY l.request_session_id, l.resource_type;
```

**Deadlock Analysis**:
```sql
-- Recent deadlock information
SELECT 
    deadlocks.deadlock_graph.value('(deadlock/victim-list/victim/@id)[1]', 'varchar(100)') AS victim_id,
    deadlocks.deadlock_graph.value('(deadlock/process-list/process/@spid)[1]', 'int') AS victim_spid,
    deadlocks.deadlock_graph.value('(deadlock/process-list/process/@loginname)[1]', 'varchar(100)') AS victim_login,
    deadlocks.deadlock_graph.value('(deadlock/process-list/process/@hostname)[1]', 'varchar(100)') AS victim_host,
    deadlocks.deadlock_graph.value('(deadlock/process-list/process/@appname)[1]', 'varchar(100)') AS victim_app,
    deadlocks.deadlock_graph.value('(deadlock/process-list/process/inputbuf)[1]', 'varchar(max)') AS victim_statement,
    deadlocks.deadlock_graph.value('(deadlock/process-list/process/@spid)[2]', 'int') AS other_spid,
    deadlocks.deadlock_graph.value('(deadlock/process-list/process/@loginname)[2]', 'varchar(100)') AS other_login,
    deadlocks.deadlock_graph.value('(deadlock/process-list/process/@hostname)[2]', 'varchar(100)') AS other_host,
    deadlocks.deadlock_graph.value('(deadlock/process-list/process/@appname)[2]', 'varchar(100)') AS other_app,
    deadlocks.deadlock_graph.value('(deadlock/process-list/process/inputbuf)[2]', 'varchar(max)') AS other_statement,
    deadlocks.deadlock_graph.value('@timestamp', 'datetime') AS deadlock_time
FROM (SELECT CAST(target_data AS XML) AS target_data 
      FROM sys.dm_xe_session_targets 
      WHERE target_name = 'ring_buffer' 
      AND event_session_address = (SELECT address FROM sys.dm_xe_sessions WHERE name = 'system_health')) AS x
CROSS APPLY x.target_data.nodes('//event[@name=''xml_deadlock_report'']') AS deadlocks(deadlock_graph)
ORDER BY deadlocks.deadlock_graph.value('@timestamp', 'datetime') DESC;

-- Extended events for deadlock monitoring
CREATE EVENT SESSION [DeadlockMonitoring] ON SERVER
ADD EVENT sqlserver.xml_deadlock_report(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.client_pid,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.query_hash,
        sqlserver.query_plan_hash,
        sqlserver.session_id,
        sqlserver.sql_text,
        sqlserver.username
    )
)
ADD TARGET package0.ring_buffer(
    SET max_memory = 4096
)
WITH (
    MAX_MEMORY = 4096 KB,
    EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY = 30 SECONDS,
    MAX_EVENT_SIZE = 0 KB,
    MEMORY_PARTITION_MODE = NONE,
    TRACK_CAUSALITY = ON,
    STARTUP_STATE = ON
);

-- Start the deadlock monitoring session
ALTER EVENT SESSION [DeadlockMonitoring] ON SERVER STATE = START;
```

### 4. Extended Events and Advanced Monitoring

#### 4.1 Extended Events Setup and Configuration

**Basic Extended Events Session**:
```sql
-- Create comprehensive monitoring session
CREATE EVENT SESSION [ComprehensiveMonitoring] ON SERVER
ADD EVENT sqlserver.sql_statement_completed(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.client_pid,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.query_hash,
        sqlserver.query_plan_hash,
        sqlserver.request_id,
        sqlserver.server_principal_name,
        sqlserver.session_id,
        sqlserver.session_nt_username,
        sqlserver.sql_text,
        sqlserver.username
    )
    WHERE (
        -- Filter for long-running queries (> 5 seconds)
        duration > 5000000 OR
        -- Filter for queries with high CPU time (> 2 seconds)
        cpu_time > 2000000 OR
        -- Filter for queries with high logical reads (> 10000 pages)
        logical_reads > 10000
    )
),
ADD EVENT sqlserver.rpc_completed(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.client_pid,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.request_id,
        sqlserver.server_principal_name,
        sqlserver.session_id,
        sqlserver.sql_text,
        sqlserver.username
    )
    WHERE (
        duration > 5000000 OR
        cpu_time > 2000000 OR
        logical_reads > 10000
    )
),
ADD EVENT sqlserver.lock_deadlock(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.client_pid,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.session_id,
        sqlserver.username
    )
),
ADD EVENT sqlserver.lock_timeout(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.client_pid,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.session_id,
        sqlserver.username
    )
)
ADD TARGET package0.event_file(
    SET filename = N'C:\ExtendedEvents\ComprehensiveMonitoring',
    max_file_size = (100),
    max_rollover_files = (10)
),
ADD TARGET package0.ring_buffer(
    SET max_memory = (4096)
)
WITH (
    MAX_MEMORY = 8192 KB,
    EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY = 30 SECONDS,
    MAX_EVENT_SIZE = 0 KB,
    MEMORY_PARTITION_MODE = NONE,
    TRACK_CAUSALITY = ON,
    STARTUP_STATE = ON
);

-- Start the monitoring session
ALTER EVENT SESSION [ComprehensiveMonitoring] ON SERVER STATE = START;

-- Query event data
SELECT 
    event_data.value('(@timestamp)[1]', 'datetime2') AS event_time,
    event_data.value('(@name)[1]', 'varchar(50)') AS event_name,
    event_data.value('(action[@name=''sqlserver.session_id'']/value)[1]', 'int') AS session_id,
    event_data.value('(action[@name=''sqlserver.database_name'']/value)[1]', 'varchar(128)') AS database_name,
    event_data.value('(action[@name=''sqlserver.client_app_name'']/value)[1]', 'varchar(128)') AS client_app,
    event_data.value('(action[@name=''sqlserver.username'']/value)[1]', 'varchar(128)') AS username,
    event_data.value('(action[@name=''sqlserver.sql_text'']/value)[1]', 'nvarchar(max)') AS sql_text,
    event_data.value('(data[@name=''cpu_time'']/value)[1]', 'bigint') AS cpu_time,
    event_data.value('(data[@name=''duration'']/value)[1]', 'bigint') AS duration,
    event_data.value('(data[@name=''logical_reads'']/value)[1]', 'bigint') AS logical_reads,
    event_data.value('(data[@name=''physical_reads'']/value)[1]', 'bigint') AS physical_reads,
    event_data.value('(data[@name=''writes'']/value)[1]', 'bigint') AS writes
FROM (SELECT CAST(event_data AS XML) AS event_data
      FROM sys.fn_xe_file_target_read_file('C:\ExtendedEvents\ComprehensiveMonitoring*.xel', NULL, NULL, NULL)) AS event_file
WHERE event_data.value('(@timestamp)[1]', 'datetime2') > DATEADD(HOUR, -1, GETDATE())
ORDER BY event_data.value('(@timestamp)[1]', 'datetime2') DESC;
```

**Performance Monitoring with Extended Events**:
```sql
-- Performance Dashboard monitoring session
CREATE EVENT SESSION [PerformanceDashboard] ON SERVER
ADD EVENT sqlserver.sql_batch_completed(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.client_pid,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.query_hash,
        sqlserver.query_plan_hash,
        sqlserver.request_id,
        sqlserver.session_id,
        sqlserver.sql_text
    )
),
ADD EVENT sqlserver.sql_statement_recompile(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.query_hash,
        sqlserver.session_id,
        sqlserver.sql_text
    )
    WHERE (
        -- Recompile events with causes
        recompile_cause > 1
    )
),
ADD EVENT sqlserver.plan_cache_eviction(
    ACTION(
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.session_id
    )
),
ADD EVENT sqlserver.background_job_error(
    ACTION(
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.session_id
    )
)
ADD TARGET package0.event_file(
    SET filename = N'C:\ExtendedEvents\PerformanceDashboard',
    max_file_size = (50),
    max_rollover_files = (5)
)
WITH (
    MAX_MEMORY = 2048 KB,
    EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY = 60 SECONDS,
    MAX_EVENT_SIZE = 0 KB,
    MEMORY_PARTITION_MODE = NONE,
    TRACK_CAUSALITY = OFF,
    STARTUP_STATE = OFF
);

-- Start performance dashboard session
ALTER EVENT SESSION [PerformanceDashboard] ON SERVER STATE = START;
```

#### 4.2 Automated Monitoring and Alerting

**Performance Alerting System**:
```sql
-- Create monitoring alerts table
CREATE TABLE dbo.PerformanceAlerts (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    AlertType NVARCHAR(50) NOT NULL,
    Severity NVARCHAR(20) NOT NULL, -- CRITICAL, HIGH, MEDIUM, LOW
    DatabaseName NVARCHAR(128),
    TableName NVARCHAR(128),
    MetricName NVARCHAR(100),
    ThresholdValue DECIMAL(18,2),
    CurrentValue DECIMAL(18,2),
    DurationMinutes INT,
    Description NVARCHAR(MAX),
    Recommendations NVARCHAR(MAX),
    Status NVARCHAR(20) NOT NULL DEFAULT 'OPEN', -- OPEN, ACKNOWLEDGED, RESOLVED, CLOSED
    AssignedTo NVARCHAR(128),
    ResolvedDate DATETIME2,
    ResolutionNotes NVARCHAR(MAX)
);

-- Performance monitoring procedure
CREATE PROCEDURE sp_PerformanceMonitoring
    @AlertThresholds TABLE (
        MetricName NVARCHAR(100),
        WarningThreshold DECIMAL(18,2),
        CriticalThreshold DECIMAL(18,2),
        MetricType NVARCHAR(20), -- HIGHER_IS_WORSE, LOWER_IS_WORSE
        DurationMinutes INT DEFAULT 5
    )
AS
BEGIN
    SET NOCOUNT ON;
    
    -- CPU usage monitoring
    IF EXISTS (
        SELECT 1 FROM @AlertThresholds 
        WHERE MetricName = 'CPU_Usage' AND MetricType = 'HIGHER_IS_WORSE'
    )
    BEGIN
        DECLARE @CPUUsage DECIMAL(5,2);
        
        SELECT @CPUUsage = AVG(CASE WHEN SQLProcessUtilization + (100 - SystemIdle - SQLProcessUtilization) > 90 THEN 1 ELSE 0 END) * 100
        FROM (
            SELECT 
                record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
                record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SQLProcessUtilization)[1]', 'int') AS SQLProcessUtilization
            FROM (
                SELECT TOP 10 CAST(record AS XML) AS record
                FROM sys.dm_os_ring_buffers
                WHERE ring_buffer_type = 'RING_BUFFER_SCHEDULER_MONITOR'
                ORDER BY timestamp DESC
            ) AS x
            CROSS APPLY x.record.nodes('/Record') AS r(record)
        ) AS y;
        
        -- Check CPU thresholds
        IF @CPUUsage > (SELECT CriticalThreshold FROM @AlertThresholds WHERE MetricName = 'CPU_Usage')
        BEGIN
            INSERT INTO dbo.PerformanceAlerts (
                AlertType, Severity, MetricName, CurrentValue, Description, Recommendations
            )
            VALUES (
                'CPU_PERFORMANCE', 'CRITICAL', 'CPU_Usage', @CPUUsage,
                'CPU usage is critically high: ' + CAST(@CPUUsage AS VARCHAR(10)) + '%',
                '1. Identify CPU-intensive queries\n2. Check for missing indexes\n3. Review parallel query execution plans\n4. Consider adding more CPU resources'
            );
        END
        ELSE IF @CPUUsage > (SELECT WarningThreshold FROM @AlertThresholds WHERE MetricName = 'CPU_Usage')
        BEGIN
            INSERT INTO dbo.PerformanceAlerts (
                AlertType, Severity, MetricName, CurrentValue, Description
            )
            VALUES (
                'CPU_PERFORMANCE', 'HIGH', 'CPU_Usage', @CPUUsage,
                'CPU usage is elevated: ' + CAST(@CPUUsage AS VARCHAR(10)) + '%'
            );
        END
    END
    
    -- Page Life Expectancy monitoring
    IF EXISTS (
        SELECT 1 FROM @AlertThresholds 
        WHERE MetricName = 'Page_Life_Expectancy' AND MetricType = 'LOWER_IS_WORSE'
    )
    BEGIN
        DECLARE @PageLifeExpectancy BIGINT;
        
        SELECT @PageLifeExpectancy = cntr_value
        FROM sys.dm_os_performance_counters
        WHERE counter_name = 'Page life expectancy'
        AND object_name = 'MSSQL' + CAST(SERVERPROPERTY('InstanceName') AS NVARCHAR(128)) + ':Buffer Manager';
        
        -- Check PLE threshold
        IF @PageLifeExpectancy < (SELECT CriticalThreshold FROM @AlertThresholds WHERE MetricName = 'Page_Life_Expectancy')
        BEGIN
            INSERT INTO dbo.PerformanceAlerts (
                AlertType, Severity, MetricName, CurrentValue, Description, Recommendations
            )
            VALUES (
                'MEMORY_PRESSURE', 'CRITICAL', 'Page_Life_Expectancy', @PageLifeExpectancy,
                'Page Life Expectancy is critically low: ' + CAST(@PageLifeExpectancy AS VARCHAR(10)) + ' seconds',
                '1. Increase SQL Server memory allocation\n2. Check for memory leaks\n3. Optimize queries with high memory grants\n4. Review buffer cache usage patterns'
            );
        END
        ELSE IF @PageLifeExpectancy < (SELECT WarningThreshold FROM @AlertThresholds WHERE MetricName = 'Page_Life_Expectancy')
        BEGIN
            INSERT INTO dbo.PerformanceAlerts (
                AlertType, Severity, MetricName, CurrentValue, Description
            )
            VALUES (
                'MEMORY_PRESSURE', 'HIGH', 'Page_Life_Expectancy', @PageLifeExpectancy,
                'Page Life Expectancy is below normal: ' + CAST(@PageLifeExpectancy AS VARCHAR(10)) + ' seconds'
            );
        END
    END
    
    -- Blocking monitoring
    IF EXISTS (
        SELECT 1 FROM @AlertThresholds WHERE MetricName = 'Blocking_Sessions'
    )
    BEGIN
        DECLARE @BlockedSessionsCount INT;
        
        SELECT @BlockedSessionsCount = COUNT(*)
        FROM sys.dm_exec_requests
        WHERE blocking_session_id > 0;
        
        IF @BlockedSessionsCount > 0
        BEGIN
            DECLARE @Severity NVARCHAR(20) = CASE 
                WHEN @BlockedSessionsCount > 10 THEN 'CRITICAL'
                WHEN @BlockedSessionsCount > 5 THEN 'HIGH'
                ELSE 'MEDIUM'
            END;
            
            INSERT INTO dbo.PerformanceAlerts (
                AlertType, Severity, MetricName, CurrentValue, Description, Recommendations
            )
            VALUES (
                'BLOCKING', @Severity, 'Blocking_Sessions', @BlockedSessionsCount,
                'Number of blocked sessions: ' + CAST(@BlockedSessionsCount AS VARCHAR(10)),
                '1. Identify blocking session using sp_who2 or sys.dm_exec_requests\n2. Review blocking query and optimize if needed\n3. Consider implementing query timeout\n4. Review transaction isolation levels'
            );
        END
    END
    
    -- Return recent alerts
    SELECT TOP 20 
        AlertID,
        AlertDate,
        AlertType,
        Severity,
        MetricName,
        CurrentValue,
        Description,
        Status,
        AssignedTo
    FROM dbo.PerformanceAlerts
    WHERE AlertDate > DATEADD(HOUR, -24, GETDATE())
    ORDER BY AlertDate DESC;
END;

-- Create alert thresholds
DECLARE @AlertThresholds TABLE (
    MetricName NVARCHAR(100),
    WarningThreshold DECIMAL(18,2),
    CriticalThreshold DECIMAL(18,2),
    MetricType NVARCHAR(20),
    DurationMinutes INT
);

INSERT INTO @AlertThresholds (MetricName, WarningThreshold, CriticalThreshold, MetricType, DurationMinutes)
VALUES 
    ('CPU_Usage', 70, 90, 'HIGHER_IS_WORSE', 5),
    ('Page_Life_Expectancy', 300, 150, 'LOWER_IS_WORSE', 10),
    ('Blocking_Sessions', 3, 10, 'HIGHER_IS_WORSE', 2);

-- Execute monitoring
EXEC sp_PerformanceMonitoring @AlertThresholds;
```

**Proactive Maintenance Alerts**:
```sql
-- Index fragmentation alerts
CREATE PROCEDURE sp_IndexFragmentationAlert
    @DatabaseName NVARCHAR(128) = NULL,
    @FragmentationThreshold DECIMAL(5,2) = 30.0
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX);
    
    -- Check fragmentation for specified database or all user databases
    SET @SQL = '
    SELECT 
        ''' + ISNULL(@DatabaseName, 'ALL_DATABASES') + ''' AS database_checked,
        OBJECT_SCHEMA_NAME(ips.object_id) AS schema_name,
        OBJECT_NAME(ips.object_id) AS table_name,
        i.name AS index_name,
        ips.avg_fragmentation_in_percent,
        ips.page_count,
        CASE 
            WHEN ips.avg_fragmentation_in_percent > ' + CAST(@FragmentationThreshold AS VARCHAR(10)) + ' THEN ''HIGH_FRAGMENTATION''
            WHEN ips.avg_fragmentation_in_percent > 10 THEN ''MODERATE_FRAGMENTATION''
            ELSE ''LOW_FRAGMENTATION''
        END AS fragmentation_status,
        ''ALTER INDEX ' + i.name + ' ON ' + OBJECT_SCHEMA_NAME(ips.object_id) + '.' + OBJECT_NAME(ips.object_id) + ' REBUILD WITH (ONLINE = ON, FILLFACTOR = 90);'' AS rebuild_command
    FROM sys.dm_db_index_physical_stats(DB_ID(' + ISNULL('''' + @DatabaseName + '''', 'NULL') + '), NULL, NULL, NULL, ''DETAILED'') ips
    INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE ips.avg_fragmentation_in_percent > ' + CAST(@FragmentationThreshold AS VARCHAR(10)) + '
    AND ips.page_count > 100
    AND i.name NOT LIKE ''PK_%''
    ORDER BY ips.avg_fragmentation_in_percent DESC, ips.page_count DESC';
    
    EXEC sp_executesql @SQL;
    
    -- Create alert if highly fragmented indexes found
    IF EXISTS (
        SELECT 1 
        FROM sys.dm_db_index_physical_stats(DB_ID(@DatabaseName), NULL, NULL, NULL, 'DETAILED') ips
        INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
        WHERE ips.avg_fragmentation_in_percent > @FragmentationThreshold
        AND ips.page_count > 100
        AND i.name NOT LIKE 'PK_%'
    )
    BEGIN
        INSERT INTO dbo.PerformanceAlerts (
            AlertType, Severity, DatabaseName, MetricName, CurrentValue, Description, Recommendations
        )
        SELECT 
            'INDEX_FRAGMENTATION', 
            CASE WHEN AVG(ips.avg_fragmentation_in_percent) > 50 THEN 'HIGH' ELSE 'MEDIUM' END,
            @DatabaseName,
            'Index_Fragmentation',
            AVG(ips.avg_fragmentation_in_percent),
            'Average index fragmentation: ' + CAST(AVG(ips.avg_fragmentation_in_percent) AS VARCHAR(10)) + '%',
            '1. Execute index maintenance procedures\n2. Review index usage statistics\n3. Consider dropping unused indexes\n4. Update table statistics'
        FROM sys.dm_db_index_physical_stats(DB_ID(@DatabaseName), NULL, NULL, NULL, 'DETAILED') ips
        INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
        WHERE ips.avg_fragmentation_in_percent > @FragmentationThreshold
        AND ips.page_count > 100;
    END
END;
```

## Practical Exercises

### Exercise 1: Comprehensive Performance Monitoring Setup

**Objective**: Implement a comprehensive performance monitoring solution for a production SQL Server environment

**Scenario**: Set up monitoring for TechCorp's production database with the following requirements:

**Monitoring Requirements**:
- **Database**: ProductionDB (100GB, 500 concurrent users)
- **Performance Targets**: Page Life Expectancy > 300 seconds, CPU usage < 80%
- **Alerting**: Immediate alerts for critical performance issues
- **Historical Data**: 30 days of performance metrics
- **Reporting**: Daily performance reports

**Tasks**:

1. **Create Performance Monitoring Infrastructure**
```sql
-- Create performance monitoring schema
CREATE SCHEMA PerformanceMonitor AUTHORIZATION dbo;

-- Create performance metrics collection tables
CREATE TABLE PerformanceMonitor.ServerMetrics (
    MetricID BIGINT IDENTITY(1,1) PRIMARY KEY,
    CollectionDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    MetricName NVARCHAR(100) NOT NULL,
    MetricValue DECIMAL(18,4) NOT NULL,
    MetricUnit NVARCHAR(20),
    Severity NVARCHAR(20),
    AdditionalData NVARCHAR(MAX)
);

CREATE TABLE PerformanceMonitor.QueryMetrics (
    QueryID BIGINT IDENTITY(1,1) PRIMARY KEY,
    CollectionDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    QueryHash BINARY(8),
    QueryText NVARCHAR(MAX),
    ExecutionCount BIGINT,
    TotalDurationMS BIGINT,
    AvgDurationMS DECIMAL(18,4),
    TotalCPUMS BIGINT,
    AvgCPUMS DECIMAL(18,4),
    TotalLogicalReads BIGINT,
    AvgLogicalReads DECIMAL(18,4),
    DatabaseName NVARCHAR(128),
    LastExecutionDate DATETIME2
);

CREATE TABLE PerformanceMonitor.IndexMetrics (
    IndexID BIGINT IDENTITY(1,1) PRIMARY KEY,
    CollectionDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    TableSchema NVARCHAR(128),
    TableName NVARCHAR(128),
    IndexName NVARCHAR(128),
    IndexType NVARCHAR(50),
    FragmentationPercent DECIMAL(5,2),
    PageCount BIGINT,
    RecordCount BIGINT,
    ActionRequired NVARCHAR(100)
);

-- Create indexes for performance monitoring tables
CREATE INDEX IX_ServerMetrics_DateName ON PerformanceMonitor.ServerMetrics(CollectionDate, MetricName);
CREATE INDEX IX_QueryMetrics_Date ON PerformanceMonitor.QueryMetrics(CollectionDate);
CREATE INDEX IX_QueryMetrics_Hash ON PerformanceMonitor.QueryMetrics(QueryHash);
CREATE INDEX IX_IndexMetrics_TableDate ON PerformanceMonitor.IndexMetrics(TableName, CollectionDate);
```

2. **Implement Data Collection Procedures**
```sql
-- Server metrics collection procedure
CREATE PROCEDURE PerformanceMonitor.sp_CollectServerMetrics
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CollectionTime DATETIME2 = GETDATE();
    
    -- CPU utilization
    INSERT INTO PerformanceMonitor.ServerMetrics (MetricName, MetricValue, MetricUnit, Severity)
    SELECT 
        'CPU_Usage_Percent',
        CAST(AVG(CASE WHEN SQLProcessUtilization + (100 - SystemIdle - SQLProcessUtilization) > 90 THEN 1 ELSE 0 END) * 100 AS DECIMAL(5,2)),
        'PERCENT',
        CASE 
            WHEN AVG(CASE WHEN SQLProcessUtilization + (100 - SystemIdle - SQLProcessUtilization) > 90 THEN 1 ELSE 0 END) > 0.9 THEN 'CRITICAL'
            WHEN AVG(CASE WHEN SQLProcessUtilization + (100 - SystemIdle - SQLProcessUtilization) > 90 THEN 1 ELSE 0 END) > 0.7 THEN 'HIGH'
            ELSE 'NORMAL'
        END
    FROM (
        SELECT 
            record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
            record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SQLProcessUtilization)[1]', 'int') AS SQLProcessUtilization
        FROM (
            SELECT TOP 10 CAST(record AS XML) AS record
            FROM sys.dm_os_ring_buffers
            WHERE ring_buffer_type = 'RING_BUFFER_SCHEDULER_MONITOR'
            ORDER BY timestamp DESC
        ) AS x
        CROSS APPLY x.record.nodes('/Record') AS r(record)
    ) AS y;
    
    -- Memory metrics
    INSERT INTO PerformanceMonitor.ServerMetrics (MetricName, MetricValue, MetricUnit, Severity)
    SELECT counter_name,
           CAST(cntr_value AS DECIMAL(18,4)),
           CASE counter_name
               WHEN 'Page life expectancy' THEN 'SECONDS'
               WHEN 'Buffer cache hit ratio' THEN 'PERCENT'
               ELSE 'COUNT'
           END,
           CASE counter_name
               WHEN 'Page life expectancy' THEN 
                   CASE WHEN CAST(cntr_value AS DECIMAL(18,4)) < 150 THEN 'CRITICAL'
                        WHEN CAST(cntr_value AS DECIMAL(18,4)) < 300 THEN 'HIGH'
                        ELSE 'NORMAL'
                   END
               WHEN 'Buffer cache hit ratio' THEN
                   CASE WHEN CAST(cntr_value AS DECIMAL(18,4)) < 90 THEN 'CRITICAL'
                        WHEN CAST(cntr_value AS DECIMAL(18,4)) < 95 THEN 'HIGH'
                        ELSE 'NORMAL'
                   END
               ELSE 'NORMAL'
           END
    FROM sys.dm_os_performance_counters
    WHERE counter_name IN ('Page life expectancy', 'Buffer cache hit ratio')
    AND object_name LIKE '%Buffer Manager%';
    
    -- Connection metrics
    INSERT INTO PerformanceMonitor.ServerMetrics (MetricName, MetricValue, MetricUnit, Severity)
    SELECT 
        'Active_Connections',
        COUNT(*),
        'COUNT',
        CASE WHEN COUNT(*) > 400 THEN 'HIGH' ELSE 'NORMAL' END
    FROM sys.dm_exec_sessions
    WHERE is_user_process = 1;
    
    -- Transaction log metrics
    INSERT INTO PerformanceMonitor.ServerMetrics (MetricName, MetricValue, MetricUnit, Severity)
    SELECT 
        'Log_File_Usage_Percent',
        CAST((CAST(FILEPROPERTY(name, 'SpaceUsed') AS FLOAT) / size * 100) AS DECIMAL(5,2)),
        'PERCENT',
        CASE 
            WHEN (CAST(FILEPROPERTY(name, 'SpaceUsed') AS FLOAT) / size * 100) > 90 THEN 'CRITICAL'
            WHEN (CAST(FILEPROPERTY(name, 'SpaceUsed') AS FLOAT) / size * 100) > 80 THEN 'HIGH'
            ELSE 'NORMAL'
        END
    FROM sys.database_files
    WHERE type_desc = 'LOG';
    
    -- I/O metrics
    INSERT INTO PerformanceMonitor.ServerMetrics (MetricName, MetricValue, MetricUnit, Severity)
    SELECT 
        'Avg_I/O_Latency_MS',
        CAST(io_stall / NULLIF(num_of_reads + num_of_writes, 0) AS DECIMAL(10,2)),
        'MILLISECONDS',
        CASE 
            WHEN io_stall / NULLIF(num_of_reads + num_of_writes, 0) > 50 THEN 'CRITICAL'
            WHEN io_stall / NULLIF(num_of_reads + num_of_writes, 0) > 20 THEN 'HIGH'
            ELSE 'NORMAL'
        END
    FROM sys.dm_io_virtual_file_stats(NULL, NULL)
    WHERE io_stall / NULLIF(num_of_reads + num_of_writes, 0) > 0
    AND num_of_reads + num_of_writes > 1000;
END;
```

3. **Query Performance Collection**
```sql
-- Query metrics collection procedure
CREATE PROCEDURE PerformanceMonitor.sp_CollectQueryMetrics
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO PerformanceMonitor.QueryMetrics (
        QueryHash, QueryText, ExecutionCount, TotalDurationMS, AvgDurationMS,
        TotalCPUMS, AvgCPUMS, TotalLogicalReads, AvgLogicalReads, 
        DatabaseName, LastExecutionDate
    )
    SELECT 
        qs.query_hash,
        SUBSTRING(st.text, 1, 2000),  -- First 2000 characters
        qs.execution_count,
        qs.total_elapsed_time / 1000,
        qs.total_elapsed_time / NULLIF(qs.execution_count, 0) / 1000,
        qs.total_worker_time / 1000,
        qs.total_worker_time / NULLIF(qs.execution_count, 0) / 1000,
        qs.total_logical_reads,
        qs.total_logical_reads / NULLIF(qs.execution_count, 0),
        DB_NAME(st.dbid),
        qs.last_execution_time
    FROM sys.dm_exec_query_stats qs
    OUTER APPLY sys.dm_exec_sql_text(qs.sql_handle) st
    WHERE qs.execution_count > 10  -- Only collect metrics for frequently executed queries
    AND qs.last_execution_time > DATEADD(HOUR, -1, GETDATE())  -- Only recent queries
    ORDER BY qs.total_elapsed_time DESC;
    
    -- Identify problematic queries
    INSERT INTO PerformanceMonitor.ServerMetrics (MetricName, MetricValue, MetricUnit, Severity, AdditionalData)
    SELECT 
        'Problematic_Queries_Count',
        COUNT(*),
        'COUNT',
        CASE 
            WHEN COUNT(*) > 20 THEN 'CRITICAL'
            WHEN COUNT(*) > 10 THEN 'HIGH'
            ELSE 'MEDIUM'
        END,
        'Queries with avg duration > 5 seconds and execution count > 10'
    FROM PerformanceMonitor.QueryMetrics
    WHERE AvgDurationMS > 5000
    AND ExecutionCount > 10
    AND CollectionDate = CAST(GETDATE() AS DATE);
END;
```

4. **Index Metrics Collection**
```sql
-- Index metrics collection procedure
CREATE PROCEDURE PerformanceMonitor.sp_CollectIndexMetrics
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO PerformanceMonitor.IndexMetrics (
        TableSchema, TableName, IndexName, IndexType, FragmentationPercent,
        PageCount, RecordCount, ActionRequired
    )
    SELECT 
        OBJECT_SCHEMA_NAME(ips.object_id),
        OBJECT_NAME(ips.object_id),
        i.name,
        i.type_desc,
        ips.avg_fragmentation_in_percent,
        ips.page_count,
        ips.record_count,
        CASE 
            WHEN ips.avg_fragmentation_in_percent > 30 THEN 'REBUILD'
            WHEN ips.avg_fragmentation_in_percent > 10 THEN 'REORGANIZE'
            ELSE 'MONITOR'
        END
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
    INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE ips.avg_fragmentation_in_percent > 5
    AND ips.page_count > 100
    ORDER BY ips.avg_fragmentation_in_percent DESC;
    
    -- Create alert for highly fragmented indexes
    IF EXISTS (
        SELECT 1 FROM PerformanceMonitor.IndexMetrics 
        WHERE FragmentationPercent > 30 
        AND CollectionDate = CAST(GETDATE() AS DATE)
    )
    BEGIN
        INSERT INTO PerformanceMonitor.ServerMetrics (MetricName, MetricValue, MetricUnit, Severity)
        SELECT 
            'Highly_Fragmented_Indexes_Count',
            COUNT(*),
            'COUNT',
            'HIGH'
        FROM PerformanceMonitor.IndexMetrics
        WHERE FragmentationPercent > 30
        AND CollectionDate = CAST(GETDATE() AS DATE);
    END
END;
```

5. **Performance Alerting**
```sql
-- Performance alerting procedure
CREATE PROCEDURE PerformanceMonitor.sp_PerformanceAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentTime DATETIME2 = GETDATE();
    
    -- CPU usage alerts
    IF EXISTS (
        SELECT 1 FROM PerformanceMonitor.ServerMetrics
        WHERE MetricName = 'CPU_Usage_Percent'
        AND CollectionDate >= DATEADD(MINUTE, -5, @CurrentTime)
        AND MetricValue > 80
    )
    BEGIN
        INSERT INTO dbo.PerformanceAlerts (
            AlertType, Severity, MetricName, CurrentValue, Description, Recommendations
        )
        SELECT 
            'CPU_PERFORMANCE',
            'HIGH',
            'CPU_Usage',
            AVG(MetricValue),
            'Average CPU usage over last 5 minutes: ' + CAST(AVG(MetricValue) AS VARCHAR(10)) + '%',
            '1. Identify CPU-intensive queries from QueryMetrics\n2. Review execution plans for missing indexes\n3. Check for parameter sniffing issues\n4. Consider query optimization'
        FROM PerformanceMonitor.ServerMetrics
        WHERE MetricName = 'CPU_Usage_Percent'
        AND CollectionDate >= DATEADD(MINUTE, -5, @CurrentTime);
    END
    
    -- Memory pressure alerts
    IF EXISTS (
        SELECT 1 FROM PerformanceMonitor.ServerMetrics
        WHERE MetricName = 'Page life expectancy'
        AND CollectionDate >= DATEADD(MINUTE, -5, @CurrentTime)
        AND MetricValue < 150
    )
    BEGIN
        INSERT INTO dbo.PerformanceAlerts (
            AlertType, Severity, MetricName, CurrentValue, Description, Recommendations
        )
        SELECT 
            'MEMORY_PRESSURE',
            'CRITICAL',
            'Page_Life_Expectancy',
            AVG(MetricValue),
            'Page Life Expectancy critically low: ' + CAST(AVG(MetricValue) AS VARCHAR(10)) + ' seconds',
            '1. Increase SQL Server memory allocation\n2. Check for memory-intensive queries\n3. Review buffer cache hit ratio\n4. Consider adding more RAM to server'
        FROM PerformanceMonitor.ServerMetrics
        WHERE MetricName = 'Page life expectancy'
        AND CollectionDate >= DATEADD(MINUTE, -5, @CurrentTime);
    END
    
    -- Query performance alerts
    INSERT INTO dbo.PerformanceAlerts (
        AlertType, Severity, MetricName, CurrentValue, Description, Recommendations, DatabaseName
    )
    SELECT TOP 5
        'QUERY_PERFORMANCE',
        CASE WHEN AvgDurationMS > 10000 THEN 'CRITICAL'
             WHEN AvgDurationMS > 5000 THEN 'HIGH'
             ELSE 'MEDIUM'
        END AS Severity,
        'Slow_Query',
        AvgDurationMS,
        'Slow query average duration: ' + CAST(AvgDurationMS AS VARCHAR(10)) + ' ms (Executions: ' + CAST(ExecutionCount AS VARCHAR(10)) + ')',
        '1. Review query execution plan\n2. Check for missing indexes\n3. Consider query rewriting\n4. Review statistics update status',
        DatabaseName
    FROM PerformanceMonitor.QueryMetrics
    WHERE CollectionDate >= DATEADD(HOUR, -1, @CurrentTime)
    AND AvgDurationMS > 1000
    AND ExecutionCount > 10
    ORDER BY AvgDurationMS DESC;
END;
```

6. **Automated Monitoring Job**
```sql
-- Create SQL Server Agent job for automated monitoring
USE msdb;
EXEC dbo.sp_add_job
    @job_name = N'Performance Monitoring',
    @enabled = 1,
    @description = N'Automated performance monitoring and alerting';

EXEC sp_add_jobstep
    @job_name = N'Performance Monitoring',
    @step_name = N'Collect Server Metrics',
    @command = N'EXEC PerformanceMonitor.sp_CollectServerMetrics;';

EXEC sp_add_jobstep
    @job_name = N'Performance Monitoring',
    @step_name = N'Collect Query Metrics',
    @command = N'EXEC PerformanceMonitor.sp_CollectQueryMetrics;';

EXEC sp_add_jobstep
    @job_name = N'Performance Monitoring',
    @step_name = N'Collect Index Metrics',
    @command = N'EXEC PerformanceMonitor.sp_CollectIndexMetrics;';

EXEC sp_add_jobstep
    @job_name = N'Performance Monitoring',
    @step_name = N'Process Performance Alerts',
    @command = N'EXEC PerformanceMonitor.sp_PerformanceAlerts;';

EXEC sp_add_schedule
    @schedule_name = N'Every 5 Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 5,
    @active_start_date = 20230101,
    @active_end_date = 99991231,
    @active_start_time = 0,
    @active_end_time = 235959;

EXEC sp_attach_schedule
    @job_name = N'Performance Monitoring',
    @schedule_name = N'Every 5 Minutes';

EXEC sp_add_jobserver
    @job_name = N'Performance Monitoring',
    @server_name = N'(LOCAL)';
```

**Deliverable**: Complete performance monitoring system with automated data collection, alerting, and historical tracking

### Exercise 2: Query Performance Optimization

**Objective**: Analyze and optimize slow-running queries to improve database performance

**Scenario**: Identify and optimize the top 10 slowest queries in the production database

**Requirements**:
- **Performance Target**: Reduce average query execution time by 50%
- **Analysis Method**: Execution plans, missing indexes, statistics
- **Optimization**: Index creation, query rewriting, statistics updates
- **Validation**: Before/after performance comparison

**Tasks**:

1. **Identify Slow-Running Queries**
```sql
-- Step 1: Find top 10 slowest queries by average execution time
SELECT TOP 10
    qs.query_hash,
    DB_NAME(st.dbid) AS database_name,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS statement_text,
    qs.execution_count,
    qs.total_elapsed_time / 1000.0 AS total_duration_ms,
    (qs.total_elapsed_time / qs.execution_count) / 1000.0 AS avg_duration_ms,
    qs.total_logical_reads / 1024.0 AS total_logical_reads_kb,
    (qs.total_logical_reads / qs.execution_count) / 1024.0 AS avg_logical_reads_kb,
    qs.total_worker_time / 1000.0 AS total_cpu_time_ms,
    (qs.total_worker_time / qs.execution_count) / 1000.0 AS avg_cpu_time_ms,
    qs.last_execution_time,
    qs.creation_time,
    'Optimize this query to reduce execution time' AS optimization_note
FROM sys.dm_exec_query_stats qs
OUTER APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.execution_count > 10  -- Only analyze frequently executed queries
AND qs.last_execution_time > DATEADD(HOUR, -24, GETDATE())
AND st.dbid = DB_ID()  -- Only current database
ORDER BY (qs.total_elapsed_time / qs.execution_count) DESC;

-- Step 2: Analyze query wait times and blocking
SELECT 
    r.session_id,
    r.status,
    r.command,
    r.cpu_time,
    r.total_elapsed_time / 1000.0 AS duration_ms,
    r.reads,
    r.writes,
    r.logical_reads,
    r.wait_type,
    r.wait_time / 1000.0 AS wait_duration_ms,
    SUBSTRING(st.text, (r.statement_start_offset/2)+1,
        ((CASE r.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE r.statement_end_offset
        END - r.statement_start_offset)/2) + 1) AS statement_text,
    s.login_name,
    s.host_name,
    s.program_name,
    CASE 
        WHEN r.total_elapsed_time / 1000.0 > 5000 THEN 'Critical - > 5 seconds'
        WHEN r.total_elapsed_time / 1000.0 > 1000 THEN 'High - > 1 second'
        WHEN r.total_elapsed_time / 1000.0 > 100 THEN 'Medium - > 100ms'
        ELSE 'Low'
    END AS performance_level
FROM sys.dm_exec_requests r
LEFT JOIN sys.dm_exec_sessions s ON r.session_id = s.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.session_id <> @@SPID
AND s.is_user_process = 1
ORDER BY r.total_elapsed_time DESC;
```

2. **Analyze Execution Plans**
```sql
-- Enable actual execution plan in SSMS and analyze
-- Step 1: Get execution plans for slow queries
SELECT 
    qp.query_plan,
    qs.query_text_hash,
    qs.execution_count,
    qs.avg_elapsed_time / 1000.0 AS avg_duration_ms,
    qs.avg_logical_reads
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE qs.avg_elapsed_time / 1000.0 > 1000  -- Queries taking more than 1 second
AND qs.execution_count > 10
ORDER BY qs.avg_elapsed_time DESC;

-- Step 2: Analyze plan cache for compilation issues
SELECT 
    cp.usecounts,
    cp.cacheobjtype,
    cp.objtype,
    SUBSTRING(st.text, (cp.sid/2)+1,
        CASE WHEN CHARINDEX('(', st.text, cp.sid) > 0
            THEN CHARINDEX('(', st.text, cp.sid) - cp.sid
            ELSE DATALENGTH(st.text) - cp.sid/2
        END) AS query_text,
    cp.plan_handle
FROM sys.dm_exec_cached_plans cp
OUTER APPLY sys.dm_exec_sql_text(cp.plan_handle) st
WHERE cp.usecounts > 1
AND st.text LIKE '%CREATE TABLE%'
OR st.text LIKE '%INSERT INTO%'
OR st.text LIKE '%UPDATE%'
OR st.text LIKE '%DELETE%'
ORDER BY cp.usecounts DESC;

-- Step 3: Identify parameter sniffing issues
SELECT 
    qs.query_hash,
    qs.execution_count,
    qs.min_elapsed_time / 1000.0 AS min_duration_ms,
    qs.max_elapsed_time / 1000.0 AS max_duration_ms,
    (qs.max_elapsed_time - qs.min_elapsed_time) / 1000.0 AS duration_variance_ms,
    (qs.max_elapsed_time / NULLIF(qs.min_elapsed_time, 0)) AS performance_ratio,
    CASE 
        WHEN (qs.max_elapsed_time / NULLIF(qs.min_elapsed_time, 0)) > 10 THEN 'Parameter Sniffing Issue'
        WHEN (qs.max_elapsed_time / NULLIF(qs.min_elapsed_time, 0)) > 5 THEN 'Potential Parameter Sniffing'
        ELSE 'Normal Variation'
    END AS sniffing_analysis
FROM sys.dm_exec_query_stats qs
WHERE qs.execution_count > 20
AND qs.min_elapsed_time > 0
AND (qs.max_elapsed_time / NULLIF(qs.min_elapsed_time, 0)) > 3
ORDER BY (qs.max_elapsed_time / NULLIF(qs.min_elapsed_time, 0)) DESC;
```

3. **Identify and Implement Index Solutions**
```sql
-- Step 1: Comprehensive missing index analysis
SELECT 
    mid.statement AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.avg_user_impact,
    migs.avg_total_user_cost,
    migs.user_seeks,
    migs.user_scans,
    migs.user_lookups,
    migs.last_user_seek,
    'CREATE INDEX IX_' + 
    REPLACE(REPLACE(REPLACE(mid.statement, '[', ''), ']', ''), '.', '_') + '_' + 
    CAST(migs.index_handle AS VARCHAR(10)) + ' ON ' + 
    mid.statement + 
    ' (' + 
    ISNULL(mid.equality_columns, '') + 
    CASE WHEN mid.equality_columns IS NOT NULL AND mid.inequality_columns IS NOT NULL THEN ', ' ELSE '' END +
    ISNULL(mid.inequality_columns, '') + ')' +
    CASE WHEN mid.included_columns IS NOT NULL THEN ' INCLUDE (' + mid.included_columns + ')' ELSE '' END +
    ' WITH (ONLINE = ON, FILLFACTOR = 90);' AS create_index_ddl,
    CASE 
        WHEN migs.avg_user_impact > 70 AND (migs.user_seeks + migs.user_scans) > 100 THEN 'High Priority - Create Index'
        WHEN migs.avg_user_impact > 30 AND (migs.user_seeks + migs.user_scans) > 50 THEN 'Medium Priority - Consider Index'
        ELSE 'Low Priority - Monitor'
    END AS recommendation
FROM sys.dm_db_missing_index_details mid
INNER JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE migs.avg_user_impact > 10
AND migs.user_seeks + migs.user_scans > 10
ORDER BY migs.avg_total_user_cost DESC;

-- Step 2: Create recommended indexes
-- Example: Create index for Orders table based on missing index analysis
CREATE NONCLUSTERED INDEX IX_Orders_CustomerDate_Covering
ON Orders(CustomerID, OrderDate)
INCLUDE (OrderID, TotalAmount, Status)
WITH (ONLINE = ON, FILLFACTOR = 90);

-- Example: Create index for OrderDetails table
CREATE NONCLUSTERED INDEX IX_OrderDetails_OrderProduct_Covering
ON OrderDetails(OrderID, ProductID)
INCLUDE (Quantity, UnitPrice, ExtendedPrice)
WITH (ONLINE = ON, FILLFACTOR = 90);

-- Step 3: Update statistics after index creation
UPDATE STATISTICS Orders WITH FULLSCAN;
UPDATE STATISTICS OrderDetails WITH FULLSCAN;
UPDATE STATISTICS Customers WITH FULLSCAN;
```

4. **Query Optimization Examples**
```sql
-- Example 1: Optimize inefficient JOIN query
-- Before optimization
SELECT 
    c.CustomerID,
    c.FirstName,
    c.LastName,
    o.OrderID,
    o.OrderDate,
    od.ProductID,
    p.ProductName,
    SUM(od.Quantity * od.UnitPrice) AS OrderTotal
FROM Customers c
INNER JOIN Orders o ON c.CustomerID = o.CustomerID
INNER JOIN OrderDetails od ON o.OrderID = od.OrderID
INNER JOIN Products p ON od.ProductID = p.ProductID
WHERE c.Region = 'North'
AND o.OrderDate >= DATEADD(MONTH, -3, GETDATE())
GROUP BY c.CustomerID, c.FirstName, c.LastName, o.OrderID, o.OrderDate, od.ProductID, p.ProductName
ORDER BY OrderTotal DESC;

-- After optimization with proper indexes and query rewrite
SELECT 
    c.CustomerID,
    c.FirstName,
    c.LastName,
    o.OrderID,
    o.OrderDate,
    od.ProductID,
    p.ProductName,
    SUM(od.Quantity * od.UnitPrice) AS OrderTotal
FROM Customers c WITH (INDEX(IX_Customers_Region))  -- Force index usage
INNER JOIN Orders o ON c.CustomerID = o.CustomerID
INNER JOIN OrderDetails od WITH (INDEX(IX_OrderDetails_OrderProduct_Covering)) ON o.OrderID = od.OrderID
INNER JOIN Products p ON od.ProductID = p.ProductID
WHERE c.Region = 'North'
AND o.OrderDate >= DATEADD(MONTH, -3, GETDATE())
GROUP BY c.CustomerID, c.FirstName, c.LastName, o.OrderID, o.OrderDate, od.ProductID, p.ProductName
OPTION (OPTIMIZE FOR UNKNOWN);  -- Prevent parameter sniffing

-- Example 2: Optimize subquery with EXISTS
-- Before optimization
SELECT 
    c.CustomerID,
    c.FirstName,
    c.LastName,
    c.Email
FROM Customers c
WHERE EXISTS (
    SELECT 1 
    FROM Orders o
    INNER JOIN OrderDetails od ON o.OrderID = od.OrderID
    WHERE o.CustomerID = c.CustomerID
    AND o.OrderDate >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY o.OrderID
    HAVING SUM(od.Quantity * od.UnitPrice) > 1000
);

-- After optimization with CTE and proper indexing
WITH HighValueOrders AS (
    SELECT 
        o.CustomerID,
        SUM(od.Quantity * od.UnitPrice) AS OrderTotal
    FROM Orders o
    INNER JOIN OrderDetails od ON o.OrderID = od.OrderID
    WHERE o.OrderDate >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY o.CustomerID
    HAVING SUM(od.Quantity * od.UnitPrice) > 1000
)
SELECT 
    c.CustomerID,
    c.FirstName,
    c.LastName,
    c.Email
FROM Customers c
INNER JOIN HighValueOrders hvo ON c.CustomerID = hvo.CustomerID;
```

5. **Performance Validation**
```sql
-- Step 1: Capture performance baseline before optimization
CREATE TABLE #PerformanceBaseline (
    QueryHash BINARY(8),
    QueryText NVARCHAR(MAX),
    ExecutionCount INT,
    AvgDurationMS DECIMAL(18,4),
    TotalDurationMS BIGINT,
    CollectionTime DATETIME2 DEFAULT GETDATE()
);

INSERT INTO #PerformanceBaseline
SELECT 
    qs.query_hash,
    SUBSTRING(st.text, 1, 2000),
    qs.execution_count,
    qs.total_elapsed_time / NULLIF(qs.execution_count, 0) / 1000.0,
    qs.total_elapsed_time / 1000.0
FROM sys.dm_exec_query_stats qs
OUTER APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.last_execution_time > DATEADD(HOUR, -24, GETDATE())
AND st.dbid = DB_ID();

-- Step 2: Wait for performance data collection (simulate time passing)
-- In practice, this would be after running the optimized queries
-- For this exercise, we'll simulate performance improvement

-- Step 3: Compare performance after optimization
SELECT 
    pb.QueryText,
    pb.ExecutionCount as BaselineExecutions,
    qs.execution_count as CurrentExecutions,
    pb.AvgDurationMS as BaselineAvgMS,
    qs.total_elapsed_time / NULLIF(qs.execution_count, 0) / 1000.0 as CurrentAvgMS,
    pb.AvgDurationMS - (qs.total_elapsed_time / NULLIF(qs.execution_count, 0) / 1000.0) as ImprovementMS,
    CASE 
        WHEN pb.AvgDurationMS > 0 THEN 
            ((pb.AvgDurationMS - (qs.total_elapsed_time / NULLIF(qs.execution_count, 0) / 1000.0)) / pb.AvgDurationMS * 100)
        ELSE 0 
    END as ImprovementPercent,
    CASE 
        WHEN (qs.total_elapsed_time / NULLIF(qs.execution_count, 0) / 1000.0) < (pb.AvgDurationMS * 0.5) THEN 'Optimization Successful'
        WHEN (qs.total_elapsed_time / NULLIF(qs.execution_count, 0) / 1000.0) < (pb.AvgDurationMS * 0.8) THEN 'Moderate Improvement'
        ELSE 'Needs Further Optimization'
    END as OptimizationResult
FROM #PerformanceBaseline pb
INNER JOIN sys.dm_exec_query_stats qs ON pb.QueryHash = qs.query_hash
OUTER APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.last_execution_time > DATEADD(HOUR, -1, GETDATE())
ORDER BY ImprovementPercent DESC;

-- Clean up baseline table
DROP TABLE #PerformanceBaseline;
```

**Deliverable**: Complete query optimization report with before/after performance metrics and implemented optimizations

### Exercise 3: Index Maintenance Automation

**Objective**: Create an automated index maintenance system that proactively manages index health

**Scenario**: Implement a comprehensive index maintenance solution for TechCorp's production database

**Requirements**:
- **Automation**: Daily index fragmentation checks
- **Intelligence**: Smart decisions on rebuild vs. reorganize
- **Scheduling**: Non-intrusive maintenance windows
- **Monitoring**: Alert on maintenance issues
- **Reporting**: Weekly index health reports

**Tasks**:

1. **Create Index Maintenance Tables**
```sql
-- Create index maintenance schema
CREATE SCHEMA IndexMaintenance AUTHORIZATION dbo;

-- Index maintenance history
CREATE TABLE IndexMaintenance.MaintenanceHistory (
    MaintenanceID INT IDENTITY(1,1) PRIMARY KEY,
    MaintenanceDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    TableSchema NVARCHAR(128) NOT NULL,
    TableName NVARCHAR(128) NOT NULL,
    IndexName NVARCHAR(128) NOT NULL,
    ActionTaken NVARCHAR(20) NOT NULL, -- REBUILD, REORGANIZE, NOTHING
    FragmentationBefore DECIMAL(5,2),
    FragmentationAfter DECIMAL(5,2),
    PageCount BIGINT,
    DurationSeconds INT,
    Success BIT NOT NULL,
    ErrorMessage NVARCHAR(MAX),
    Operator NVARCHAR(128) NOT NULL DEFAULT SYSTEM_USER
);

-- Index health monitoring
CREATE TABLE IndexMaintenance.IndexHealth (
    HealthID INT IDENTITY(1,1) PRIMARY KEY,
    CheckDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    TableSchema NVARCHAR(128),
    TableName NVARCHAR(128),
    IndexName NVARCHAR(128),
    IndexType NVARCHAR(50),
    FragmentationPercent DECIMAL(5,2),
    PageCount BIGINT,
    RecordCount BIGINT,
    AvgPageSpaceUsed DECIMAL(5,2),
    IndexLevel TINYINT,
    RecommendedAction NVARCHAR(20),
    Priority TINYINT, -- 1=Critical, 2=High, 3=Medium, 4=Low
    MaintenanceWindow NVARCHAR(50) -- OFF_PEAK, BUSINESS_HOURS, EMERGENCY
);

-- Maintenance configuration
CREATE TABLE IndexMaintenance.MaintenanceConfig (
    ConfigID INT IDENTITY(1,1) PRIMARY KEY,
    SettingName NVARCHAR(100) NOT NULL,
    SettingValue NVARCHAR(500) NOT NULL,
    Description NVARCHAR(500),
    LastUpdated DATETIME2 NOT NULL DEFAULT GETDATE(),
    UpdatedBy NVARCHAR(128) NOT NULL DEFAULT SYSTEM_USER
);

-- Insert default configuration
INSERT INTO IndexMaintenance.MaintenanceConfig (SettingName, SettingValue, Description)
VALUES 
    ('RebuildThreshold', '30', 'Rebuild indexes with fragmentation > 30%'),
    ('ReorganizeThreshold', '10', 'Reorganize indexes with fragmentation > 10%'),
    ('MinPageCount', '100', 'Only maintain indexes with > 100 pages'),
    ('MaintenanceStartTime', '02:00', 'Start maintenance at 2:00 AM'),
    ('MaintenanceEndTime', '06:00', 'End maintenance at 6:00 AM'),
    ('OnlineRebuild', '1', 'Use ONLINE = ON for rebuilds'),
    ('FillFactor', '90', 'Set fill factor to 90%'),
    ('MaxExecutionTime', '1800', 'Max 30 minutes per maintenance cycle'),
    ('AlertEmail', 'dba@company.com', 'Email for maintenance alerts'),
    ('HighPriorityTables', 'Orders,Customers,Products', 'Tables that need priority maintenance');
```

2. **Create Intelligent Index Analysis**
```sql
-- Intelligent index analysis procedure
CREATE PROCEDURE IndexMaintenance.sp_AnalyzeIndexHealth
    @DatabaseName NVARCHAR(128) = NULL,
    @DetailedAnalysis BIT = 0
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @RebuildThreshold DECIMAL(5,2);
    DECLARE @ReorganizeThreshold DECIMAL(5,2);
    DECLARE @MinPageCount INT;
    DECLARE @HighPriorityTables NVARCHAR(500);
    
    -- Get configuration values
    SELECT @RebuildThreshold = CAST(SettingValue AS DECIMAL(5,2)) 
    FROM IndexMaintenance.MaintenanceConfig WHERE SettingName = 'RebuildThreshold';
    
    SELECT @ReorganizeThreshold = CAST(SettingValue AS DECIMAL(5,2)) 
    FROM IndexMaintenance.MaintenanceConfig WHERE SettingName = 'ReorganizeThreshold';
    
    SELECT @MinPageCount = CAST(SettingValue AS INT) 
    FROM IndexMaintenance.MaintenanceConfig WHERE SettingName = 'MinPageCount';
    
    SELECT @HighPriorityTables = SettingValue 
    FROM IndexMaintenance.MaintenanceConfig WHERE SettingName = 'HighPriorityTables';
    
    -- Clear previous analysis
    TRUNCATE TABLE IndexMaintenance.IndexHealth;
    
    -- Analyze indexes
    IF @DetailedAnalysis = 1
    BEGIN
        SET @SQL = '
        INSERT INTO IndexMaintenance.IndexHealth (
            TableSchema, TableName, IndexName, IndexType, FragmentationPercent,
            PageCount, RecordCount, AvgPageSpaceUsed, IndexLevel, RecommendedAction, Priority
        )
        SELECT 
            OBJECT_SCHEMA_NAME(ips.object_id) AS schema_name,
            OBJECT_NAME(ips.object_id) AS table_name,
            i.name AS index_name,
            i.type_desc AS index_type,
            ips.avg_fragmentation_in_percent,
            ips.page_count,
            ips.record_count,
            ips.avg_page_space_used_in_percent,
            ips.index_level,
            CASE 
                WHEN ips.avg_fragmentation_in_percent > ' + CAST(@RebuildThreshold AS VARCHAR(10)) + ' THEN ''REBUILD''
                WHEN ips.avg_fragmentation_in_percent > ' + CAST(@ReorganizeThreshold AS VARCHAR(10)) + ' THEN ''REORGANIZE''
                ELSE ''NOTHING''
            END AS recommended_action,
            CASE 
                -- Critical: High fragmentation on priority tables
                WHEN ips.avg_fragmentation_in_percent > ' + CAST(@RebuildThreshold AS VARCHAR(10)) + ' 
                    AND OBJECT_NAME(ips.object_id) IN (SELECT value FROM STRING_SPLIT(''' + @HighPriorityTables + ''', '','')) THEN 1
                -- High: Very high fragmentation on any table
                WHEN ips.avg_fragmentation_in_percent > ' + CAST(@RebuildThreshold AS VARCHAR(10)) + ' 
                    AND ips.page_count > 1000 THEN 2
                -- Medium: Moderate fragmentation on large indexes
                WHEN ips.avg_fragmentation_in_percent > ' + CAST(@ReorganizeThreshold AS VARCHAR(10)) + ' 
                    AND ips.page_count > ' + CAST(@MinPageCount AS VARCHAR(10)) + ' THEN 3
                -- Low: Low fragmentation or small indexes
                ELSE 4
            END AS priority
        FROM sys.dm_db_index_physical_stats(DB_ID(' + ISNULL('''' + @DatabaseName + '''', 'NULL') + '), NULL, NULL, NULL, ''DETAILED'') ips
        INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
        WHERE ips.avg_fragmentation_in_percent > ' + CAST(@ReorganizeThreshold AS VARCHAR(10)) + '
        AND ips.page_count > ' + CAST(@MinPageCount AS VARCHAR(10)) + '
        AND i.name NOT LIKE ''PK_%''  -- Skip primary keys
        AND i.name NOT LIKE ''_WA_%''  -- Skip worktable indexes
        ORDER BY priority, ips.avg_fragmentation_in_percent DESC, ips.page_count DESC;';
    END
    ELSE
    BEGIN
        -- Simplified analysis
        SET @SQL = '
        INSERT INTO IndexMaintenance.IndexHealth (
            TableSchema, TableName, IndexName, IndexType, FragmentationPercent,
            PageCount, RecordCount, RecommendedAction, Priority
        )
        SELECT 
            OBJECT_SCHEMA_NAME(ips.object_id) AS schema_name,
            OBJECT_NAME(ips.object_id) AS table_name,
            i.name AS index_name,
            i.type_desc AS index_type,
            ips.avg_fragmentation_in_percent,
            ips.page_count,
            ips.record_count,
            CASE 
                WHEN ips.avg_fragmentation_in_percent > ' + CAST(@RebuildThreshold AS VARCHAR(10)) + ' THEN ''REBUILD''
                WHEN ips.avg_fragmentation_in_percent > ' + CAST(@ReorganizeThreshold AS VARCHAR(10)) + ' THEN ''REORGANIZE''
                ELSE ''NOTHING''
            END AS recommended_action,
            CASE 
                WHEN ips.avg_fragmentation_in_percent > ' + CAST(@RebuildThreshold AS VARCHAR(10)) + ' THEN 1
                WHEN ips.avg_fragmentation_in_percent > ' + CAST(@ReorganizeThreshold AS VARCHAR(10)) + ' THEN 2
                ELSE 3
            END AS priority
        FROM sys.dm_db_index_physical_stats(DB_ID(' + ISNULL('''' + @DatabaseName + '''', 'NULL') + '), NULL, NULL, NULL, ''LIMITED'') ips
        INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
        WHERE ips.avg_fragmentation_in_percent > ' + CAST(@ReorganizeThreshold AS VARCHAR(10)) + '
        AND ips.page_count > ' + CAST(@MinPageCount AS VARCHAR(10)) + '
        ORDER BY priority, ips.avg_fragmentation_in_percent DESC;';
    END
    
    EXEC sp_executesql @SQL;
    
    -- Set maintenance window for each index
    UPDATE IndexMaintenance.IndexHealth
    SET MaintenanceWindow = CASE 
        WHEN Priority = 1 THEN 'EMERGENCY'
        WHEN Priority = 2 THEN 'OFF_PEAK'
        WHEN Priority = 3 THEN 'OFF_PEAK'
        ELSE 'SCHEDULED'
    END;
    
    -- Return analysis summary
    SELECT 
        'Index Health Analysis Summary' AS summary_type,
        COUNT(*) AS total_indexes_analyzed,
        SUM(CASE WHEN RecommendedAction = 'REBUILD' THEN 1 ELSE 0 END) AS indexes_need_rebuild,
        SUM(CASE WHEN RecommendedAction = 'REORGANIZE' THEN 1 ELSE 0 END) AS indexes_need_reorganize,
        SUM(CASE WHEN Priority = 1 THEN 1 ELSE 0 END) AS critical_indexes,
        SUM(CASE WHEN Priority = 2 THEN 1 ELSE 0 END) AS high_priority_indexes,
        AVG(FragmentationPercent) AS avg_fragmentation_percent,
        MAX(FragmentationPercent) AS max_fragmentation_percent,
        SUM(PageCount) AS total_pages,
        DatabaseName = ISNULL(@DatabaseName, 'Current Database')
    FROM IndexMaintenance.IndexHealth;
END;
```

3. **Create Automated Maintenance Execution**
```sql
-- Automated index maintenance procedure
CREATE PROCEDURE IndexMaintenance.sp_ExecuteMaintenance
    @MaintenanceWindow NVARCHAR(50) = 'OFF_PEAK',
    @MaxExecutionTime INT = 1800, -- 30 minutes
    @DryRun BIT = 0
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @StartTime DATETIME2 = GETDATE();
    DECLARE @EndTime DATETIME2 = DATEADD(SECOND, @MaxExecutionTime, @StartTime);
    DECLARE @OnlineRebuild BIT;
    DECLARE @FillFactor INT;
    
    -- Get configuration
    SELECT @OnlineRebuild = CAST(SettingValue AS BIT) 
    FROM IndexMaintenance.MaintenanceConfig WHERE SettingName = 'OnlineRebuild';
    
    SELECT @FillFactor = CAST(SettingValue AS INT) 
    FROM IndexMaintenance.MaintenanceConfig WHERE SettingName = 'FillFactor';
    
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @SchemaName NVARCHAR(128);
    DECLARE @TableName NVARCHAR(128);
    DECLARE @IndexName NVARCHAR(128);
    DECLARE @Action NVARCHAR(20);
    DECLARE @Priority TINYINT;
    DECLARE @Fragmentation DECIMAL(5,2);
    DECLARE @MaintenanceID INT;
    DECLARE @Success BIT;
    DECLARE @ErrorMessage NVARCHAR(MAX);
    
    -- Cursor through indexes that need maintenance
    DECLARE maintenance_cursor CURSOR FOR
    SELECT 
        TableSchema, TableName, IndexName, RecommendedAction, Priority, FragmentationPercent
    FROM IndexMaintenance.IndexHealth
    WHERE RecommendedAction IN ('REBUILD', 'REORGANIZE')
    AND Priority <= CASE @MaintenanceWindow
        WHEN 'EMERGENCY' THEN 1
        WHEN 'OFF_PEAK' THEN 2
        WHEN 'SCHEDULED' THEN 3
        ELSE 4
    END
    ORDER BY Priority, FragmentationPercent DESC;
    
    OPEN maintenance_cursor;
    FETCH NEXT FROM maintenance_cursor INTO @SchemaName, @TableName, @IndexName, @Action, @Priority, @Fragmentation;
    
    WHILE @@FETCH_STATUS = 0 AND GETDATE() < @EndTime
    BEGIN
        SET @Success = 1;
        SET @ErrorMessage = NULL;
        
        BEGIN TRY
            IF @DryRun = 1
            BEGIN
                -- Log dry run
                SET @SQL = 'DRY RUN: ' + @Action + ' index ' + @IndexName + ' on ' + @SchemaName + '.' + @TableName;
                PRINT @SQL;
            END
            ELSE
            BEGIN
                -- Execute actual maintenance
                IF @Action = 'REBUILD'
                BEGIN
                    SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @SchemaName + '.' + @TableName + 
                              ' REBUILD WITH (FILLFACTOR = ' + CAST(@FillFactor AS VARCHAR(10));
                    
                    IF @OnlineRebuild = 1
                        SET @SQL = @SQL + ', ONLINE = ON';
                    
                    SET @SQL = @SQL + ');';
                END
                ELSE IF @Action = 'REORGANIZE'
                BEGIN
                    SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @SchemaName + '.' + @TableName + ' REORGANIZE;';
                END
                
                -- Execute maintenance
                EXEC sp_executesql @SQL;
                
                -- Get fragmentation after maintenance
                DECLARE @FragmentationAfter DECIMAL(5,2);
                
                SELECT @FragmentationAfter = ips.avg_fragmentation_in_percent
                FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
                INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
                WHERE OBJECT_SCHEMA_NAME(ips.object_id) = @SchemaName
                AND OBJECT_NAME(ips.object_id) = @TableName
                AND i.name = @IndexName;
                
                -- Log successful maintenance
                INSERT INTO IndexMaintenance.MaintenanceHistory (
                    TableSchema, TableName, IndexName, ActionTaken, FragmentationBefore,
                    FragmentationAfter, PageCount, DurationSeconds, Success
                )
                VALUES (
                    @SchemaName, @TableName, @IndexName, @Action, @Fragmentation,
                    @FragmentationAfter, 
                    (SELECT page_count FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') 
                     WHERE OBJECT_SCHEMA_NAME(object_id) = @SchemaName AND OBJECT_NAME(object_id) = @TableName AND name = @IndexName),
                    DATEDIFF(SECOND, @StartTime, GETDATE()),
                    1
                );
            END
        END TRY
        BEGIN CATCH
            SET @Success = 0;
            SET @ErrorMessage = ERROR_MESSAGE();
            
            -- Log failed maintenance
            INSERT INTO IndexMaintenance.MaintenanceHistory (
                TableSchema, TableName, IndexName, ActionTaken, FragmentationBefore,
                DurationSeconds, Success, ErrorMessage
            )
            VALUES (
                @SchemaName, @TableName, @IndexName, @Action, @Fragmentation,
                DATEDIFF(SECOND, @StartTime, GETDATE()), 0, @ErrorMessage
            );
        END CATCH
        
        -- Update index health status
        IF @Success = 1
        BEGIN
            UPDATE IndexMaintenance.IndexHealth
            SET RecommendedAction = 'COMPLETED'
            WHERE TableSchema = @SchemaName
            AND TableName = @TableName
            AND IndexName = @IndexName;
        END
        
        FETCH NEXT FROM maintenance_cursor INTO @SchemaName, @TableName, @IndexName, @Action, @Priority, @Fragmentation;
    END
    
    CLOSE maintenance_cursor;
    DEALLOCATE maintenance_cursor;
    
    -- Return maintenance summary
    SELECT 
        'Index Maintenance Summary' AS summary_type,
        StartTime = @StartTime,
        EndTime = GETDATE(),
        DurationSeconds = DATEDIFF(SECOND, @StartTime, GETDATE()),
        TotalIndexesProcessed = (
            SELECT COUNT(*) 
            FROM IndexMaintenance.MaintenanceHistory 
            WHERE MaintenanceDate >= @StartTime
        ),
        SuccessfulMaintenance = (
            SELECT COUNT(*) 
            FROM IndexMaintenance.MaintenanceHistory 
            WHERE MaintenanceDate >= @StartTime AND Success = 1
        ),
        FailedMaintenance = (
            SELECT COUNT(*) 
            FROM IndexMaintenance.MaintenanceHistory 
            WHERE MaintenanceDate >= @StartTime AND Success = 0
        ),
        MaintenanceWindow = @MaintenanceWindow,
        IsDryRun = @DryRun;
END;
```

4. **Create Maintenance Scheduling and Alerts**
```sql
-- Create weekly maintenance report
CREATE PROCEDURE IndexMaintenance.sp_GenerateWeeklyReport
    @WeeksBack INT = 1
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @ReportStartDate DATETIME2 = DATEADD(WEEK, -@WeeksBack, GETDATE());
    
    -- Index maintenance summary
    SELECT 
        'Weekly Index Maintenance Report' AS report_title,
        ReportPeriod = 'From ' + CONVERT(VARCHAR(10), @ReportStartDate, 120) + ' to ' + CONVERT(VARCHAR(10), GETDATE(), 120),
        TotalMaintenanceActivities = COUNT(*),
        SuccessfulActivities = SUM(CASE WHEN Success = 1 THEN 1 ELSE 0 END),
        FailedActivities = SUM(CASE WHEN Success = 0 THEN 1 ELSE 0 END),
        RebuildsExecuted = SUM(CASE WHEN ActionTaken = 'REBUILD' THEN 1 ELSE 0 END),
        ReorganizationsExecuted = SUM(CASE WHEN ActionTaken = 'REORGANIZE' THEN 1 ELSE 0 END),
        AverageMaintenanceTime = AVG(DurationSeconds),
        TotalMaintenanceTime = SUM(DurationSeconds)
    FROM IndexMaintenance.MaintenanceHistory
    WHERE MaintenanceDate >= @ReportStartDate;
    
    -- Index health by priority
    SELECT 
        Priority,
        COUNT(*) AS IndexCount,
        AVG(FragmentationPercent) AS AvgFragmentation,
        MAX(FragmentationPercent) AS MaxFragmentation,
        SUM(CASE WHEN RecommendedAction = 'REBUILD' THEN 1 ELSE 0 END) AS NeedRebuild,
        SUM(CASE WHEN RecommendedAction = 'REORGANIZE' THEN 1 ELSE 0 END) AS NeedReorganize,
        PriorityDescription = CASE 
            WHEN Priority = 1 THEN 'Critical - Immediate attention required'
            WHEN Priority = 2 THEN 'High - Schedule for next maintenance window'
            WHEN Priority = 3 THEN 'Medium - Regular maintenance cycle'
            ELSE 'Low - Monitor only'
        END
    FROM IndexMaintenance.IndexHealth
    GROUP BY Priority
    ORDER BY Priority;
    
    -- Tables with highest fragmentation
    SELECT TOP 10
        TableSchema + '.' + TableName AS TableName,
        COUNT(*) AS IndexCount,
        AVG(FragmentationPercent) AS AvgFragmentation,
        MAX(FragmentationPercent) AS MaxFragmentation,
        SUM(PageCount) AS TotalPages,
        SUM(CASE WHEN RecommendedAction = 'REBUILD' THEN 1 ELSE 0 END) AS CriticalIndexes,
        MIN(MaintenanceWindow) AS RecommendedMaintenanceWindow
    FROM IndexMaintenance.IndexHealth
    GROUP BY TableSchema, TableName
    HAVING COUNT(*) > 1  -- Tables with multiple indexes
    ORDER BY AvgFragmentation DESC;
    
    -- Failed maintenance activities (if any)
    SELECT 
        TableSchema + '.' + TableName AS TableName,
        IndexName,
        ActionTaken,
        MaintenanceDate,
        ErrorMessage,
        DurationSeconds,
        RetryRecommended = CASE 
            WHEN MaintenanceDate > DATEADD(HOUR, -24, GETDATE()) THEN 'Yes'
            ELSE 'Schedule for next maintenance window'
        END
    FROM IndexMaintenance.MaintenanceHistory
    WHERE Success = 0
    AND MaintenanceDate >= @ReportStartDate
    ORDER BY MaintenanceDate DESC;
END;

-- Create maintenance scheduling job
USE msdb;
EXEC dbo.sp_add_job
    @job_name = N'Index Maintenance - Weekly Analysis',
    @enabled = 1,
    @description = N'Weekly index health analysis and maintenance planning';

EXEC sp_add_jobstep
    @job_name = N'Index Maintenance - Weekly Analysis',
    @step_name = N'Analyze Index Health',
    @command = N'EXEC IndexMaintenance.sp_AnalyzeIndexHealth @DatabaseName = NULL, @DetailedAnalysis = 1;';

EXEC sp_add_schedule
    @schedule_name = N'Weekly Sunday 1 AM',
    @freq_type = 8,  -- Weekly
    @freq_interval = 1,  -- Sunday
    @freq_subday_type = 1,  -- Once
    @freq_subday_interval = 0,
    @active_start_date = 20230101,
    @active_end_date = 99991231,
    @active_start_time = 100,  -- 1:00 AM (100 = 1:00 AM)
    @active_end_time = 20000;  -- End of day

EXEC sp_add_schedule
    @schedule_name = N'Daily Off-Peak Maintenance',
    @freq_type = 4,  -- Daily
    @freq_subday_type = 1,  -- Once
    @freq_subday_interval = 0,
    @active_start_date = 20230101,
    @active_end_date = 99991231,
    @active_start_time = 20000,  -- 2:00 AM
    @active_end_time = 60000;  -- 6:00 AM

EXEC sp_attach_schedule
    @job_name = N'Index Maintenance - Weekly Analysis',
    @schedule_name = N'Weekly Sunday 1 AM';

-- Create maintenance execution job
EXEC dbo.sp_add_job
    @job_name = N'Index Maintenance - Off-Peak Execution',
    @enabled = 1,
    @description = N'Daily off-peak index maintenance execution';

EXEC sp_add_jobstep
    @job_name = N'Index Maintenance - Off-Peak Execution',
    @step_name = N'Execute Index Maintenance',
    @command = N'EXEC IndexMaintenance.sp_ExecuteMaintenance @MaintenanceWindow = ''OFF_PEAK'', @MaxExecutionTime = 14400, @DryRun = 0;';

EXEC sp_attach_schedule
    @job_name = N'Index Maintenance - Off-Peak Execution',
    @schedule_name = N'Daily Off-Peak Maintenance';
```

**Deliverable**: Complete automated index maintenance system with intelligent analysis, scheduled execution, and comprehensive reporting

## Real-World Scenarios

### Scenario 1: E-commerce Performance Crisis Resolution

**Background**: ShopFast Online experiences critical performance degradation during Black Friday sales. Customer complaints are flooding in about slow page loads, timeout errors, and checkout failures. The database team must quickly identify and resolve performance issues to minimize revenue loss.

**Current Situation**:
- **Database**: E-commerce platform database (2TB, 10,000 concurrent users peak)
- **Performance Crisis**: Page load times > 30 seconds, 50% timeout rate
- **Business Impact**: $100,000/hour revenue loss
- **Available Time**: 2 hours to resolve or escalate to hardware upgrade
- **Current Metrics**: CPU at 95%, Memory pressure, High I/O wait times

**Performance Diagnosis Process**:

1. **Immediate Crisis Assessment**
```sql
-- Step 1: Quick health check
SELECT 
    'Current System Status' AS assessment_type,
    -- CPU usage
    (SELECT CAST(AVG(CASE WHEN SQLProcessUtilization + (100 - SystemIdle - SQLProcessUtilization) > 90 THEN 1 ELSE 0 END) * 100 AS DECIMAL(5,2))
     FROM (SELECT 
             record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
             record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SQLProcessUtilization)[1]', 'int') AS SQLProcessUtilization
           FROM (SELECT TOP 10 CAST(record AS XML) AS record
                 FROM sys.dm_os_ring_buffers
                 WHERE ring_buffer_type = 'RING_BUFFER_SCHEDULER_MONITOR'
                 ORDER BY timestamp DESC) AS x
           CROSS APPLY x.record.nodes('/Record') AS r(record)) AS y) AS cpu_usage_percent,
    
    -- Memory pressure
    (SELECT cntr_value FROM sys.dm_os_performance_counters
     WHERE counter_name = 'Page life expectancy'
     AND object_name LIKE '%Buffer Manager%') AS page_life_expectancy,
    
    -- Connection count
    (SELECT COUNT(*) FROM sys.dm_exec_sessions WHERE is_user_process = 1) AS active_connections,
    
    -- Blocking sessions
    (SELECT COUNT(*) FROM sys.dm_exec_requests WHERE blocking_session_id > 0) AS blocked_sessions,
    
    -- Longest running query
    (SELECT MAX(total_elapsed_time / 1000.0) FROM sys.dm_exec_requests) AS longest_query_seconds,
    
    -- Current time
    GETDATE() AS assessment_time;

-- Step 2: Identify top resource consumers
SELECT TOP 10
    r.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    r.status,
    r.command,
    r.cpu_time / 1000.0 AS cpu_ms,
    r.total_elapsed_time / 1000.0 AS duration_ms,
    r.reads,
    r.writes,
    r.logical_reads,
    r.blocking_session_id,
    r.wait_type,
    r.wait_time / 1000.0 AS wait_ms,
    SUBSTRING(st.text, (r.statement_start_offset/2)+1,
        ((CASE r.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE r.statement_end_offset
        END - r.statement_start_offset)/2) + 1) AS statement_text,
    CASE 
        WHEN r.total_elapsed_time / 1000.0 > 10000 THEN 'CRITICAL'
        WHEN r.total_elapsed_time / 1000.0 > 5000 THEN 'HIGH'
        WHEN r.total_elapsed_time / 1000.0 > 1000 THEN 'MEDIUM'
        ELSE 'NORMAL'
    END AS performance_impact
FROM sys.dm_exec_requests r
LEFT JOIN sys.dm_exec_sessions s ON r.session_id = s.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.session_id <> @@SPID
AND s.is_user_process = 1
ORDER BY r.total_elapsed_time DESC;
```

2. **Identify Top Query Performance Issues**
```sql
-- Step 3: Find the most problematic queries
SELECT TOP 20
    qs.query_hash,
    qs.execution_count,
    qs.total_elapsed_time / 1000.0 AS total_duration_ms,
    (qs.total_elapsed_time / qs.execution_count) / 1000.0 AS avg_duration_ms,
    qs.total_logical_reads / 1024.0 AS total_logical_reads_kb,
    (qs.total_logical_reads / qs.execution_count) / 1024.0 AS avg_logical_reads_kb,
    qs.total_worker_time / 1000.0 AS total_cpu_time_ms,
    (qs.total_worker_time / qs.execution_count) / 1000.0 AS avg_cpu_time_ms,
    qs.last_execution_time,
    SUBSTRING(st.text, 1, 1000) AS query_text,
    CASE 
        WHEN (qs.total_elapsed_time / qs.execution_count) / 1000.0 > 10000 THEN 'URGENT - > 10 seconds'
        WHEN (qs.total_elapsed_time / qs.execution_count) / 1000.0 > 5000 THEN 'CRITICAL - > 5 seconds'
        WHEN (qs.total_elapsed_time / qs.execution_count) / 1000.0 > 1000 THEN 'HIGH - > 1 second'
        ELSE 'MEDIUM'
    END AS urgency_level
FROM sys.dm_exec_query_stats qs
OUTER APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.execution_count > 10
AND qs.last_execution_time > DATEADD(MINUTE, -30, GETDATE())
ORDER BY (qs.total_elapsed_time / qs.execution_count) DESC;

-- Step 4: Check for missing indexes on top queries
SELECT TOP 10
    mid.statement AS table_name,
    migs.avg_user_impact,
    migs.avg_total_user_cost,
    migs.user_seeks,
    migs.user_scans,
    migs.last_user_seek,
    'CREATE INDEX IX_' + REPLACE(REPLACE(REPLACE(mid.statement, '[', ''), ']', ''), '.', '_') + '_' + 
    CAST(migs.index_handle AS VARCHAR(10)) + ' ON ' + 
    mid.statement + 
    ' (' + 
    ISNULL(mid.equality_columns, '') + 
    CASE WHEN mid.equality_columns IS NOT NULL AND mid.inequality_columns IS NOT NULL THEN ', ' ELSE '' END +
    ISNULL(mid.inequality_columns, '') + ')' +
    CASE WHEN mid.included_columns IS NOT NULL THEN ' INCLUDE (' + mid.included_columns + ')' ELSE '' END +
    ' WITH (ONLINE = ON);' AS create_index_ddl,
    CASE 
        WHEN migs.avg_user_impact > 70 THEN 'CRITICAL - Create immediately'
        WHEN migs.avg_user_impact > 30 THEN 'HIGH - Create during maintenance'
        ELSE 'MEDIUM - Monitor'
    END AS recommendation
FROM sys.dm_db_missing_index_details mid
INNER JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE migs.avg_user_impact > 10
AND migs.user_seeks + migs.user_scans > 100
ORDER BY migs.avg_total_user_cost DESC;
```

3. **Emergency Performance Interventions**
```sql
-- Step 5: Emergency actions for immediate relief

-- Action 1: Clear procedure cache if queries are using old plans
-- (Use with caution - only for critical situations)
-- DBCC FREEPROCCACHE;

-- Action 2: Update statistics on critical tables
UPDATE STATISTICS dbo.Products WITH FULLSCAN;
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;
UPDATE STATISTICS dbo.OrderDetails WITH FULLSCAN;
UPDATE STATISTICS dbo.Customers WITH FULLSCAN;

-- Action 3: Reorganize severely fragmented indexes
DECLARE @SchemaName NVARCHAR(128), @TableName NVARCHAR(128), @IndexName NVARCHAR(128);
DECLARE fragmentation_cursor CURSOR FOR
SELECT OBJECT_SCHEMA_NAME(ips.object_id), OBJECT_NAME(ips.object_id), i.name
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30
AND ips.page_count > 1000
AND i.name NOT LIKE 'PK_%'
ORDER BY ips.avg_fragmentation_in_percent DESC;

OPEN fragmentation_cursor;
FETCH NEXT FROM fragmentation_cursor INTO @SchemaName, @TableName, @IndexName;

WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT 'Reorganizing index: ' + @SchemaName + '.' + @TableName + '.' + @IndexName;
    EXEC('ALTER INDEX ' + @IndexName + ' ON ' + @SchemaName + '.' + @TableName + ' REORGANIZE;');
    FETCH NEXT FROM fragmentation_cursor INTO @SchemaName, @TableName, @IndexName;
END

CLOSE fragmentation_cursor;
DEALLOCATE fragmentation_cursor;

-- Action 4: Create critical missing indexes immediately
-- Based on missing index analysis, create the most critical ones
CREATE NONCLUSTERED INDEX IX_Products_Category_Covering
ON dbo.Products(CategoryID, IsActive)
INCLUDE (ProductID, ProductName, UnitPrice, InStock)
WITH (ONLINE = ON, FILLFACTOR = 90);

CREATE NONCLUSTERED INDEX IX_Orders_CustomerDate_Covering
ON dbo.Orders(CustomerID, OrderDate)
INCLUDE (OrderID, Status, TotalAmount)
WITH (ONLINE = ON, FILLFACTOR = 90);

-- Action 5: Kill blocking sessions if necessary
-- (Use very carefully - only for critical blocking)
-- SELECT 'KILL ' + CAST(session_id AS VARCHAR(10)) + ';' 
-- FROM sys.dm_exec_requests 
-- WHERE blocking_session_id > 0 
-- AND wait_time > 30000;  -- Only kill if blocking for > 30 seconds
```

4. **Monitor Crisis Resolution Progress**
```sql
-- Step 6: Monitor performance improvement
CREATE TABLE #PerformanceSnapshot (
    SnapshotTime DATETIME2 DEFAULT GETDATE(),
    MetricName NVARCHAR(100),
    MetricValue DECIMAL(18,4)
);

-- Take performance snapshot every 30 seconds for 5 minutes
WHILE DATEDIFF(SECOND, (SELECT MIN(SnapshotTime) FROM #PerformanceSnapshot), GETDATE()) < 300
BEGIN
    -- CPU usage
    INSERT INTO #PerformanceSnapshot (MetricName, MetricValue)
    SELECT 'CPU_Usage_Percent', CAST(AVG(CASE WHEN SQLProcessUtilization + (100 - SystemIdle - SQLProcessUtilization) > 90 THEN 1 ELSE 0 END) * 100 AS DECIMAL(5,2))
    FROM (SELECT 
             record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
             record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SQLProcessUtilization)[1]', 'int') AS SQLProcessUtilization
           FROM (SELECT TOP 10 CAST(record AS XML) AS record
                 FROM sys.dm_os_ring_buffers
                 WHERE ring_buffer_type = 'RING_BUFFER_SCHEDULER_MONITOR'
                 ORDER BY timestamp DESC) AS x
           CROSS APPLY x.record.nodes('/Record') AS r(record)) AS y;
    
    -- Page Life Expectancy
    INSERT INTO #PerformanceSnapshot (MetricName, MetricValue)
    SELECT 'Page_Life_Expectancy', CAST(cntr_value AS DECIMAL(18,4))
    FROM sys.dm_os_performance_counters
    WHERE counter_name = 'Page life expectancy';
    
    -- Longest query time
    INSERT INTO #PerformanceSnapshot (MetricName, MetricValue)
    SELECT 'Longest_Query_Seconds', CAST(MAX(total_elapsed_time) / 1000.0 AS DECIMAL(10,2))
    FROM sys.dm_exec_requests;
    
    -- Wait for 30 seconds
    WAITFOR DELAY '00:00:30';
END

-- Analyze performance improvement
SELECT 
    MetricName,
    MIN(MetricValue) AS MinValue,
    MAX(MetricValue) AS MaxValue,
    AVG(MetricValue) AS AvgValue,
    (MAX(MetricValue) - MIN(MetricValue)) / NULLIF(MIN(MetricValue), 0) * 100 AS ImprovementPercent,
    MIN(SnapshotTime) AS FirstSnapshot,
    MAX(SnapshotTime) AS LastSnapshot
FROM #PerformanceSnapshot
GROUP BY MetricName
ORDER BY MetricName;

DROP TABLE #PerformanceSnapshot;
```

5. **Post-Crisis Analysis and Prevention**
```sql
-- Step 7: Document the crisis and create prevention plan
CREATE TABLE dbo.PerformanceCrisisLog (
    CrisisID INT IDENTITY(1,1) PRIMARY KEY,
    CrisisDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    CrisisType NVARCHAR(100) NOT NULL,
    PeakUsers INT,
    CPUPeak DECIMAL(5,2),
    PLEPeak DECIMAL(10,2),
    DurationMinutes INT,
    RootCause NVARCHAR(MAX),
    ActionsTaken NVARCHAR(MAX),
    ResolutionTimeMinutes INT,
    RevenueImpact_Estimate DECIMAL(18,2),
    PreventionMeasures NVARCHAR(MAX),
    FollowUpRequired BIT DEFAULT 1
);

INSERT INTO dbo.PerformanceCrisisLog (
    CrisisType, PeakUsers, CPUPeak, PLEPeak, DurationMinutes, 
    RootCause, ActionsTaken, ResolutionTimeMinutes, RevenueImpact_Estimate
)
VALUES (
    'Black Friday Performance Degradation',
    10000,
    95.0,
    45,  -- Very low Page Life Expectancy
    120,
    'Missing indexes on Products and Orders tables, outdated statistics, high fragmentation',
    'Updated statistics, created critical indexes, reorganized fragmented indexes, killed blocking sessions',
    90,
    150000.00  -- Estimated revenue impact
);

-- Create preventive monitoring alerts
CREATE PROCEDURE sp_BlackFriday_PerformanceAlert
AS
BEGIN
    -- High CPU alert
    IF (SELECT CAST(AVG(CASE WHEN SQLProcessUtilization + (100 - SystemIdle - SQLProcessUtilization) > 85 THEN 1 ELSE 0 END) * 100 AS DECIMAL(5,2))
        FROM (SELECT 
                record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
                record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SQLProcessUtilization)[1]', 'int') AS SQLProcessUtilization
              FROM (SELECT TOP 5 CAST(record AS XML) AS record
                    FROM sys.dm_os_ring_buffers
                    WHERE ring_buffer_type = 'RING_BUFFER_SCHEDULER_MONITOR'
                    ORDER BY timestamp DESC) AS x
              CROSS APPLY x.record.nodes('/Record') AS r(record)) AS y) > 85
    BEGIN
        -- Automated emergency actions for Black Friday
        PRINT 'ALERT: High CPU detected during peak hours!';
        -- Update statistics immediately
        UPDATE STATISTICS dbo.Products WITH FULLSCAN;
        UPDATE STATISTICS dbo.Orders WITH FULLSCAN;
        UPDATE STATISTICS dbo.OrderDetails WITH FULLSCAN;
        -- Force plan recompilation
        DBCC FREEPROCCACHE;
    END
    
    -- Memory pressure alert
    DECLARE @PLE BIGINT;
    SELECT @PLE = cntr_value FROM sys.dm_os_performance_counters WHERE counter_name = 'Page life expectancy';
    
    IF @PLE < 300  -- Less than 5 minutes
    BEGIN
        PRINT 'ALERT: Critical memory pressure! Page Life Expectancy: ' + CAST(@PLE AS VARCHAR(10));
        -- Monitor memory usage and potential queries causing pressure
    END
END;
```

**Learning Outcome**: Rapid crisis response methodology for critical performance issues, including immediate diagnostics, emergency interventions, and progress monitoring during high-stakes business situations

### Scenario 2: Healthcare System Performance Optimization

**Background**: MedCare Health System needs to optimize their patient records database to handle increased patient load and support new electronic health record features. The current system is experiencing slow patient lookup times, delayed order entry, and reporting bottlenecks.

**Current Situation**:
- **Database**: PatientRecordsDB (800GB, 100 concurrent clinicians)
- **Performance Issues**: 8-second average patient lookup, 15-second order entry, 45-minute reporting queries
- **Compliance Requirements**: HIPAA audit trails, 99.9% availability, 2-second response time for critical functions
- **Growth**: 20% increase in patient volume over next 6 months
- **Regulatory**: Must maintain audit trails for all patient data access

**DBA Solution Implementation**:

1. **Comprehensive Performance Baseline**
```sql
-- Step 1: Establish comprehensive performance baseline
CREATE SCHEMA PerformanceBaseline AUTHORIZATION dbo;

-- Create performance metrics table for healthcare system
CREATE TABLE PerformanceBaseline.HealthcareMetrics (
    MetricID BIGINT IDENTITY(1,1) PRIMARY KEY,
    CollectionDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    MetricCategory NVARCHAR(50) NOT NULL, -- PATIENT_LOOKUP, ORDER_ENTRY, REPORTING, AUDIT
    QueryType NVARCHAR(100),
    AverageResponseTimeMS DECIMAL(10,2),
    MedianResponseTimeMS DECIMAL(10,2),
    P95ResponseTimeMS DECIMAL(10,2),
    P99ResponseTimeMS DECIMAL(10,2),
    ExecutionCount BIGINT,
    TotalDurationMS BIGINT,
    SuccessRate DECIMAL(5,2),  -- Percentage of successful executions
    ComplianceFlag BIT,  -- HIPAA compliance requirement
    AdditionalContext NVARCHAR(MAX)
);

-- Capture baseline for patient lookup operations
INSERT INTO PerformanceBaseline.HealthcareMetrics (
    MetricCategory, QueryType, AverageResponseTimeMS, MedianResponseTimeMS,
    P95ResponseTimeMS, P99ResponseTimeMS, ExecutionCount, TotalDurationMS,
    SuccessRate, ComplianceFlag
)
SELECT 
    'PATIENT_LOOKUP',
    'Patient_Search_By_Name',
    AVG(qs.total_elapsed_time / NULLIF(qs.execution_count, 0) / 1000.0),
    (SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY qs2.total_elapsed_time / qs2.execution_count) 
     FROM sys.dm_exec_query_stats qs2 
     WHERE qs2.query_hash = qs.query_hash) / 1000.0,
    (SELECT PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY qs2.total_elapsed_time / qs2.execution_count) 
     FROM sys.dm_exec_query_stats qs2 
     WHERE qs2.query_hash = qs.query_hash) / 1000.0,
    (SELECT PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY qs2.total_elapsed_time / qs2.execution_count) 
     FROM sys.dm_exec_query_stats qs2 
     WHERE qs2.query_hash = qs.query_hash) / 1000.0,
    qs.execution_count,
    qs.total_elapsed_time / 1000.0,
    100.0,  -- Assuming all executions are successful
    1  -- HIPAA compliance required
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE st.text LIKE '%Patients%'
AND st.text LIKE '%WHERE%'
AND qs.last_execution_time > DATEADD(DAY, -7, GETDATE())
AND qs.execution_count > 50
GROUP BY qs.query_hash
ORDER BY AVG(qs.total_elapsed_time / NULLIF(qs.execution_count, 0)) DESC;

-- Baseline for order entry operations
INSERT INTO PerformanceBaseline.HealthcareMetrics (
    MetricCategory, QueryType, AverageResponseTimeMS, MedianResponseTimeMS,
    P95ResponseTimeMS, P99ResponseTimeMS, ExecutionCount, TotalDurationMS,
    SuccessRate, ComplianceFlag
)
SELECT 
    'ORDER_ENTRY',
    'Order_Insert_Update',
    AVG(qs.total_elapsed_time / NULLIF(qs.execution_count, 0) / 1000.0),
    (SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY qs2.total_elapsed_time / qs2.execution_count) 
     FROM sys.dm_exec_query_stats qs2 
     WHERE qs2.query_hash = qs.query_hash) / 1000.0,
    (SELECT PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY qs2.total_elapsed_time / qs2.execution_count) 
     FROM sys.dm_exec_query_stats qs2 
     WHERE qs2.query_hash = qs.query_hash) / 1000.0,
    (SELECT PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY qs2.total_elapsed_time / qs2.execution_count) 
     FROM sys.dm_exec_query_stats qs2 
     WHERE qs2.query_hash = qs.query_hash) / 1000.0,
    qs.execution_count,
    qs.total_elapsed_time / 1000.0,
    100.0,
    1  -- HIPAA compliance required
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE (st.text LIKE '%Orders%' OR st.text LIKE '%OrderDetails%')
AND (st.text LIKE '%INSERT%' OR st.text LIKE '%UPDATE%')
AND qs.last_execution_time > DATEADD(DAY, -7, GETDATE())
AND qs.execution_count > 20
GROUP BY qs.query_hash
ORDER BY AVG(qs.total_elapsed_time / NULLIF(qs.execution_count, 0)) DESC;
```

2. **Healthcare-Specific Index Strategy**
```sql
-- Step 2: Analyze missing indexes for healthcare queries
SELECT 
    'Healthcare Index Analysis' AS analysis_type,
    mid.statement AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.avg_user_impact,
    migs.avg_total_user_cost,
    migs.user_seeks,
    migs.user_scans,
    CASE 
        WHEN mid.statement LIKE '%Patients%' THEN 'PATIENT_DATA'
        WHEN mid.statement LIKE '%Orders%' THEN 'CLINICAL_ORDERS'
        WHEN mid.statement LIKE '%MedicalRecords%' THEN 'MEDICAL_RECORDS'
        WHEN mid.statement LIKE '%Billing%' THEN 'BILLING_DATA'
        ELSE 'OTHER'
    END AS data_category,
    'CREATE INDEX IX_' + 
    REPLACE(REPLACE(REPLACE(mid.statement, '[', ''), ']', ''), '.', '_') + '_' + 
    CAST(migs.index_handle AS VARCHAR(10)) + '_HEALTHCARE ON ' + 
    mid.statement + 
    ' (' + 
    ISNULL(mid.equality_columns, '') + 
    CASE WHEN mid.equality_columns IS NOT NULL AND mid.inequality_columns IS NOT NULL THEN ', ' ELSE '' END +
    ISNULL(mid.inequality_columns, '') + ')' +
    CASE WHEN mid.included_columns IS NOT NULL THEN ' INCLUDE (' + mid.included_columns + ')' ELSE '' END +
    ' WITH (ONLINE = ON, FILLFACTOR = 90, PAD_INDEX = ON);' AS create_index_ddl,
    CASE 
        WHEN migs.avg_user_impact > 50 AND mid.statement LIKE '%Patients%' THEN 'CRITICAL - Patient lookup'
        WHEN migs.avg_user_impact > 40 AND mid.statement LIKE '%Orders%' THEN 'HIGH - Order entry'
        WHEN migs.avg_user_impact > 30 THEN 'MEDIUM - Regular optimization'
        ELSE 'LOW - Monitor'
    END AS healthcare_priority
FROM sys.dm_db_missing_index_details mid
INNER JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE migs.avg_user_impact > 10
AND migs.user_seeks + migs.user_scans > 50
ORDER BY migs.avg_total_user_cost DESC;

-- Implement critical healthcare indexes
-- Patient lookup optimization
CREATE NONCLUSTERED INDEX IX_Patients_Name_DOB_Covering
ON dbo.Patients(LastName, FirstName, DateOfBirth)
INCLUDE (PatientID, SSN, Phone, Email, LastVisitDate)
WITH (ONLINE = ON, FILLFORDER = 90);

-- Medical record number search
CREATE NONCLUSTERED INDEX IX_Patients_MRN_Covering
ON dbo.Patients(MedicalRecordNumber)
INCLUDE (PatientID, LastName, FirstName, DateOfBirth)
WITH (ONLINE = ON, FILLFACTOR = 90);

-- Active patients index
CREATE NONCLUSTERED INDEX IX_Patients_Active_Status
ON dbo.Patients(IsActive, LastVisitDate)
INCLUDE (PatientID, LastName, FirstName)
WHERE IsActive = 1;  -- Filtered index for active patients only

-- Order entry optimization
CREATE NONCLUSTERED INDEX IX_Orders_PatientDate_Covering
ON dbo.Orders(PatientID, OrderDate, Status)
INCLUDE (OrderID, OrderType, Priority)
WITH (ONLINE = ON, FILLFACTOR = 90);

-- Medical records optimization
CREATE NONCLUSTERED INDEX IX_MedicalRecords_PatientEncounter_Covering
ON dbo.MedicalRecords(PatientID, EncounterID, RecordDate)
INCLUDE (RecordType, ProviderID, DiagnosisCode)
WITH (ONLINE = ON, FILLFACTOR = 90);

-- Billing optimization
CREATE NONCLUSTERED INDEX IX_Billing_Claims_Status_Covering
ON dbo.BillingClaims(PatientID, ClaimStatus, ServiceDate)
INCLUDE (ClaimID, TotalAmount, ProviderID)
WITH (ONLINE = ON, FILLFACTOR = 90);
```

3. **HIPAA-Compliant Performance Optimization**
```sql
-- Step 3: Implement HIPAA audit trail performance optimization
-- Create optimized audit tables with proper indexing for compliance queries

-- Patient access audit table with optimized queries
CREATE TABLE dbo.PatientAccessAudit (
    AuditID BIGINT IDENTITY(1,1) PRIMARY KEY,
    PatientID INT NOT NULL,
    UserID NVARCHAR(128) NOT NULL,
    AccessType NVARCHAR(50) NOT NULL, -- VIEW, MODIFY, EXPORT
    AccessTime DATETIME2 NOT NULL DEFAULT GETDATE(),
    SessionID INT,
    IPAddress NVARCHAR(45),
    ApplicationName NVARCHAR(128),
    ReasonCode NVARCHAR(100),  -- Clinical need, administrative, etc.
    ComplianceFlags NVARCHAR(MAX),  -- JSON for regulatory flags
    -- Indexed columns for performance
    INDEX IX_PatientAccess_PatientID_Time (PatientID, AccessTime),
