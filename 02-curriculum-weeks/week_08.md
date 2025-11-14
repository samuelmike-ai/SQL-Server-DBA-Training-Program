# Week 08: Indexing Strategies

## Overview

This week focuses on mastering SQL Server indexing strategies to optimize query performance, reduce I/O operations, and maintain database efficiency. You'll learn about different index types, optimization techniques, and maintenance procedures that are essential for high-performance database administration.

## Learning Objectives

By the end of this week, you will be able to:
- Understand different SQL Server index types and their use cases
- Design effective indexing strategies based on query patterns
- Implement and optimize clustered and nonclustered indexes
- Use index fragmentation analysis and maintenance procedures
- Optimize columnstore indexes for data warehousing workloads
- Monitor index usage and performance impact
- Troubleshoot index-related performance issues

## Monday - Index Fundamentals and Types

### Understanding SQL Server Index Architecture

SQL Server indexes are specialized data structures that improve the speed of data retrieval operations on a database table. They work similarly to a book's index, providing quick access to data without scanning entire tables.

#### Clustered Indexes

Clustered indexes determine the physical order of data in a table. Each table can have only one clustered index because the data rows can be stored in only one order.

**Key Characteristics:**
- Defines the physical storage order of table data
- Only one per table
- Leaf level contains the actual data pages
- Automatically created on primary key columns (if specified)

**Implementation Example:**
```sql
-- Create clustered index on primary key
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY CLUSTERED,
    CustomerID INT,
    OrderDate DATETIME,
    OrderTotal DECIMAL(18,2)
);

-- Create clustered index on date column (alternative to primary key)
CREATE TABLE SalesData (
    SaleID INT IDENTITY(1,1),
    SaleDate DATETIME NOT NULL,
    ProductID INT,
    Quantity INT,
    UnitPrice DECIMAL(10,2)
);

CREATE CLUSTERED INDEX IX_SalesData_SaleDate 
ON SalesData(SaleDate);

-- Verify clustered index structure
SELECT 
    i.name AS IndexName,
    i.type_desc AS IndexType,
    i.is_primary_key AS IsPrimaryKey,
    i.is_unique AS IsUnique,
    i.fill_factor AS FillFactor,
    i.is_padded AS IsPadded
FROM sys.indexes i
INNER JOIN sys.tables t ON i.object_id = t.object_id
WHERE t.name = 'SalesData'
ORDER BY i.index_id;
```

**Performance Impact Analysis:**
```sql
-- Analyze clustered index usage and effectiveness
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.user_seeks AS UserSeeks,
    s.user_scans AS UserScans,
    s.user_lookups AS UserLookups,
    s.user_updates AS UserUpdates,
    s.last_user_seek AS LastUserSeek,
    s.last_user_scan AS LastUserScan,
    s.last_user_lookup AS LastUserLookup,
    s.last_user_update AS LastUserUpdate
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
AND OBJECT_NAME(s.object_id) = 'SalesData'
ORDER BY s.object_id, s.index_id;
```

#### Nonclustered Indexes

Nonclustered indexes don't change the physical order of the table data. Instead, they create a separate structure that points to the data rows.

**Key Characteristics:**
- Can have multiple per table (up to 999)
- Leaf level contains index key values and row identifiers (RID) or clustering key
- Can include additional columns (included columns)
- Can be filtered to cover subset of rows

**Implementation Examples:**

**Basic Nonclustered Index:**
```sql
-- Create nonclustered index on frequently queried column
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID 
ON Orders(CustomerID);

-- Create covering index with included columns
CREATE NONCLUSTERED INDEX IX_Orders_CustomerDate 
ON Orders(CustomerID, OrderDate)
INCLUDE (OrderTotal);

-- Filtered index for specific use case
CREATE NONCLUSTERED INDEX IX_Orders_ActiveCustomer 
ON Orders(CustomerID, OrderDate)
WHERE OrderStatus = 'Active';

-- Unique nonclustered index
CREATE UNIQUE NONCLUSTERED INDEX IX_Orders_OrderNumber 
ON Orders(OrderNumber);
```

**Advanced Indexing Strategies:**
```sql
-- Index with multiple key columns for composite queries
CREATE NONCLUSTERED INDEX IX_Orders_CustomerDateStatus 
ON Orders(CustomerID, OrderDate, OrderStatus)
INCLUDE (OrderTotal, ShippingAddress)
WITH (FILLFACTOR = 90, PAD_INDEX = ON);

-- Index for range queries
CREATE NONCLUSTERED INDEX IX_Orders_DateRange 
ON Orders(OrderDate)
INCLUDE (OrderID, CustomerID, OrderTotal)
WITH (FILLFACTOR = 95);

-- Index for JOIN optimization
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID_INC 
ON Orders(CustomerID)
INCLUDE (OrderID, OrderDate, OrderTotal, OrderStatus)
WITH (ONLINE = ON);
```

### Columnstore Indexes

Columnstore indexes store data in a columnar format, dramatically reducing I/O and improving performance for data warehousing queries.

**Key Benefits:**
- 10x better compression than rowstore
- 10x faster query performance for data warehousing
- Significantly reduced memory usage

**Implementation Examples:**

**Nonclustered Columnstore Index:**
```sql
-- Create table for data warehousing
CREATE TABLE SalesFact (
    SaleID BIGINT IDENTITY(1,1),
    DateKey INT,
    ProductKey INT,
    CustomerKey INT,
    StoreKey INT,
    SalesAmount DECIMAL(18,2),
    Quantity INT,
    DiscountAmount DECIMAL(10,2)
);

-- Create nonclustered columnstore index
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_Columnstore_SalesFact 
ON SalesFact(
    DateKey, ProductKey, CustomerKey, StoreKey, 
    SalesAmount, Quantity, DiscountAmount
)
WITH (DATA_COMPRESSION = COLUMNSTORE_ARCHIVE);

-- Query performance comparison
-- Enable actual execution plan to see differences
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Rowstore query (potentially slower)
SELECT 
    ProductKey,
    SUM(SalesAmount) as TotalSales,
    AVG(Quantity) as AvgQuantity,
    COUNT(*) as TransactionCount
FROM SalesFact
WHERE DateKey BETWEEN 20240101 AND 20241231
GROUP BY ProductKey
ORDER BY TotalSales DESC;

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
```

**Clustered Columnstore Index:**
```sql
-- Create table with clustered columnstore index
CREATE TABLE SalesFact_Clustered (
    SaleID BIGINT,
    DateKey INT,
    ProductKey INT,
    CustomerKey INT,
    StoreKey INT,
    SalesAmount DECIMAL(18,2),
    Quantity INT,
    DiscountAmount DECIMAL(10,2)
);

-- Create clustered columnstore index (this becomes the table's primary storage)
CREATE CLUSTERED COLUMNSTORE INDEX IX_ClusteredColumnstore_SalesFact 
ON SalesFact_Clustered
WITH (DATA_COMPRESSION = COLUMNSTORE, MAXDOP = 2);
```

### Index Design Best Practices

#### Index Design Principles

**1. Query Pattern Analysis:**
```sql
-- Analyze query patterns to guide index design
SELECT 
    OBJECT_NAME(qt.objectid) AS TableName,
    SUBSTRING(qt.text, 1, 200) AS QueryText,
    qs.execution_count AS ExecutionCount,
    qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
    qs.total_elapsed_time / qs.execution_count AS AvgElapsedTimeMs,
    qs.last_execution_time AS LastExecution
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
WHERE qt.text LIKE '%Orders%'
AND qt.text LIKE '%WHERE%'
ORDER BY qs.execution_count DESC;
```

**2. Column Selectivity Analysis:**
```sql
-- Analyze column selectivity for indexing decisions
SELECT 
    c.name AS ColumnName,
    t.name AS TableName,
    COUNT(DISTINCT c.name) OVER() AS TotalDistinct,
    COUNT(*) AS TotalRows,
    CAST(COUNT(DISTINCT c.name) AS FLOAT) / COUNT(*) AS Selectivity,
    -- Determine if column is suitable for indexing
    CASE 
        WHEN CAST(COUNT(DISTINCT c.name) AS FLOAT) / COUNT(*) > 0.95 THEN 'HIGHLY SELECTIVE - Good for index'
        WHEN CAST(COUNT(DISTINCT c.name) AS FLOAT) / COUNT(*) > 0.80 THEN 'MODERATELY SELECTIVE - Consider composite index'
        WHEN CAST(COUNT(DISTINCT c.name) AS FLOAT) / COUNT(*) > 0.50 THEN 'LOW SELECTIVE - Consider filtered index'
        ELSE 'NOT SELECTIVE - Index not recommended'
    END AS IndexRecommendation
FROM sys.columns c
INNER JOIN sys.tables t ON c.object_id = t.object_id
INNER JOIN sys.index_columns ic ON c.object_id = ic.object_id AND c.column_id = ic.column_id
INNER JOIN sys.indexes i ON ic.object_id = i.object_id AND ic.index_id = i.index_id
WHERE t.name = 'Orders' 
AND i.type_desc = 'HEAP'
GROUP BY c.name, t.name
ORDER BY c.name;
```

## Tuesday - Index Optimization Techniques

### Query Optimization with Indexes

#### Covering Indexes

A covering index includes all columns needed to satisfy a query, eliminating the need for key lookups.

**Example Implementation:**
```sql
-- Create covering index for customer order queries
CREATE NONCLUSTERED INDEX IX_Orders_Covering_Customer
ON Orders(CustomerID, OrderDate, OrderStatus)
INCLUDE (OrderID, OrderTotal, ShippingAddress, BillingAddress)
WITH (ONLINE = ON);

-- Query that benefits from covering index
SELECT 
    OrderID,
    CustomerID,
    OrderDate,
    OrderTotal,
    ShippingAddress
FROM Orders
WHERE CustomerID = @CustomerID
  AND OrderDate >= @StartDate
  AND OrderStatus = 'Active'
ORDER BY OrderDate DESC;
```

**Performance Comparison:**
```sql
-- Compare query performance with and without covering index
-- Step 1: Create test data
CREATE TABLE PerformanceTest (
    ID INT IDENTITY(1,1) PRIMARY KEY,
    CustomerID INT,
    OrderDate DATETIME,
    OrderStatus NVARCHAR(50),
    OrderTotal DECIMAL(18,2),
    ShippingAddress NVARCHAR(200),
    ProductInfo NVARCHAR(500)
);

-- Insert test data
INSERT INTO PerformanceTest (CustomerID, OrderDate, OrderStatus, OrderTotal, ShippingAddress, ProductInfo)
SELECT 
    ABS(CHECKSUM(NEWID())) % 1000 + 1,
    DATEADD(DAY, -ABS(CHECKSUM(NEWID())) % 365, GETDATE()),
    CASE ABS(CHECKSUM(NEWID())) % 4 
        WHEN 0 THEN 'Active'
        WHEN 1 THEN 'Completed'
        WHEN 2 THEN 'Cancelled'
        ELSE 'Pending'
    END,
    CAST(ABS(CHECKSUM(NEWID())) % 1000 AS DECIMAL(10,2)),
    'Address ' + CAST(ABS(CHECKSUM(NEWID())) % 1000 AS NVARCHAR),
    'Product info ' + CAST(ABS(CHECKSUM(NEWID())) % 100 AS NVARCHAR)
FROM sys.all_objects a
CROSS JOIN sys.all_objects b;

-- Test without covering index
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT 
    ID,
    CustomerID,
    OrderDate,
    OrderStatus,
    OrderTotal,
    ShippingAddress
FROM PerformanceTest
WHERE CustomerID = 500
  AND OrderDate >= '2024-01-01'
  AND OrderStatus = 'Active'
ORDER BY OrderDate DESC;

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;

-- Create covering index
CREATE NONCLUSTERED INDEX IX_PerformanceTest_Covering
ON PerformanceTest(CustomerID, OrderDate, OrderStatus)
INCLUDE (ID, OrderTotal, ShippingAddress);

-- Test with covering index
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT 
    ID,
    CustomerID,
    OrderDate,
    OrderStatus,
    OrderTotal,
    ShippingAddress
FROM PerformanceTest
WHERE CustomerID = 500
  AND OrderDate >= '2024-01-01'
  AND OrderStatus = 'Active'
ORDER BY OrderDate DESC;

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
```

#### Filtered Indexes

Filtered indexes are optimized for queries that select a specific subset of data.

**Implementation Examples:**
```sql
-- Filtered index for active customers
CREATE NONCLUSTERED INDEX IX_Orders_ActiveStatus
ON Orders(CustomerID, OrderDate)
INCLUDE (OrderID, OrderTotal)
WHERE OrderStatus = 'Active';

-- Filtered index for high-value orders
CREATE NONCLUSTERED INDEX IX_Orders_HighValue
ON Orders(CustomerID, OrderDate)
INCLUDE (OrderID, OrderTotal)
WHERE OrderTotal >= 1000.00;

-- Filtered index for specific date range
CREATE NONCLUSTERED INDEX IX_Orders_2024
ON Orders(CustomerID, OrderDate)
INCLUDE (OrderID, OrderTotal)
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01';

-- Filtered index for archived data
CREATE NONCLUSTERED INDEX IX_Orders_Archived
ON Orders(CustomerID, OrderDate, OrderStatus)
INCLUDE (OrderID)
WHERE OrderStatus = 'Archived' AND OrderDate < DATEADD(YEAR, -2, GETDATE());
```

**Performance Benefits Analysis:**
```sql
-- Analyze filtered index usage and benefits
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    i.filter_definition AS FilterDefinition,
    s.user_seeks AS Seeks,
    s.user_scans AS Scans,
    s.user_lookups AS Lookups,
    s.user_updates AS Updates,
    s.last_user_seek AS LastSeek,
    (s.user_seeks + s.user_scans + s.user_lookups) AS TotalUsage,
    -- Calculate index effectiveness
    CASE 
        WHEN s.user_updates > 0 THEN CAST((s.user_seeks + s.user_scans + s.user_lookups) AS FLOAT) / s.user_updates
        ELSE 0 
    END AS UsageToUpdateRatio
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE i.has_filter = 1
AND OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
ORDER BY TotalUsage DESC;
```

### Index Fragmentation and Optimization

#### Understanding Fragmentation

**Internal Fragmentation:**
- Occurs when data pages are not fully utilized
- Caused by page splits and varying row sizes
- Results in wasted space and increased I/O

**External Fragmentation:**
- Occurs when logical order of pages doesn't match physical order
- Caused by page splits that create non-contiguous storage
- Results in slower sequential I/O operations

**Analyzing Fragmentation:**
```sql
-- Comprehensive index fragmentation analysis
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.index_type_desc AS IndexType,
    ips.avg_fragmentation_in_percent AS FragmentationPercent,
    ips.page_count AS PageCount,
    ips.avg_page_space_used_in_percent AS AvgSpaceUsedPercent,
    ips.record_count AS RecordCount,
    -- Fragmentation severity classification
    CASE 
        WHEN ips.avg_fragmentation_in_percent < 10 THEN 'Optimal - No action needed'
        WHEN ips.avg_fragmentation_in_percent < 30 THEN 'Moderate - Consider reorganize'
        WHEN ips.avg_fragmentation_in_percent < 50 THEN 'High - Rebuild recommended'
        ELSE 'Critical - Immediate rebuild required'
    END AS ActionRecommendation,
    -- Maintenance priority calculation
    CASE 
        WHEN ips.page_count > 10000 AND ips.avg_fragmentation_in_percent > 30 THEN 'High Priority'
        WHEN ips.page_count > 1000 AND ips.avg_fragmentation_in_percent > 20 THEN 'Medium Priority'
        WHEN ips.page_count > 100 AND ips.avg_fragmentation_in_percent > 15 THEN 'Low Priority'
        ELSE 'Monitor Only'
    END AS MaintenancePriority
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.index_id > 0 -- Exclude heaps
AND OBJECTPROPERTY(ips.object_id, 'IsUserTable') = 1
ORDER BY ips.avg_fragmentation_in_percent DESC, ips.page_count DESC;
```

#### Index Maintenance Procedures

**Reorganization vs Rebuild Decisions:**
```sql
-- Create procedure for intelligent index maintenance
CREATE OR ALTER PROCEDURE sp_MaintainIndexes
    @TableName NVARCHAR(128) = NULL,
    @RebuildThreshold FLOAT = 30.0,
    @ReorganizeThreshold FLOAT = 10.0,
    @OnlineRebuild BIT = 1,
    @MaxExecutionTime INT = 3600 -- 1 hour
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @IndexName NVARCHAR(128);
    DECLARE @IndexID INT;
    DECLARE @ObjectID INT;
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @StartTime DATETIME;
    DECLARE @ExecutionTime INT;
    DECLARE @RowCount INT = 0;
    
    -- Cursor for indexes that need maintenance
    DECLARE index_cursor CURSOR FOR
    SELECT 
        OBJECT_NAME(ips.object_id) AS TableName,
        i.name AS IndexName,
        ips.index_id,
        ips.object_id,
        ips.avg_fragmentation_in_percent,
        ips.page_count,
        ips.index_type_desc
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
    INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE ips.index_id > 0 -- Exclude heaps
    AND (OBJECTPROPERTY(ips.object_id, 'IsUserTable') = 1 OR OBJECTPROPERTY(ips.object_id, 'IsMsShipped') = 0)
    AND (@TableName IS NULL OR OBJECT_NAME(ips.object_id) = @TableName)
    AND ips.avg_fragmentation_in_percent > @ReorganizeThreshold
    ORDER BY ips.avg_fragmentation_in_percent DESC, ips.page_count DESC;
    
    OPEN index_cursor;
    FETCH NEXT FROM index_cursor INTO @TableName, @IndexName, @IndexID, @ObjectID, @Fragmentation, @PageCount, @IndexType;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @StartTime = GETDATE();
        SET @RowCount = @RowCount + 1;
        
        -- Determine maintenance action
        IF @Fragmentation > @RebuildThreshold AND @PageCount > 100
        BEGIN
            -- Rebuild index
            SET @SQL = 'ALTER INDEX ' + QUOTENAME(@IndexName) + ' ON ' + QUOTENAME(@TableName);
            
            IF @OnlineRebuild = 1 AND @IndexType != 'CLUSTERED COLUMNSTORE'
                SET @SQL = @SQL + ' REBUILD WITH (ONLINE = ON, MAXDOP = 2, FILLFACTOR = 90)';
            ELSE
                SET @SQL = @SQL + ' REBUILD WITH (MAXDOP = 2, FILLFACTOR = 90)';
            
            PRINT 'Rebuilding index: ' + @IndexName + ' on table: ' + @TableName;
            PRINT 'Current fragmentation: ' + CAST(@Fragmentation AS VARCHAR) + '%';
            
            BEGIN TRY
                EXEC sp_executesql @SQL;
                SET @ExecutionTime = DATEDIFF(SECOND, @StartTime, GETDATE());
                PRINT 'Rebuild completed in ' + CAST(@ExecutionTime AS VARCHAR) + ' seconds';
                
                -- Log maintenance activity
                INSERT INTO IndexMaintenanceLog (TableName, IndexName, ActionType, FragmentationBefore, ExecutionTimeSeconds, StartTime)
                VALUES (@TableName, @IndexName, 'REBUILD', @Fragmentation, @ExecutionTime, @StartTime);
                
            END TRY
            BEGIN CATCH
                PRINT 'Rebuild failed for ' + @IndexName + ': ' + ERROR_MESSAGE();
                
                -- Log failure
                INSERT INTO IndexMaintenanceLog (TableName, IndexName, ActionType, ErrorMessage, StartTime)
                VALUES (@TableName, @IndexName, 'REBUILD_FAILED', ERROR_MESSAGE(), @StartTime);
            END CATCH
        END
        ELSE IF @Fragmentation > @ReorganizeThreshold
        BEGIN
            -- Reorganize index
            SET @SQL = 'ALTER INDEX ' + QUOTENAME(@IndexName) + ' ON ' + QUOTENAME(@TableName) + ' REORGANIZE';
            
            IF @IndexType = 'NONCLUSTERED COLUMNSTORE'
                SET @SQL = @SQL + ' WITH (MAXDOP = 2)';
            
            PRINT 'Reorganizing index: ' + @IndexName + ' on table: ' + @TableName;
            PRINT 'Current fragmentation: ' + CAST(@Fragmentation AS VARCHAR) + '%';
            
            BEGIN TRY
                EXEC sp_executesql @SQL;
                SET @ExecutionTime = DATEDIFF(SECOND, @StartTime, GETDATE());
                PRINT 'Reorganize completed in ' + CAST(@ExecutionTime AS VARCHAR) + ' seconds';
                
                -- Log maintenance activity
                INSERT INTO IndexMaintenanceLog (TableName, IndexName, ActionType, FragmentationBefore, ExecutionTimeSeconds, StartTime)
                VALUES (@TableName, @IndexName, 'REORGANIZE', @Fragmentation, @ExecutionTime, @StartTime);
                
            END TRY
            BEGIN CATCH
                PRINT 'Reorganize failed for ' + @IndexName + ': ' + ERROR_MESSAGE();
                
                -- Log failure
                INSERT INTO IndexMaintenanceLog (TableName, IndexName, ActionType, ErrorMessage, StartTime)
                VALUES (@TableName, @IndexName, 'REORGANIZE_FAILED', ERROR_MESSAGE(), @StartTime);
            END CATCH
        END
        ELSE
        BEGIN
            PRINT 'No action needed for ' + @IndexName + ' - Fragmentation: ' + CAST(@Fragmentation AS VARCHAR) + '%';
        END
        
        -- Check execution time limit
        IF @ExecutionTime > @MaxExecutionTime
        BEGIN
            PRINT 'Maintenance stopped due to time limit exceeded';
            BREAK;
        END
        
        FETCH NEXT FROM index_cursor INTO @TableName, @IndexName, @IndexID, @ObjectID, @Fragmentation, @PageCount, @IndexType;
    END
    
    CLOSE index_cursor;
    DEALLOCATE index_cursor;
    
    PRINT 'Index maintenance completed. Processed ' + CAST(@RowCount AS VARCHAR) + ' indexes.';
    
    -- Return maintenance summary
    SELECT 
        ActionType,
        COUNT(*) as ActionCount,
        AVG(ExecutionTimeSeconds) as AvgExecutionTimeSeconds,
        SUM(CASE WHEN ErrorMessage IS NOT NULL THEN 1 ELSE 0 END) as ErrorCount
    FROM IndexMaintenanceLog
    WHERE CAST(StartTime AS DATE) = CAST(GETDATE() AS DATE)
    GROUP BY ActionType;
END;
GO

-- Create table to log index maintenance activities
CREATE TABLE IndexMaintenanceLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    TableName NVARCHAR(128),
    IndexName NVARCHAR(128),
    ActionType NVARCHAR(50),
    FragmentationBefore FLOAT,
    FragmentationAfter FLOAT,
    ExecutionTimeSeconds INT,
    ErrorMessage NVARCHAR(MAX),
    StartTime DATETIME DEFAULT GETDATE(),
    EndTime AS DATEADD(SECOND, ExecutionTimeSeconds, StartTime) PERSISTED
);

-- Grant execute permission
GRANT EXECUTE ON sp_MaintainIndexes TO [YourDbaUser];
```

**Automated Index Maintenance with PowerShell:**
```powershell
# Automated Index Maintenance Script
# Save as: Maintain-Indexes.ps1

param(
    [Parameter(Mandatory=$true)]
    [string]$ServerInstance,
    
    [Parameter(Mandatory=$false)]
    [string]$Database = 'master',
    
    [Parameter(Mandatory=$false)]
    [float]$RebuildThreshold = 30.0,
    
    [Parameter(Mandatory=$false)]
    [float]$ReorganizeThreshold = 10.0,
    
    [Parameter(Mandatory=$false)]
    [switch]$OnlineRebuild,
    
    [Parameter(Mandatory=$false)]
    [string]$LogPath = 'C:\Logs\IndexMaintenance',
    
    [Parameter(Mandatory=$false)]
    [int]$MaxExecutionTime = 3600
)

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
    
    # Ensure log directory exists
    if (-not (Test-Path $LogPath)) {
        New-Item -ItemType Directory -Path $LogPath -Force | Out-Null
    }
    
    # Write to daily log file
    $logFile = Join-Path $LogPath "IndexMaintenance_$(Get-Date -Format 'yyyyMMdd').log"
    Add-Content -Path $logFile -Value $logMessage
}

function Get-IndexMaintenanceReport {
    param([string]$Server, [string]$DbName)
    
    $query = @"
    SELECT 
        OBJECT_NAME(ips.object_id) AS TableName,
        i.name AS IndexName,
        ips.index_type_desc AS IndexType,
        ips.avg_fragmentation_in_percent AS FragmentationPercent,
        ips.page_count AS PageCount,
        ips.avg_page_space_used_in_percent AS AvgSpaceUsedPercent,
        ips.record_count AS RecordCount,
        CASE 
            WHEN ips.avg_fragmentation_in_percent < $ReorganizeThreshold THEN 'Monitor'
            WHEN ips.avg_fragmentation_in_percent < $RebuildThreshold THEN 'Reorganize'
            ELSE 'Rebuild'
        END AS RecommendedAction,
        CASE 
            WHEN ips.page_count > 10000 AND ips.avg_fragmentation_in_percent > $RebuildThreshold THEN 'High Priority'
            WHEN ips.page_count > 1000 AND ips.avg_fragmentation_in_percent > ($RebuildThreshold * 0.7) THEN 'Medium Priority'
            ELSE 'Low Priority'
        END AS Priority
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ips
    INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE ips.index_id > 0
    AND OBJECTPROPERTY(ips.object_id, 'IsUserTable') = 1
    AND ips.avg_fragmentation_in_percent > $ReorganizeThreshold
    ORDER BY ips.avg_fragmentation_in_percent DESC, ips.page_count DESC
"@
    
    try {
        $connectionString = "Server=$Server;Database=$DbName;Integrated Security=True;"
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
        Write-Log "Failed to get index maintenance report: $($_.Exception.Message)" 'ERROR'
        return $null
    }
}

function Start-IndexMaintenance {
    param(
        [string]$Server,
        [string]$DbName,
        [object]$MaintenanceReport,
        [bool]$PerformMaintenance = $true
    )
    
    $maintenanceResults = @()
    $totalStartTime = Get-Date
    
    foreach ($index in $MaintenanceReport) {
        $indexStartTime = Get-Date
        
        try {
            $action = switch ($index.RecommendedAction) {
                'Rebuild' { 'ALTER INDEX' }
                'Reorganize' { 'ALTER INDEX' }
                default { 'No Action' }
            }
            
            if ($action -ne 'No Action' -and $PerformMaintenance) {
                $sqlCommand = if ($index.RecommendedAction -eq 'Rebuild') {
                    $onlineOption = if ($OnlineRebuild -and $index.IndexType -ne 'CLUSTERED COLUMNSTORE') { ', ONLINE = ON' } else { '' }
                    "ALTER INDEX [$($index.IndexName)] ON [$($index.TableName)] REBUILD WITH (MAXDOP = 2, FILLFACTOR = 90$onlineOption)"
                }
                else {
                    "ALTER INDEX [$($index.IndexName)] ON [$($index.TableName)] REORGANIZE"
                }
                
                Write-Log "Performing $($index.RecommendedAction) on $($index.IndexName) ($($index.FragmentationPercent)% fragmentation)"
                
                $result = Invoke-Sqlcmd -ServerInstance $Server -Database $DbName -Query $sqlCommand -ErrorAction Stop
                
                $indexEndTime = Get-Date
                $duration = ($indexEndTime - $indexStartTime).TotalSeconds
                
                $maintenanceResults += [PSCustomObject]@{
                    TableName = $index.TableName
                    IndexName = $index.IndexName
                    Action = $index.RecommendedAction
                    BeforeFragmentation = $index.FragmentationPercent
                    DurationSeconds = $duration
                    Status = 'SUCCESS'
                    Priority = $index.Priority
                }
                
                Write-Log "$($index.RecommendedAction) completed for $($index.IndexName) in $duration seconds"
            }
            else {
                Write-Log "No action needed for $($index.IndexName) ($($index.FragmentationPercent)% fragmentation)"
                
                $maintenanceResults += [PSCustomObject]@{
                    TableName = $index.TableName
                    IndexName = $index.IndexName
                    Action = 'MONITOR'
                    BeforeFragmentation = $index.FragmentationPercent
                    DurationSeconds = 0
                    Status = 'SKIPPED'
                    Priority = $index.Priority
                }
            }
        }
        catch {
            $indexEndTime = Get-Date
            $duration = ($indexEndTime - $indexStartTime).TotalSeconds
            
            Write-Log "Maintenance failed for $($index.IndexName): $($_.Exception.Message)" 'ERROR'
            
            $maintenanceResults += [PSCustomObject]@{
                TableName = $index.TableName
                IndexName = $index.IndexName
                Action = $index.RecommendedAction
                BeforeFragmentation = $index.FragmentationPercent
                DurationSeconds = $duration
                Status = 'FAILED'
                Error = $_.Exception.Message
                Priority = $index.Priority
            }
        }
        
        # Check total execution time
        $totalDuration = (Get-Date - $totalStartTime).TotalSeconds
        if ($totalDuration -gt $MaxExecutionTime) {
            Write-Log "Maintenance stopped due to time limit exceeded"
            break
        }
    }
    
    return $maintenanceResults
}

function New-MaintenanceReport {
    param([object]$Results, [string]$ReportPath)
    
    $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    $reportFile = Join-Path $ReportPath "IndexMaintenance_Report_$timestamp.html"
    
    $totalIndices = $Results.Count
    $successfulMaintenance = ($Results | Where-Object { $_.Status -eq 'SUCCESS' }).Count
    $failedMaintenance = ($Results | Where-Object { $_.Status -eq 'FAILED' }).Count
    $skippedMaintenance = ($Results | Where-Object { $_.Status -eq 'SKIPPED' }).Count
    
    $htmlReport = @"
<!DOCTYPE html>
<html>
<head>
    <title>Index Maintenance Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background-color: #f0f0f0; padding: 10px; border-bottom: 2px solid #ccc; }
        .summary { margin: 20px 0; padding: 15px; border: 1px solid #ddd; background-color: #f9f9f9; }
        table { border-collapse: collapse; width: 100%; margin: 20px 0; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .success { color: green; }
        .failed { color: red; }
        .skipped { color: orange; }
        .high-priority { background-color: #ffe6e6; }
        .medium-priority { background-color: #fff3e6; }
        .low-priority { background-color: #e6f3ff; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Index Maintenance Report</h1>
        <p>Generated: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')</p>
        <p>Server: $ServerInstance</p>
        <p>Database: $Database</p>
    </div>
    
    <div class="summary">
        <h2>Executive Summary</h2>
        <p><strong>Total Indices Processed:</strong> $totalIndices</p>
        <p class="success"><strong>Successful Maintenance:</strong> $successfulMaintenance</p>
        <p class="failed"><strong>Failed Maintenance:</strong> $failedMaintenance</p>
        <p class="skipped"><strong>Skipped (Monitoring):</strong> $skippedMaintenance</p>
        <p><strong>Success Rate:</strong> $([math]::Round(($successfulMaintenance / $totalIndices) * 100, 2))%</p>
    </div>
    
    <table>
        <tr>
            <th>Table Name</th>
            <th>Index Name</th>
            <th>Action</th>
            <th>Priority</th>
            <th>Before Fragmentation %</th>
            <th>Duration (sec)</th>
            <th>Status</th>
            <th>Error</th>
        </tr>
"@
    
    foreach ($result in $Results) {
        $priorityClass = switch ($result.Priority) {
            'High Priority' { 'high-priority' }
            'Medium Priority' { 'medium-priority' }
            default { 'low-priority' }
        }
        
        $statusClass = switch ($result.Status) {
            'SUCCESS' { 'success' }
            'FAILED' { 'failed' }
            default { 'skipped' }
        }
        
        $htmlReport += @"
        <tr class="$priorityClass">
            <td>$($result.TableName)</td>
            <td>$($result.IndexName)</td>
            <td>$($result.Action)</td>
            <td>$($result.Priority)</td>
            <td>$([math]::Round($result.BeforeFragmentation, 2))</td>
            <td>$([math]::Round($result.DurationSeconds, 2))</td>
            <td class="$statusClass">$($result.Status)</td>
            <td>$($result.Error)</td>
        </tr>
"@
    }
    
    $htmlReport += @"
    </table>
</body>
</html>
"@
    
    $htmlReport | Out-File -FilePath $reportFile -Encoding UTF8
    return $reportFile
}

# Main execution
try {
    Write-Log "Starting index maintenance for $ServerInstance"
    
    # Get index maintenance report
    $maintenanceReport = Get-IndexMaintenanceReport -Server $ServerInstance -DbName $Database
    
    if (-not $maintenanceReport -or $maintenanceReport.Rows.Count -eq 0) {
        Write-Log "No indices require maintenance"
        exit 0
    }
    
    Write-Log "Found $($maintenanceReport.Rows.Count) indices requiring maintenance"
    
    # Display maintenance recommendations
    Write-Log "Maintenance Recommendations:"
    $maintenanceReport | ForEach-Object {
        Write-Log "  - $($_.TableName).$($_.IndexName): $($_.RecommendedAction) ($($_.FragmentationPercent)% fragmentation) - $($_.Priority)"
    }
    
    # Perform maintenance
    $results = Start-IndexMaintenance -Server $ServerInstance -DbName $Database -MaintenanceReport $maintenanceReport
    
    # Generate report
    $reportFile = New-MaintenanceReport -Results $results -ReportPath $LogPath
    
    Write-Log "Index maintenance completed. Report saved to: $reportFile"
    
    # Summary
    $totalProcessed = $results.Count
    $successful = ($results | Where-Object { $_.Status -eq 'SUCCESS' }).Count
    $failed = ($results | Where-Object { $_.Status -eq 'FAILED' }).Count
    
    Write-Log "=== Index Maintenance Summary ==="
    Write-Log "Total processed: $totalProcessed"
    Write-Log "Successful: $successful"
    Write-Log "Failed: $failed"
    
    if ($failed -gt 0) {
        Write-Log "Failed indices:" 'WARNING'
        $results | Where-Object { $_.Status -eq 'FAILED' } | ForEach-Object {
            Write-Log "  - $($_.TableName).$($_.IndexName): $($_.Error)" 'ERROR'
        }
    }
}
catch {
    Write-Log "Index maintenance failed: $($_.Exception.Message)" 'ERROR'
    exit 1
}
```

## Wednesday - Advanced Indexing Strategies

### Columnstore Index Optimization

#### Clustered Columnstore Index Best Practices

**Data Loading Considerations:**
```sql
-- Create table optimized for columnstore
CREATE TABLE SalesFact_CL (
    SaleID BIGINT,
    DateKey INT NOT NULL,
    ProductKey INT NOT NULL,
    CustomerKey INT NOT NULL,
    StoreKey INT NOT NULL,
    SalesAmount DECIMAL(18,2) NOT NULL,
    Quantity INT NOT NULL,
    DiscountAmount DECIMAL(10,2) NOT NULL,
    ProfitMargin DECIMAL(5,2),
    -- Add additional columns for data quality
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    ModifiedDate DATETIME2,
    IsDeleted BIT DEFAULT 0,
    DataQualityScore DECIMAL(3,2) DEFAULT 1.0
);

-- Create clustered columnstore index with compression options
CREATE CLUSTERED COLUMNSTORE INDEX IX_ClusteredColumnstore_Sales_CL
ON SalesFact_CL
WITH (
    DATA_COMPRESSION = COLUMNSTORE, -- Basic compression
    MAXDOP = 2, -- Limit parallelism for better compression
    -- Optional: Archive compression for rarely accessed data
    -- DATA_COMPRESSION = COLUMNSTORE_ARCHIVE
);

-- Create nonclustered indexes on clustered columnstore for OLTP operations
CREATE NONCLUSTERED INDEX IX_Sales_CL_DateKey
ON SalesFact_CL(DateKey)
INCLUDE (SaleID, SalesAmount, Quantity);

CREATE NONCLUSTERED INDEX IX_Sales_CL_ProductKey
ON SalesFact_CL(ProductKey)
INCLUDE (SaleID, SalesAmount, ProfitMargin);

CREATE NONCLUSTERED INDEX IX_Sales_CL_CustomerKey
ON SalesFact_CL(CustomerKey)
INCLUDE (SaleID, SalesAmount, Quantity, DiscountAmount);

-- Nonclustered columnstore index for mixed workloads
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_Sales_CL_NCCI
ON SalesFact_CL(SaleID, DateKey, ProductKey, CustomerKey, SalesAmount, Quantity)
WITH (DATA_COMPRESSION = COLUMNSTORE);
```

**Bulk Loading into Columnstore:**
```sql
-- Staging table for efficient columnstore loading
CREATE TABLE SalesStaging (
    SaleID BIGINT,
    DateKey INT,
    ProductKey INT,
    CustomerKey INT,
    StoreKey INT,
    SalesAmount DECIMAL(18,2),
    Quantity INT,
    DiscountAmount DECIMAL(10,2),
    CONSTRAINT PK_SalesStaging PRIMARY KEY CLUSTERED (SaleID)
);

-- Load data into staging table
INSERT INTO SalesStaging (SaleID, DateKey, ProductKey, CustomerKey, StoreKey, SalesAmount, Quantity, DiscountAmount)
SELECT 
    SaleID, DateKey, ProductKey, CustomerKey, StoreKey, SalesAmount, Quantity, DiscountAmount
FROM ExternalDataSource;

-- Load into columnstore with minimal logging
ALTER TABLE SalesFact_CL SWITCH IN SalesStaging;

-- Verify columnstore segment elimination
-- This query shows which segments exist
SELECT 
    CONVERT(INT, $partition.numbers.n-1) AS partition_number,
    CAST(MIN($segment_id) AS BIGINT) AS min_segment_id,
    CAST(MAX($segment_id) AS BIGINT) AS max_segment_id,
    COUNT(*) AS segment_count
FROM salesfact_CL
CROSS JOIN numbers
WHERE numbers.n BETWEEN 1 AND $partition.maximum_partition_number
GROUP BY $partition.numbers.n
ORDER BY $partition.numbers.n;
```

#### Nonclustered Columnstore Index Optimization

**Memory-Optimized Workloads:**
```sql
-- Memory-optimized table with columnstore index
CREATE TABLE SalesFact_Memory (
    SaleID BIGINT NOT NULL PRIMARY KEY NONCLUSTERED HASH (SaleID) WITH (BUCKET_COUNT = 1000000),
    DateKey INT NOT NULL,
    ProductKey INT NOT NULL,
    CustomerKey INT NOT NULL,
    StoreKey INT NOT NULL,
    SalesAmount DECIMAL(18,2) NOT NULL,
    Quantity INT NOT NULL,
    DiscountAmount DECIMAL(10,2) NOT NULL,
    INDEX IX_Memory_Columnstore NONCLUSTERED COLUMNSTORE
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);

-- Insert data with minimal logging
INSERT INTO SalesFact_Memory 
SELECT TOP 100000
    SaleID, DateKey, ProductKey, CustomerKey, StoreKey, SalesAmount, Quantity, DiscountAmount
FROM SourceTable
WITH (TABLOCK);
```

**Columnstore Segment Elimination:**
```sql
-- Demonstrate segment elimination effectiveness
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Query that benefits from segment elimination
SELECT 
    ProductKey,
    SUM(SalesAmount) as TotalSales,
    AVG(Quantity) as AvgQuantity,
    COUNT(*) as TransactionCount,
    SUM(SalesAmount * Quantity) as GrossRevenue
FROM SalesFact_CL
WHERE DateKey BETWEEN 20240101 AND 20241231
  AND ProductKey IN (SELECT ProductKey FROM ProductCategory WHERE CategoryName = 'Electronics')
GROUP BY ProductKey
ORDER BY TotalSales DESC;

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;

-- Analyze segment elimination
SELECT 
    p.name AS PartitionName,
    fg.name AS FileGroupName,
    ps.partition_number,
    ps.row_count,
    ps.data_compression_desc,
    ps.distribution_desc,
    -- Columnstore specific metrics
    cs.columnstore_segments_read,
    cs.columnstore_segments_skipped,
    cs.columnstore_segments_scanned
FROM sys.dm_db_partition_stats ps
INNER JOIN sys.partitions p ON ps.partition_id = p.partition_id
INNER JOIN sys.allocation_units au ON ps.partition_id = au.container_id
INNER JOIN sys.filegroups fg ON au.data_space_id = fg.data_space_id
LEFT JOIN sys.dm_db_column_store_row_group_physical_stats cs ON ps.object_id = cs.object_id AND ps.index_id = cs.index_id AND ps.partition_number = cs.partition_number
WHERE ps.object_id = OBJECT_ID('SalesFact_CL')
ORDER BY ps.partition_number;
```

### Index Tuning for Specific Query Patterns

#### Optimizing JOIN Operations

**Clustered Index for Primary JOIN Keys:**
```sql
-- Create table optimized for JOIN operations
CREATE TABLE Customers (
    CustomerID INT PRIMARY KEY CLUSTERED,
    CustomerName NVARCHAR(200),
    Email NVARCHAR(200),
    CustomerSegment NVARCHAR(50),
    CreatedDate DATETIME,
    ModifiedDate DATETIME,
    IsActive BIT
);

CREATE TABLE Orders (
    OrderID INT PRIMARY KEY CLUSTERED,
    CustomerID INT NOT NULL,
    OrderDate DATETIME,
    OrderStatus NVARCHAR(50),
    OrderTotal DECIMAL(18,2),
    INDEX IX_Orders_CustomerID CLUSTERED (CustomerID) -- Optimizes JOINs
);

-- Create covering index for JOIN with SELECT
CREATE NONCLUSTERED INDEX IX_Orders_Covering_Customer
ON Orders(CustomerID, OrderDate, OrderStatus)
INCLUDE (OrderID, OrderTotal)
WITH (ONLINE = ON);

-- Query that benefits from optimized indexes
SELECT 
    c.CustomerID,
    c.CustomerName,
    c.CustomerSegment,
    COUNT(o.OrderID) as OrderCount,
    SUM(o.OrderTotal) as TotalSales,
    AVG(o.OrderTotal) as AvgOrderValue
FROM Customers c
INNER JOIN Orders o ON c.CustomerID = o.CustomerID
WHERE c.IsActive = 1
  AND o.OrderDate >= DATEADD(YEAR, -1, GETDATE())
  AND o.OrderStatus = 'Completed'
GROUP BY c.CustomerID, c.CustomerName, c.CustomerSegment
HAVING COUNT(o.OrderID) >= 5
ORDER BY TotalSales DESC;
```

#### Range Query Optimization

**B-Tree Optimization for Range Queries:**
```sql
-- Table designed for range queries on dates
CREATE TABLE EventLog (
    EventID BIGINT IDENTITY(1,1) PRIMARY KEY NONCLUSTERED,
    EventDateTime DATETIME2 NOT NULL,
    EventType NVARCHAR(50) NOT NULL,
    UserID INT,
    EventDescription NVARCHAR(MAX),
    SeverityLevel INT,
    -- Clustered index on date for range queries
    INDEX IX_EventLog_DateClustered CLUSTERED (EventDateTime, EventType),
    -- Nonclustered index for specific event types
    INDEX IX_EventLog_TypeDate NONCLUSTERED (EventType, EventDateTime),
    -- Filtered index for error events
    INDEX IX_EventLog_Errors NONCLUSTERED (EventDateTime, UserID)
    WHERE SeverityLevel >= 4
);

-- Optimized range queries
-- Query 1: Time range with event type filter
SELECT *
FROM EventLog
WHERE EventDateTime >= DATEADD(DAY, -7, GETDATE())
  AND EventDateTime < GETDATE()
  AND EventType = 'UserLogin'
ORDER BY EventDateTime DESC;

-- Query 2: Error events in time range (uses filtered index)
SELECT EventDateTime, EventDescription, UserID
FROM EventLog
WHERE EventDateTime >= DATEADD(HOUR, -24, GETDATE())
  AND SeverityLevel >= 4
ORDER BY EventDateTime DESC;

-- Query 3: Event statistics by type and time
SELECT 
    EventType,
    DATEADD(HOUR, DATEDIFF(HOUR, 0, EventDateTime), 0) as HourBucket,
    COUNT(*) as EventCount,
    COUNT(DISTINCT UserID) as UniqueUsers
FROM EventLog
WHERE EventDateTime >= DATEADD(DAY, -1, GETDATE())
GROUP BY EventType, DATEADD(HOUR, DATEDIFF(HOUR, 0, EventDateTime), 0)
ORDER BY EventCount DESC;
```

### Advanced Index Maintenance

#### Online Index Operations

**Implementing Online Rebuilds:**
```sql
-- Create stored procedure for online index maintenance
CREATE OR ALTER PROCEDURE sp_OnlineIndexMaintenance
    @DatabaseName NVARCHAR(128) = NULL,
    @TableName NVARCHAR(128) = NULL,
    @IndexName NVARCHAR(128) = NULL,
    @RebuildThreshold FLOAT = 30.0,
    @MaxExecutionTime INT = 7200 -- 2 hours
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @ObjectID INT;
    DECLARE @IndexID INT;
    DECLARE @StartTime DATETIME;
    DECLARE @ExecutionTime INT;
    DECLARE @IsOnlineRequired BIT = 1;
    
    -- Set database context
    IF @DatabaseName IS NOT NULL
        USE [master];
    
    -- Build dynamic SQL for online index rebuilds
    SET @SQL = '
    SELECT 
        t.object_id,
        i.index_id,
        t.name as TableName,
        i.name as IndexName,
        ips.avg_fragmentation_in_percent,
        ips.page_count,
        i.type_desc as IndexType,
        i.fill_factor
    FROM sys.tables t
    INNER JOIN sys.indexes i ON t.object_id = i.object_id
    INNER JOIN sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, ''DETAILED'') ips 
        ON i.object_id = ips.object_id AND i.index_id = ips.index_id
    WHERE t.is_ms_shipped = 0
    AND i.index_id > 0 -- Exclude heaps';
    
    IF @TableName IS NOT NULL
        SET @SQL = @SQL + ' AND t.name = ''' + @TableName + '''';
    
    IF @IndexName IS NOT NULL
        SET @SQL = @SQL + ' AND i.name = ''' + @IndexName + '''';
    
    SET @SQL = @SQL + ' AND ips.avg_fragmentation_in_percent > ' + CAST(@RebuildThreshold AS VARCHAR);
    
    -- Cursor for indexes to rebuild
    DECLARE rebuild_cursor CURSOR FOR
    SELECT 
        OBJECT_ID(QUOTENAME(@TableName) + '.' + QUOTENAME(@IndexName)),
        INDEX_ID,
        @TableName,
        @IndexName,
        0, -- Will be calculated in loop
        0, -- Will be calculated in loop
        '',
        90 -- Default fill factor
    
    OPEN rebuild_cursor;
    FETCH NEXT FROM rebuild_cursor INTO @ObjectID, @IndexID, @TableName, @IndexName, @StartTime, @ExecutionTime, @SQL, @SQL;
    
    -- Note: In a real implementation, you would use dynamic SQL to handle variable table/index names
    -- This is a simplified example showing the concept
    
    CLOSE rebuild_cursor;
    DEALLOCATE rebuild_cursor;
END;
GO

-- Example of online index operations
-- Online rebuild for nonclustered indexes
ALTER INDEX IX_Orders_CustomerID ON Orders
REBUILD WITH (ONLINE = ON, MAXDOP = 2, FILLFACTOR = 90);

-- Online rebuild for clustered index (SQL Server 2019+)
ALTER INDEX PK_Orders ON Orders
REBUILD WITH (ONLINE = ON, MAXDOP = 2, FILLFACTOR = 90);

-- Online create for new indexes
CREATE NONCLUSTERED INDEX IX_Orders_ProductDate
ON Orders(ProductID, OrderDate)
INCLUDE (OrderID, Quantity, UnitPrice)
WITH (ONLINE = ON, FILLFACTOR = 90);

-- Online drop and create for index modifications
CREATE NONCLUSTERED INDEX IX_Orders_NewCovering
ON Orders(CustomerID, OrderDate, OrderStatus)
INCLUDE (OrderID, OrderTotal, ShippingAddress)
WITH (ONLINE = ON, DROP_EXISTING = ON, FILLFACTOR = 90);
```

## Thursday - Index Performance Monitoring and Analysis

### Index Usage Analysis

#### Real-Time Index Usage Monitoring

**Index Usage Statistics Query:**
```sql
-- Comprehensive index usage analysis
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc AS IndexType,
    i.is_primary_key AS IsPrimaryKey,
    i.is_unique AS IsUnique,
    -- Usage statistics
    s.user_seeks AS UserSeeks,
    s.user_scans AS UserScans,
    s.user_lookups AS UserLookups,
    s.user_updates AS UserUpdates,
    (s.user_seeks + s.user_scans + s.user_lookups) AS TotalReads,
    s.last_user_seek AS LastSeek,
    s.last_user_scan AS LastScan,
    s.last_user_lookup AS LastLookup,
    s.last_user_update AS LastUpdate,
    -- Index effectiveness metrics
    CASE 
        WHEN s.user_updates = 0 THEN 
            CASE WHEN (s.user_seeks + s.user_scans + s.user_lookups) > 0 THEN 'EXCELLENT' ELSE 'UNUSED' END
        WHEN (s.user_seeks + s.user_scans + s.user_lookups) = 0 THEN 'WRITE_ONLY'
        ELSE 'MIXED'
    END AS UsageCategory,
    -- Maintenance priority
    CASE 
        WHEN s.user_updates > (s.user_seeks + s.user_scans + s.user_lookups) * 2 THEN 'HIGH_MAINTENANCE'
        WHEN s.user_seeks + s.user_scans + s.user_lookups > s.user_updates THEN 'HIGH_VALUE'
        ELSE 'BALANCED'
    END AS MaintenancePriority
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
AND s.database_id = DB_ID()
ORDER BY (s.user_seeks + s.user_scans + s.user_lookups) DESC, s.user_updates;
```

**Index Missing Recommendations:**
```sql
-- Identify missing indexes based on query execution plans
SELECT 
    migs.avg_user_impact,
    migs.avg_total_user_cost,
    migs.user_seeks,
    migs.user_scans,
    migs.avg_user_impact * (migs.user_seeks + migs.user_scans) AS TotalImpact,
    mid.statement AS TableName,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    mid.column_store_columns,
    mid.column_store_columns_not_in_key,
    mid.is_unique,
    mid.is_ignore_dup_key,
    mid.is_disabled,
    -- Recommendation score
    (migs.avg_user_impact * (migs.user_seeks + migs.user_scans)) AS ImpactScore
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE mid.database_id = DB_ID()
ORDER BY ImpactScore DESC;

-- Generate CREATE INDEX statements for missing indexes
SELECT 
    'CREATE NONCLUSTERED INDEX IX_' + 
    REPLACE(mid.statement, '.', '_') + '_' + 
    REPLACE(REPLACE(REPLACE(mid.equality_columns + mid.inequality_columns, ',', '_'), ' ', '_'), '(', '') + 
    ' ON ' + mid.statement + 
    '(' + ISNULL(mid.equality_columns + ', ', '') + ISNULL(mid.inequality_columns, '') + ')' +
    CASE WHEN mid.included_columns IS NOT NULL THEN 
        ' INCLUDE (' + mid.included_columns + ')' 
    ELSE '' END +
    ' WITH (ONLINE = ON, FILLFACTOR = 90, MAXDOP = 2);' AS CreateIndexScript,
    migs.avg_user_impact,
    migs.user_seeks,
    migs.user_scans,
    (migs.avg_user_impact * (migs.user_seeks + migs.user_scans)) AS ImpactScore
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE mid.database_id = DB_ID()
AND (migs.user_seeks + migs.user_scans) > 10 -- Only indexes with significant usage
ORDER BY ImpactScore DESC;
```

### Performance Impact Analysis

#### Query Performance Before/After Index Implementation

**Creating Performance Baseline:**
```sql
-- Create table to track index performance impact
CREATE TABLE IndexPerformanceBaseline (
    BaselineID INT IDENTITY(1,1) PRIMARY KEY,
    TestName NVARCHAR(100),
    QueryText NVARCHAR(MAX),
    ExecutionTimeBefore FLOAT,
    LogicalReadsBefore BIGINT,
    PhysicalReadsBefore BIGINT,
    ExecutionTimeAfter FLOAT,
    LogicalReadsAfter BIGINT,
    PhysicalReadsAfter BIGINT,
    IndexChanges NVARCHAR(MAX),
    ImprovementPercentage FLOAT,
    TestDate DATETIME DEFAULT GETDATE()
);

-- Stored procedure to measure query performance
CREATE OR ALTER PROCEDURE sp_MeasureQueryPerformance
    @TestName NVARCHAR(100),
    @Query NVARCHAR(MAX),
    @IndexChanges NVARCHAR(MAX) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    SET STATISTICS IO ON;
    SET STATISTICS TIME ON;
    
    DECLARE @StartTimeBefore DATETIME;
    DECLARE @EndTimeBefore DATETIME;
    DECLARE @StartTimeAfter DATETIME;
    DECLARE @EndTimeAfter DATETIME;
    DECLARE @ExecutionTimeBefore FLOAT;
    DECLARE @ExecutionTimeAfter FLOAT;
    DECLARE @LogicalReadsBefore BIGINT;
    DECLARE @LogicalReadsAfter BIGINT;
    DECLARE @PhysicalReadsBefore BIGINT;
    DECLARE @PhysicalReadsAfter BIGINT;
    
    -- Capture before metrics
    PRINT '=== BEFORE INDEX CHANGES ===';
    PRINT 'Test: ' + @TestName;
    PRINT 'Query: ' + @Query;
    PRINT 'Starting performance measurement...';
    
    SET @StartTimeBefore = GETDATE();
    
    BEGIN TRY
        EXEC sp_executesql @Query;
        SET @EndTimeBefore = GETDATE();
        SET @ExecutionTimeBefore = DATEDIFF(MILLISECOND, @StartTimeBefore, @EndTimeBefore);
    END TRY
    BEGIN CATCH
        PRINT 'Query execution failed before: ' + ERROR_MESSAGE();
        SET @ExecutionTimeBefore = -1;
    END CATCH
    
    -- Wait a moment to ensure clean separation
    WAITFOR DELAY '00:00:01';
    
    -- Simulate or apply index changes (in practice, this would be done externally)
    IF @IndexChanges IS NOT NULL
    BEGIN
        PRINT 'Applying index changes: ' + @IndexChanges;
        -- This would contain the actual index creation/modification statements
        EXEC sp_executesql @IndexChanges;
    END
    
    -- Wait to allow SQL Server to compile new execution plans
    WAITFOR DELAY '00:00:05';
    
    -- Clear procedure cache for consistent testing
    DBCC DROPCLEANBUFFERS;
    DBCC FREEPROCCACHE;
    
    -- Capture after metrics
    PRINT '=== AFTER INDEX CHANGES ===';
    PRINT 'Measuring performance after changes...';
    
    SET @StartTimeAfter = GETDATE();
    
    BEGIN TRY
        EXEC sp_executesql @Query;
        SET @EndTimeAfter = GETDATE();
        SET @ExecutionTimeAfter = DATEDIFF(MILLISECOND, @StartTimeAfter, @EndTimeAfter);
    END TRY
    BEGIN CATCH
        PRINT 'Query execution failed after: ' + ERROR_MESSAGE();
        SET @ExecutionTimeAfter = -1;
    END CATCH
    
    SET STATISTICS IO OFF;
    SET STATISTICS TIME OFF;
    
    -- Calculate improvement
    DECLARE @ImprovementPercentage FLOAT = 0;
    IF @ExecutionTimeBefore > 0 AND @ExecutionTimeAfter > 0
    BEGIN
        SET @ImprovementPercentage = ((@ExecutionTimeBefore - @ExecutionTimeAfter) / @ExecutionTimeBefore) * 100;
    END
    
    -- Store results
    INSERT INTO IndexPerformanceBaseline (
        TestName, QueryText, ExecutionTimeBefore, ExecutionTimeAfter,
        IndexChanges, ImprovementPercentage
    ) VALUES (
        @TestName, @Query, @ExecutionTimeBefore, @ExecutionTimeAfter,
        @IndexChanges, @ImprovementPercentage
    );
    
    -- Display results
    PRINT '=== PERFORMANCE COMPARISON RESULTS ===';
    PRINT 'Test Name: ' + @TestName;
    PRINT 'Execution Time Before: ' + CAST(@ExecutionTimeBefore AS VARCHAR) + ' ms';
    PRINT 'Execution Time After: ' + CAST(@ExecutionTimeAfter AS VARCHAR) + ' ms';
    PRINT 'Improvement: ' + CAST(@ImprovementPercentage AS VARCHAR) + '%';
    PRINT 'Improvement Status: ' + 
        CASE 
            WHEN @ImprovementPercentage > 20 THEN 'SIGNIFICANT IMPROVEMENT'
            WHEN @ImprovementPercentage > 0 THEN 'MODERATE IMPROVEMENT'
            WHEN @ImprovementPercentage = 0 THEN 'NO CHANGE'
            ELSE 'PERFORMANCE REGRESSION'
        END;
END;
GO

-- Example usage
DECLARE @TestQuery NVARCHAR(MAX) = N'
SELECT 
    c.CustomerID,
    c.CustomerName,
    COUNT(o.OrderID) as OrderCount,
    SUM(o.OrderTotal) as TotalSales
FROM Customers c
INNER JOIN Orders o ON c.CustomerID = o.CustomerID
WHERE c.CustomerSegment = ''Premium''
  AND o.OrderDate >= DATEADD(MONTH, -6, GETDATE())
GROUP BY c.CustomerID, c.CustomerName
ORDER BY TotalSales DESC
OPTION (MAXDOP 1); -- Use MAXDOP 1 for consistent baseline testing';

DECLARE @IndexChanges NVARCHAR(MAX) = N'
CREATE NONCLUSTERED INDEX IX_Orders_CustomerDateCovering
ON Orders(CustomerID, OrderDate)
INCLUDE (OrderID, OrderTotal)
WITH (ONLINE = ON, FILLFACTOR = 90);';

EXEC sp_MeasureQueryPerformance 
    @TestName = 'Customer Sales Analysis Query',
    @Query = @TestQuery,
    @IndexChanges = @IndexChanges;
```

#### Index Usage Monitoring Dashboard

**Creating Monitoring Views:**
```sql
-- Create comprehensive index monitoring view
CREATE VIEW vw_IndexUsageSummary AS
SELECT 
    DB_NAME() as DatabaseName,
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc AS IndexType,
    i.is_primary_key AS IsPrimaryKey,
    i.is_unique AS IsUnique,
    i.fill_factor AS FillFactor,
    -- Usage statistics
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates,
    (s.user_seeks + s.user_scans + s.user_lookups) AS TotalReads,
    s.last_user_seek,
    s.last_user_scan,
    s.last_user_lookup,
    s.last_user_update,
    -- Fragmentation information
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.record_count,
    -- Classification
    CASE 
        WHEN s.user_updates = 0 AND (s.user_seeks + s.user_scans + s.user_lookups) > 0 THEN 'READ_ONLY'
        WHEN s.user_seeks + s.user_scans + s.user_lookups = 0 THEN 'UNUSED'
        WHEN s.user_updates > (s.user_seeks + s.user_scans + s.user_lookups) * 3 THEN 'HIGH_MAINTENANCE'
        WHEN (s.user_seeks + s.user_scans + s.user_lookups) > s.user_updates * 2 THEN 'HIGH_VALUE'
        ELSE 'BALANCED'
    END AS UsageCategory,
    -- Maintenance recommendations
    CASE 
        WHEN ips.avg_fragmentation_in_percent > 30 AND ips.page_count > 1000 THEN 'REBUILD_REQUIRED'
        WHEN ips.avg_fragmentation_in_percent > 10 AND ips.page_count > 100 THEN 'REORGANIZE_REQUIRED'
        WHEN s.user_seeks + s.user_scans + s.user_lookups = 0 THEN 'CONSIDER_DROP'
        ELSE 'MAINTENANCE_OK'
    END AS MaintenanceAction,
    -- Last maintenance check
    ISNULL(iml.LastMaintenanceDate, 'NEVER') as LastMaintenanceDate
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
LEFT JOIN sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips 
    ON i.object_id = ips.object_id AND i.index_id = ips.index_id
LEFT JOIN IndexMaintenanceLog iml ON i.name = iml.IndexName AND i.object_id = OBJECT_ID(iml.TableName)
WHERE OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
AND s.database_id = DB_ID();

-- Query for daily index usage report
SELECT 
    TableName,
    IndexName,
    IndexType,
    TotalReads,
    UserUpdates,
    UsageCategory,
    MaintenanceAction,
    avg_fragmentation_in_percent,
    page_count,
    LastMaintenanceDate,
    CASE 
        WHEN TotalReads = 0 THEN 'INDEX_MAY_BE_DROPPABLE'
        WHEN avg_fragmentation_in_percent > 30 THEN 'HIGH_FRAGMENTATION'
        WHEN UserUpdates > TotalReads * 2 THEN 'HIGH_WRITE_OVERHEAD'
        ELSE 'HEALTHY'
    END AS OverallStatus
FROM vw_IndexUsageSummary
ORDER BY TotalReads DESC, page_count DESC;

-- Identify indexes for optimization
SELECT 
    TableName,
    IndexName,
    UsageCategory,
    TotalReads,
    UserUpdates,
    avg_fragmentation_in_percent,
    MaintenanceAction,
    -- Recommendations
    CASE 
        WHEN TotalReads = 0 AND UserUpdates > 0 THEN 
            'DROP: Index is write-only - consider dropping to reduce maintenance overhead'
        WHEN avg_fragmentation_in_percent > 50 THEN 
            'REBUILD: Critical fragmentation - immediate rebuild required'
        WHEN avg_fragmentation_in_percent > 30 THEN 
            'REBUILD: High fragmentation - scheduled rebuild recommended'
        WHEN avg_fragmentation_in_percent > 10 THEN 
            'REORGANIZE: Moderate fragmentation - reorganize during maintenance window'
        WHEN TotalReads < UserUpdates * 0.1 THEN 
            'REVIEW: Low usage relative to maintenance cost - evaluate business need'
        ELSE 
            'HEALTHY: No action required'
    END AS Recommendation
FROM vw_IndexUsageSummary
WHERE MaintenanceAction != 'MAINTENANCE_OK'
ORDER BY avg_fragmentation_in_percent DESC, page_count DESC;
```

### PowerShell Performance Monitoring

```powershell
# Index Performance Monitoring Script
# Save as: Monitor-IndexPerformance.ps1

param(
    [Parameter(Mandatory=$true)]
    [string]$ServerInstance,
    
    [Parameter(Mandatory=$false)]
    [string]$Database = 'master',
    
    [Parameter(Mandatory=$false)]
    [string]$ReportPath = 'C:\Reports\IndexPerformance',
    
    [Parameter(Mandatory=$false)]
    [int]$FragmentationThreshold = 30,
    
    [Parameter(Mandatory=$false)]
    [int]$UsageThreshold = 100,
    
    [Parameter(Mandatory=$false)]
    [switch]$EmailAlerts
)

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
    
    if (-not (Test-Path $ReportPath)) {
        New-Item -ItemType Directory -Path $ReportPath -Force | Out-Null
    }
    
    $logFile = Join-Path $ReportPath "IndexPerformance_$(Get-Date -Format 'yyyyMMdd').log"
    Add-Content -Path $logFile -Value $logMessage
}

function Get-IndexPerformanceData {
    param([string]$Server, [string]$DbName)
    
    $query = @"
    SELECT 
        OBJECT_NAME(s.object_id) AS TableName,
        i.name AS IndexName,
        i.type_desc AS IndexType,
        i.fill_factor AS FillFactor,
        -- Usage statistics
        s.user_seeks,
        s.user_scans,
        s.user_lookups,
        s.user_updates,
        (s.user_seeks + s.user_scans + s.user_lookups) AS TotalReads,
        -- Fragmentation
        ips.avg_fragmentation_in_percent,
        ips.page_count,
        ips.record_count,
        -- Last activity
        s.last_user_seek,
        s.last_user_scan,
        s.last_user_lookup,
        s.last_user_update,
        -- Classification
        CASE 
            WHEN s.user_updates = 0 AND (s.user_seeks + s.user_scans + s.user_lookups) > 0 THEN 'READ_ONLY'
            WHEN s.user_seeks + s.user_scans + s.user_lookups = 0 THEN 'UNUSED'
            WHEN s.user_updates > (s.user_seeks + s.user_scans + s.user_lookups) * 3 THEN 'HIGH_MAINTENANCE'
            WHEN (s.user_seeks + s.user_scans + s.user_lookups) > s.user_updates * 2 THEN 'HIGH_VALUE'
            ELSE 'BALANCED'
        END AS UsageCategory,
        CASE 
            WHEN ips.avg_fragmentation_in_percent > 30 AND ips.page_count > 1000 THEN 'REBUILD_REQUIRED'
            WHEN ips.avg_fragmentation_in_percent > 10 AND ips.page_count > 100 THEN 'REORGANIZE_REQUIRED'
            WHEN s.user_seeks + s.user_scans + s.user_lookups = 0 THEN 'CONSIDER_DROP'
            ELSE 'MAINTENANCE_OK'
        END AS MaintenanceAction
    FROM sys.dm_db_index_usage_stats s
    INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
    LEFT JOIN sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ips 
        ON i.object_id = ips.object_id AND i.index_id = ips.index_id
    WHERE OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
    AND s.database_id = DB_ID()
    ORDER BY ips.avg_fragmentation_in_percent DESC, ips.page_count DESC
"@
    
    try {
        $connectionString = "Server=$Server;Database=$DbName;Integrated Security=True;"
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
        Write-Log "Failed to get index performance data: $($_.Exception.Message)" 'ERROR'
        return $null
    }
}

function Get-MissingIndexRecommendations {
    param([string]$Server, [string]$DbName)
    
    $query = @"
    SELECT 
        migs.avg_user_impact,
        migs.avg_total_user_cost,
        migs.user_seeks,
        migs.user_scans,
        migs.avg_user_impact * (migs.user_seeks + migs.user_scans) AS TotalImpact,
        mid.statement AS TableName,
        mid.equality_columns,
        mid.inequality_columns,
        mid.included_columns,
        'CREATE NONCLUSTERED INDEX IX_' + 
        REPLACE(mid.statement, '.', '_') + '_' + 
        REPLACE(REPLACE(REPLACE(mid.equality_columns + mid.inequality_columns, ',', '_'), ' ', '_'), '(', '') + 
        ' ON ' + mid.statement + 
        '(' + ISNULL(mid.equality_columns + ', ', '') + ISNULL(mid.inequality_columns, '') + ')' +
        CASE WHEN mid.included_columns IS NOT NULL THEN 
            ' INCLUDE (' + mid.included_columns + ')' 
        ELSE '' END +
        ' WITH (ONLINE = ON, FILLFACTOR = 90, MAXDOP = 2);' AS CreateScript
    FROM sys.dm_db_missing_index_groups mig
    INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
    INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
    WHERE mid.database_id = DB_ID()
    AND (migs.user_seeks + migs.user_scans) > $UsageThreshold
    ORDER BY TotalImpact DESC
"@
    
    try {
        $connectionString = "Server=$Server;Database=$DbName;Integrated Security=True;"
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
        Write-Log "Failed to get missing index recommendations: $($_.Exception.Message)" 'ERROR'
        return $null
    }
}

function New-PerformanceReport {
    param([object]$PerformanceData, [object]$MissingIndexes, [string]$ReportPath)
    
    $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    $reportFile = Join-Path $ReportPath "IndexPerformance_Report_$timestamp.html"
    
    $totalIndexes = $PerformanceData.Count
    $fragmentedIndexes = ($PerformanceData | Where-Object { $_.avg_fragmentation_in_percent -gt $FragmentationThreshold }).Count
    $unusedIndexes = ($PerformanceData | Where-Object { $_.TotalReads -eq 0 -and $_.user_updates -gt 0 }).Count
    $highMaintenanceIndexes = ($PerformanceData | Where-Object { $_.MaintenanceAction -eq 'REBUILD_REQUIRED' }).Count
    $missingIndexesCount = if ($MissingIndexes) { $MissingIndexes.Count } else { 0 }
    
    $htmlReport = @"
<!DOCTYPE html>
<html>
<head>
    <title>Index Performance Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background-color: #f0f0f0; padding: 10px; border-bottom: 2px solid #ccc; }
        .summary { margin: 20px 0; padding: 15px; border: 1px solid #ddd; background-color: #f9f9f9; }
        .alert { background-color: #ffebee; padding: 10px; margin: 10px 0; border-left: 4px solid #f44336; }
        .warning { background-color: #fff3e0; padding: 10px; margin: 10px 0; border-left: 4px solid #ff9800; }
        .success { background-color: #e8f5e8; padding: 10px; margin: 10px 0; border-left: 4px solid #4caf50; }
        table { border-collapse: collapse; width: 100%; margin: 20px 0; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .fragmented { color: red; font-weight: bold; }
        .unused { color: orange; }
        .healthy { color: green; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Index Performance Report</h1>
        <p>Generated: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')</p>
        <p>Server: $ServerInstance</p>
        <p>Database: $Database</p>
    </div>
    
    <div class="summary">
        <h2>Executive Summary</h2>
        <p><strong>Total Indexes Analyzed:</strong> $totalIndexes</p>
        <p class="fragmented"><strong>Fragmented Indexes (>{$FragmentationThreshold}%):</strong> $fragmentedIndexes</p>
        <p class="unused"><strong>Unused Indexes:</strong> $unusedIndexes</p>
        <p class="alert"><strong>Indexes Requiring Rebuild:</strong> $highMaintenanceIndexes</p>
        <p class="warning"><strong>Missing Index Recommendations:</strong> $missingIndexesCount</p>
    </div>
    
    <div class="alert">
        <h3>Critical Issues</h3>
        <ul>
"@
    
    # Add critical issues
    if ($highMaintenanceIndexes -gt 0) {
        $htmlReport += "<li>$highMaintenanceIndexes indexes require immediate rebuild due to high fragmentation</li>"
    }
    if ($unusedIndexes -gt 0) {
        $htmlReport += "<li>$unusedIndexes indexes are write-only and may be candidates for dropping</li>"
    }
    
    $htmlReport += @"
        </ul>
    </div>
    
    <h2>Index Performance Analysis</h2>
    <table>
        <tr>
            <th>Table</th>
            <th>Index Name</th>
            <th>Type</th>
            <th>Fragmentation %</th>
            <th>Total Reads</th>
            <th>Updates</th>
            <th>Usage Category</th>
            <th>Maintenance Action</th>
            <th>Status</th>
        </tr>
"@
    
    foreach ($index in $PerformanceData) {
        $fragmentationClass = if ($index.avg_fragmentation_in_percent -gt $FragmentationThreshold) { 'fragmented' } else { 'healthy' }
        $usageClass = if ($index.TotalReads -eq 0) { 'unused' } else { '' }
        
        $htmlReport += @"
        <tr>
            <td>$($index.TableName)</td>
            <td>$($index.IndexName)</td>
            <td>$($index.IndexType)</td>
            <td class="$fragmentationClass">$([math]::Round($index.avg_fragmentation_in_percent, 2))</td>
            <td>$($index.TotalReads)</td>
            <td>$($index.user_updates)</td>
            <td>$($index.UsageCategory)</td>
            <td>$($index.MaintenanceAction)</td>
            <td class="$usageClass">$(
                if ($index.avg_fragmentation_in_percent -gt $FragmentationThreshold) { 'CRITICAL' }
                elseif ($index.TotalReads -eq 0) { 'UNUSED' }
                else { 'HEALTHY' }
            )</td>
        </tr>
"@
    }
    
    $htmlReport += @"
    </table>
"@
    
    if ($MissingIndexes -and $MissingIndexes.Count -gt 0) {
        $htmlReport += @"
    <h2>Missing Index Recommendations</h2>
    <p>The following indexes are recommended based on query execution patterns:</p>
    <table>
        <tr>
            <th>Table</th>
            <th>Impact Score</th>
            <th>User Seeks</th>
            <th>User Scans</th>
            <th>Recommended Index Script</th>
        </tr>
"@
        
        foreach ($missingIndex in $MissingIndexes) {
            $htmlReport += @"
        <tr>
            <td>$($missingIndex.TableName)</td>
            <td>$([math]::Round($missingIndex.TotalImpact, 2))</td>
            <td>$($missingIndex.user_seeks)</td>
            <td>$($missingIndex.user_scans)</td>
            <td><pre>$($missingIndex.CreateScript)</pre></td>
        </tr>
"@
        }
        
        $htmlReport += @"
    </table>
"@
    }
    
    $htmlReport += @"
</body>
</html>
"@
    
    $htmlReport | Out-File -FilePath $reportFile -Encoding UTF8
    return $reportFile
}

# Main execution
try {
    Write-Log "Starting index performance monitoring for $ServerInstance"
    
    # Get performance data
    $performanceData = Get-IndexPerformanceData -Server $ServerInstance -DbName $Database
    
    if (-not $performanceData) {
        Write-Log "Failed to get performance data"
        exit 1
    }
    
    # Get missing index recommendations
    $missingIndexes = Get-MissingIndexRecommendations -Server $ServerInstance -DbName $Database
    
    # Generate report
    $reportFile = New-PerformanceReport -PerformanceData $performanceData -MissingIndexes $missingIndexes -ReportPath $ReportPath
    
    Write-Log "Performance report generated: $reportFile"
    
    # Display summary
    $totalIndexes = $performanceData.Count
    $fragmentedIndexes = ($performanceData | Where-Object { $_.avg_fragmentation_in_percent -gt $FragmentationThreshold }).Count
    $unusedIndexes = ($performanceData | Where-Object { $_.TotalReads -eq 0 -and $_.user_updates -gt 0 }).Count
    
    Write-Log "=== Index Performance Summary ==="
    Write-Log "Total indexes analyzed: $totalIndexes"
    Write-Log "Fragmented indexes (>$FragmentationThreshold%): $fragmentedIndexes"
    Write-Log "Unused indexes: $unusedIndexes"
    
    if ($fragmentedIndexes -gt 0 -or $unusedIndexes -gt 0) {
        Write-Log "Index maintenance required" 'WARNING'
    }
    else {
        Write-Log "All indexes are performing well" 'SUCCESS'
    }
    
    # Send alert if significant issues found
    if ($EmailAlerts -and ($fragmentedIndexes -gt 5 -or $unusedIndexes -gt 3)) {
        Write-Log "Sending performance alert email"
        # Email implementation would go here
    }
}
catch {
    Write-Log "Index performance monitoring failed: $($_.Exception.Message)" 'ERROR'
    exit 1
}
```

## Friday - Index Troubleshooting and Advanced Scenarios

### Common Index-Related Performance Issues

#### Issue 1: Excessive Index Fragmentation

**Diagnosis and Solutions:**
```sql
-- Identify severely fragmented indexes
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent AS FragmentationPercent,
    ips.page_count AS PageCount,
    ips.record_count AS RecordCount,
    -- Calculate estimated rebuild time
    CASE 
        WHEN ips.page_count < 1000 THEN 'Quick rebuild (< 1 minute)'
        WHEN ips.page_count < 10000 THEN 'Standard rebuild (1-5 minutes)'
        WHEN ips.page_count < 100000 THEN 'Extended rebuild (5-30 minutes)'
        ELSE 'Large rebuild (30+ minutes)'
    END AS EstimatedRebuildTime,
    -- Maintenance window recommendation
    CASE 
        WHEN ips.avg_fragmentation_in_percent > 50 AND ips.page_count > 10000 THEN 'Immediate maintenance required'
        WHEN ips.avg_fragmentation_in_percent > 30 AND ips.page_count > 5000 THEN 'Schedule maintenance within 24 hours'
        WHEN ips.avg_fragmentation_in_percent > 20 AND ips.page_count > 1000 THEN 'Include in next maintenance window'
        ELSE 'Monitor'
    END AS ActionRecommendation
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.index_id > 0 -- Exclude heaps
AND OBJECTPROPERTY(ips.object_id, 'IsUserTable') = 1
AND ips.avg_fragmentation_in_percent > 20 -- Only show indexes with significant fragmentation
ORDER BY ips.avg_fragmentation_in_percent DESC, ips.page_count DESC;

-- Automated fragmentation repair
CREATE OR ALTER PROCEDURE sp_FixExcessiveFragmentation
    @TableName NVARCHAR(128) = NULL,
    @MaxExecutionTime INT = 3600,
    @FragmentationThreshold FLOAT = 40.0
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @IndexName NVARCHAR(128);
    DECLARE @Fragmentation DECIMAL(5,2);
    DECLARE @PageCount INT;
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @StartTime DATETIME;
    DECLARE @IndexType NVARCHAR(50);
    DECLARE @IsPrimaryKey BIT;
    
    DECLARE frag_cursor CURSOR FOR
    SELECT 
        i.name,
        ips.avg_fragmentation_in_percent,
        ips.page_count,
        i.type_desc,
        i.is_primary_key
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
    INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE (@TableName IS NULL OR OBJECT_NAME(ips.object_id) = @TableName)
    AND ips.index_id > 0
    AND OBJECTPROPERTY(ips.object_id, 'IsUserTable') = 1
    AND ips.avg_fragmentation_in_percent > @FragmentationThreshold
    ORDER BY ips.avg_fragmentation_in_percent DESC, ips.page_count DESC;
    
    OPEN frag_cursor;
    FETCH NEXT FROM frag_cursor INTO @IndexName, @Fragmentation, @PageCount, @IndexType, @IsPrimaryKey;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @StartTime = GETDATE();
        
        -- Choose rebuild vs reorganize based on fragmentation and size
        IF @Fragmentation > 40 AND @PageCount > 1000
        BEGIN
            -- Rebuild for severely fragmented large indexes
            SET @SQL = 'ALTER INDEX ' + QUOTENAME(@IndexName) + ' ON ' + QUOTENAME(OBJECT_NAME(DB_ID())) + '.' + QUOTENAME(@TableName) + ' REBUILD';
            
            -- Add options for primary keys
            IF @IsPrimaryKey = 1
                SET @SQL = @SQL + ' WITH (ONLINE = ON, MAXDOP = 2, FILLFACTOR = 90)';
            ELSE
                SET @SQL = @SQL + ' WITH (ONLINE = ON, MAXDOP = 2, FILLFACTOR = 90)';
            
            PRINT 'Rebuilding index ' + @IndexName + ' (Fragmentation: ' + CAST(@Fragmentation AS VARCHAR) + '%)';
            
            BEGIN TRY
                EXEC sp_executesql @SQL;
                PRINT 'Rebuild completed successfully for ' + @IndexName;
            END TRY
            BEGIN CATCH
                PRINT 'Rebuild failed for ' + @IndexName + ': ' + ERROR_MESSAGE();
            END CATCH
        END
        ELSE IF @Fragmentation > 20
        BEGIN
            -- Reorganize for moderately fragmented indexes
            SET @SQL = 'ALTER INDEX ' + QUOTENAME(@IndexName) + ' ON ' + QUOTENAME(OBJECT_NAME(DB_ID())) + '.' + QUOTENAME(@TableName) + ' REORGANIZE';
            
            PRINT 'Reorganizing index ' + @IndexName + ' (Fragmentation: ' + CAST(@Fragmentation AS VARCHAR) + '%)';
            
            BEGIN TRY
                EXEC sp_executesql @SQL;
                PRINT 'Reorganize completed successfully for ' + @IndexName;
            END TRY
            BEGIN CATCH
                PRINT 'Reorganize failed for ' + @IndexName + ': ' + ERROR_MESSAGE();
            END CATCH
        END
        
        -- Check execution time limit
        IF DATEDIFF(SECOND, @StartTime, GETDATE()) > @MaxExecutionTime
        BEGIN
            PRINT 'Maintenance stopped due to time limit';
            BREAK;
        END
        
        FETCH NEXT FROM frag_cursor INTO @IndexName, @Fragmentation, @PageCount, @IndexType, @IsPrimaryKey;
    END
    
    CLOSE frag_cursor;
    DEALLOCATE frag_cursor;
END;
```

#### Issue 2: Missing Index Recommendations

**Query Performance Analysis:**
```sql
-- Analyze queries that would benefit from indexes
SELECT 
    qsqt.query_sql_text AS QueryText,
    qsp.PlanCacheObjType,
    qsp.PlanHandle,
    qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
    qs.total_elapsed_time / qs.execution_count AS AvgElapsedTimeMs,
    qs.execution_count AS ExecutionCount,
    qs.total_logical_reads,
    qs.total_elapsed_time,
    qs.last_execution_time,
    -- Missing index information
    qsi.objectid,
    qsi.equality_columns,
    qsi.inequality_columns,
    qsi.included_columns,
    qsi.statement AS AffectedTable,
    qsi.avg_user_impact,
    qsi.avg_total_user_cost,
    qsi.user_seeks,
    qsi.user_scans
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qsqt
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qsp
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle).query_plan.nodes('/MissingIndexes/MissingIndexGroup/MissingIndex') AS qsi
WHERE qs.execution_count > 10 -- Only frequently executed queries
AND qsi.avg_user_impact > 0.1 -- Only indexes with significant impact
ORDER BY qsi.avg_user_impact * (qsi.user_seeks + qsi.user_scans) DESC;
```

#### Issue 3: Index Size and Storage Issues

**Storage Analysis:**
```sql
-- Analyze index sizes and storage impact
SELECT 
    OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc AS IndexType,
    SUM(ps.reserved_page_count) * 8 / 1024 AS IndexSizeMB,
    SUM(ps.row_count) AS RowCount,
    -- Storage efficiency metrics
    CAST(SUM(ps.reserved_page_count) * 8.0 / NULLIF(SUM(ps.row_count), 0) AS DECIMAL(18,2)) AS AvgSizePerRowKB,
    -- Compression status
    ps.data_compression_desc,
    -- Space usage analysis
    CASE 
        WHEN SUM(ps.reserved_page_count) > 100000 THEN 'Large index - monitor growth'
        WHEN SUM(ps.reserved_page_count) > 10000 THEN 'Medium index - regular monitoring'
        ELSE 'Small index - normal monitoring'
    END AS MonitoringLevel,
    -- Recommendations
    CASE 
        WHEN i.type_desc = 'HEAP' THEN 'Create clustered index to reduce fragmentation'
        WHEN i.type_desc = 'CLUSTERED' AND ps.data_compression_desc = 'NONE' THEN 'Consider data compression'
        WHEN i.type_desc = 'NONCLUSTERED' AND SUM(ps.reserved_page_count) > 50000 THEN 'Review index necessity'
        ELSE 'No immediate action needed'
    END AS StorageRecommendation
FROM sys.indexes i
INNER JOIN sys.dm_db_partition_stats ps ON i.object_id = ps.object_id AND i.index_id = ps.index_id
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
GROUP BY i.object_id, i.name, i.type_desc, ps.data_compression_desc
HAVING SUM(ps.reserved_page_count) > 0
ORDER BY SUM(ps.reserved_page_count) DESC;

-- Identify oversized indexes
SELECT 
    OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc AS IndexType,
    SUM(ps.reserved_page_count) * 8 / 1024 AS SizeMB,
    SUM(ps.row_count) AS RowCount,
    -- Calculate index overhead
    CASE 
        WHEN i.type_desc = 'CLUSTERED' THEN '0% (stores actual data)'
        ELSE CAST(
            ((SUM(ps.reserved_page_count) * 8.0 / 1024) - 
             (SELECT SUM(ps2.reserved_page_count) * 8.0 / 1024 
              FROM sys.indexes i2 
              INNER JOIN sys.dm_db_partition_stats ps2 ON i2.object_id = ps2.object_id AND i2.index_id = ps2.index_id 
              WHERE i2.type_desc = 'CLUSTERED' AND i2.object_id = i.object_id)) / 
            NULLIF((SELECT SUM(ps2.reserved_page_count) * 8.0 / 1024 
                   FROM sys.indexes i2 
                   INNER JOIN sys.dm_db_partition_stats ps2 ON i2.object_id = ps2.object_id AND i2.index_id = ps2.index_id 
                   WHERE i2.type_desc = 'CLUSTERED' AND i2.object_id = i.object_id), 0) * 100 AS DECIMAL(5,2)
    END AS IndexOverheadPercent
FROM sys.indexes i
INNER JOIN sys.dm_db_partition_stats ps ON i.object_id = ps.object_id AND i.index_id = ps.index_id
WHERE i.type_desc = 'NONCLUSTERED'
AND OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
GROUP BY i.object_id, i.name, i.type_desc
HAVING SUM(ps.reserved_page_count) > 1000 -- Only show indexes larger than 8MB
ORDER BY IndexOverheadPercent DESC;
```

### Advanced Troubleshooting Scenarios

#### Scenario 1: Plan Cache Pollution from Bad Indexes

**Plan Cache Analysis:**
```sql
-- Identify queries with high memory grant and poor index usage
SELECT 
    qsqt.query_sql_text AS QueryText,
    qs.execution_count AS ExecutionCount,
    qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
    qs.total_grant_kb / qs.execution_count AS AvgMemoryGrantKB,
    qs.total_rows / qs.execution_count AS AvgRowsReturned,
    qsp.query_plan AS ExecutionPlan,
    -- Missing index recommendations
    qsi.equality_columns,
    qsi.inequality_columns,
    qsi.included_columns,
    qsi.avg_user_impact,
    -- Plan cache efficiency
    CASE 
        WHEN qs.total_grant_kb > 100000 THEN 'High memory usage - review query'
        WHEN qs.total_logical_reads > 1000000 THEN 'High I/O usage - consider indexes'
        WHEN qs.execution_count > 1000 AND qs.total_elapsed_time > 60000 THEN 'Frequent slow execution'
        ELSE 'Normal'
    END AS QueryHealth
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qsqt
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qsp
OUTER APPLY sys.dm_exec_query_plan(qs.plan_handle).query_plan.nodes('/MissingIndexes/MissingIndexGroup/MissingIndex') AS qsi
WHERE qs.execution_count > 50 -- Only frequently executed queries
AND (qs.total_logical_reads > 100000 OR qs.total_grant_kb > 50000)
ORDER BY qs.total_logical_reads DESC;
```

#### Scenario 2: Columnstore Index Performance Issues

**Columnstore-specific troubleshooting:**
```sql
-- Analyze columnstore index performance
SELECT 
    OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc AS IndexType,
    -- Columnstore-specific metrics
    cs.segment_id,
    cs.column_id,
    c.name AS ColumnName,
    cs.row_count,
    cs.min_data_id,
    cs.max_data_id,
    cs.ondisk_size,
    cs.row_group_id,
    cs.transition_tid,
    cs.created_time,
    cs.deleted_rows,
    cs.total_rows,
    -- Segment elimination effectiveness
    CASE 
        WHEN cs.created_time < DATEADD(DAY, -30, GETDATE()) THEN 'Aging segment - consider columnstore maintenance'
        WHEN cs.deleted_rows > cs.total_rows * 0.2 THEN 'High deleted rows - consider tuple mover'
        WHEN cs.row_count < 102400 THEN 'Small row group - may not benefit from columnstore'
        ELSE 'Healthy segment'
    END AS SegmentHealth
FROM sys.indexes i
INNER JOIN sys.column_store_row_groups csr ON i.object_id = csr.object_id AND i.index_id = csr.index_id
INNER JOIN sys.column_store_segments cs ON csr.object_id = cs.object_id AND csr.row_group_id = cs.row_group_id
INNER JOIN sys.columns c ON cs.object_id = c.object_id AND cs.column_id = c.column_id
WHERE i.type_desc LIKE '%COLUMNSTORE%'
AND OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
ORDER BY cs.ondisk_size DESC;

-- Columnstore segment quality analysis
SELECT 
    OBJECT_NAME(csr.object_id) AS TableName,
    csr.index_id,
    csr.row_group_id,
    csr.state_description,
    csr.total_rows,
    csr.deleted_rows,
    CAST(csr.deleted_rows * 100.0 / NULLIF(csr.total_rows, 0) AS DECIMAL(5,2)) AS DeletedRowsPercent,
    CASE 
        WHEN csr.state_description = 'COMPRESSED' AND csr.deleted_rows > csr.total_rows * 0.3 THEN 'Needs tuple mover'
        WHEN csr.state_description = 'OPEN' THEN 'Open segments - normal'
        WHEN csr.state_description = 'COMPRESSED' AND csr.total_rows < 102400 THEN 'Small compressed segment'
        ELSE 'Healthy'
    END AS RowGroupHealth
FROM sys.column_store_row_groups csr
WHERE OBJECTPROPERTY(csr.object_id, 'IsUserTable') = 1
ORDER BY csr.total_rows DESC;

-- Tuple mover execution recommendation
SELECT 
    OBJECT_NAME(csr.object_id) AS TableName,
    COUNT(*) AS OpenRowGroups,
    SUM(csr.total_rows) AS TotalRowsInOpenGroups,
    SUM(csr.deleted_rows) AS TotalDeletedRows
FROM sys.column_store_row_groups csr
WHERE csr.state_description = 'OPEN'
GROUP BY OBJECT_NAME(csr.object_id)
HAVING COUNT(*) > 5 -- Alert if more than 5 open row groups
ORDER BY COUNT(*) DESC;
```

#### Scenario 3: Index Locking and Blocking Issues

**Lock Analysis for Index Operations:**
```sql
-- Identify blocking caused by index operations
SELECT 
    r.session_id AS BlockingSessionID,
    r.command AS BlockingCommand,
    r.percent_complete AS BlockingPercent,
    r.wait_type AS BlockingWaitType,
    r.wait_resource AS BlockingWaitResource,
    r.last_wait_type AS BlockingLastWaitType,
    r.wait_time AS BlockingWaitTime,
    t.text AS BlockingQueryText,
    -- Blocked session information
    r2.session_id AS BlockedSessionID,
    r2.command AS BlockedCommand,
    r2.percent_complete AS BlockedPercent,
    t2.text AS BlockedQueryText,
    -- Index information
    i.name AS IndexName,
    OBJECT_NAME(i.object_id) AS TableName,
    CASE 
        WHEN r.command LIKE '%INDEX%' THEN 'Index operation blocking'
        WHEN r.command LIKE '%ALTER%' THEN 'Schema modification blocking'
        ELSE 'General blocking'
    END AS BlockingType
FROM sys.dm_exec_requests r
INNER JOIN sys.dm_exec_requests r2 ON r.blocking_session_id = r2.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
CROSS APPLY sys.dm_exec_sql_text(r2.sql_handle) t2
LEFT JOIN sys.indexes i ON r.blocking_session_id = i.index_id OR r2.blocking_session_id = i.index_id
WHERE r.blocking_session_id > 0
AND r.session_id != r.blocking_session_id
ORDER BY r.wait_time DESC;
```

### PowerShell Troubleshooting Script

```powershell
# Comprehensive Index Troubleshooting Script
# Save as: Troubleshoot-IndexIssues.ps1

param(
    [Parameter(Mandatory=$true)]
    [string]$ServerInstance,
    
    [Parameter(Mandatory=$false)]
    [string]$Database = 'master',
    
    [Parameter(Mandatory=$false)]
    [string]$OutputPath = 'C:\Troubleshooting\IndexIssues',
    
    [Parameter(Mandatory=$false)]
    [int]$CriticalFragmentationThreshold = 50,
    
    [Parameter(Mandatory=$false)]
    [int]$WarningFragmentationThreshold = 30,
    
    [Parameter(Mandatory=$false)]
    [switch]$GenerateFixScripts
)

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
    
    if (-not (Test-Path $OutputPath)) {
        New-Item -ItemType Directory -Path $OutputPath -Force | Out-Null
    }
    
    $logFile = Join-Path $OutputPath "IndexTroubleshooting_$(Get-Date -Format 'yyyyMMdd').log"
    Add-Content -Path $logFile -Value $logMessage
}

function Get-IndexIssues {
    param([string]$Server, [string]$DbName)
    
    $issues = @()
    
    # Issue 1: Severe fragmentation
    $fragmentationQuery = @"
    SELECT 
        'FRAGMENTATION' AS IssueType,
        'HIGH' AS Severity,
        OBJECT_NAME(ips.object_id) AS TableName,
        i.name AS IndexName,
        ips.avg_fragmentation_in_percent AS FragmentationPercent,
        ips.page_count AS PageCount,
        'Rebuild index due to high fragmentation' AS IssueDescription,
        'ALTER INDEX [' + i.name + '] ON [' + OBJECT_NAME(ips.object_id) + '] REBUILD WITH (ONLINE = ON, MAXDOP = 2, FILLFACTOR = 90)' AS FixScript
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
    INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE ips.index_id > 0
    AND OBJECTPROPERTY(ips.object_id, 'IsUserTable') = 1
    AND ips.avg_fragmentation_in_percent > $CriticalFragmentationThreshold
    AND ips.page_count > 1000
    ORDER BY ips.avg_fragmentation_in_percent DESC
"@
    
    # Issue 2: Unused indexes
    $unusedQuery = @"
    SELECT 
        'USAGE' AS IssueType,
        'MEDIUM' AS Severity,
        OBJECT_NAME(s.object_id) AS TableName,
        i.name AS IndexName,
        (s.user_seeks + s.user_scans + s.user_lookups) AS TotalReads,
        s.user_updates AS TotalUpdates,
        'Index not used but maintained - consider dropping' AS IssueDescription,
        'DROP INDEX [' + i.name + '] ON [' + OBJECT_NAME(s.object_id) + ']' AS FixScript
    FROM sys.dm_db_index_usage_stats s
    INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
    WHERE OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
    AND s.database_id = DB_ID()
    AND (s.user_seeks + s.user_scans + s.user_lookups) = 0
    AND s.user_updates > 100 -- Index is maintained but never used
    AND i.name NOT LIKE 'PK_%' -- Don't suggest dropping primary keys
    ORDER BY s.user_updates DESC
"@
    
    # Issue 3: Missing indexes
    $missingQuery = @"
    SELECT 
        'MISSING_INDEX' AS IssueType,
        'HIGH' AS Severity,
        mid.statement AS TableName,
        'MISSING_INDEX_' + CAST(mig.index_group_handle AS VARCHAR) AS IndexName,
        migs.avg_user_impact,
        migs.user_seeks + migs.user_scans AS TotalSeeks,
        'Query plan suggests missing index' AS IssueDescription,
        'CREATE NONCLUSTERED INDEX IX_' + 
        REPLACE(REPLACE(mid.statement, '.', '_'), '[', '').Replace(']', '') + '_' + 
        REPLACE(REPLACE(REPLACE(mid.equality_columns + ISNULL(mid.inequality_columns, ''), ',', '_'), ' ', '_'), '(', '') + 
        ' ON ' + mid.statement + 
        '(' + ISNULL(mid.equality_columns + ', ', '') + ISNULL(mid.inequality_columns, '') + ')' +
        CASE WHEN mid.included_columns IS NOT NULL THEN 
            ' INCLUDE (' + mid.included_columns + ')' 
        ELSE '' END +
        ' WITH (ONLINE = ON, FILLFACTOR = 90);' AS FixScript
    FROM sys.dm_db_missing_index_groups mig
    INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
    INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
    WHERE mid.database_id = DB_ID()
    AND migs.avg_user_impact > 0.1 -- Only significant missing indexes
    ORDER BY migs.avg_user_impact * (migs.user_seeks + migs.user_scans) DESC
"@
    
    $queries = @{
        'Fragmentation' = $fragmentationQuery
        'Unused' = $unusedQuery
        'Missing' = $missingQuery
    }
    
    foreach ($queryName in $queries.Keys) {
        try {
            $connectionString = "Server=$Server;Database=$DbName;Integrated Security=True;"
            $connection = New-Object System.Data.SqlClient.SqlConnection($connectionString)
            $command = $connection.CreateCommand()
            $command.CommandText = $queries[$queryName]
            $connection.Open()
            
            $adapter = New-Object System.Data.SqlClient.SqlDataAdapter($command)
            $dataset = New-Object System.Data.DataSet
            $adapter.Fill($dataset) | Out-Null
            
            $connection.Close()
            
            foreach ($row in $dataset.Tables[0]) {
                $issues += [PSCustomObject]@{
                    IssueType = $row.IssueType
                    Severity = $row.Severity
                    TableName = $row.TableName
                    IndexName = $row.IndexName
                    Description = $row.IssueDescription
                    FixScript = $row.FixScript
                    AdditionalInfo = if ($row.FragmentationPercent) { "Fragmentation: $($row.FragmentationPercent)%" }
                                   elseif ($row.TotalReads) { "Reads: $($row.TotalReads), Updates: $($row.TotalUpdates)" }
                                   elseif ($row.avg_user_impact) { "Impact: $($row.avg_user_impact)" }
                                   else { "" }
                }
            }
        }
        catch {
            Write-Log "Failed to query $queryName issues: $($_.Exception.Message)" 'ERROR'
        }
    }
    
    return $issues
}

function New-TroubleshootingReport {
    param([object]$Issues, [string]$OutputPath)
    
    $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    $reportFile = Join-Path $OutputPath "IndexTroubleshooting_Report_$timestamp.html"
    $fixScriptFile = Join-Path $OutputPath "IndexFixScripts_$timestamp.sql"
    
    $criticalIssues = $Issues | Where-Object { $_.Severity -eq 'HIGH' }
    $mediumIssues = $Issues | Where-Object { $_.Severity -eq 'MEDIUM' }
    $lowIssues = $Issues | Where-Object { $_.Severity -eq 'LOW' }
    
    $htmlReport = @"
<!DOCTYPE html>
<html>
<head>
    <title>Index Troubleshooting Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background-color: #f0f0f0; padding: 10px; border-bottom: 2px solid #ccc; }
        .critical { background-color: #ffebee; padding: 10px; margin: 10px 0; border-left: 4px solid #f44336; }
        .medium { background-color: #fff3e0; padding: 10px; margin: 10px 0; border-left: 4px solid #ff9800; }
        .low { background-color: #e8f5e8; padding: 10px; margin: 10px 0; border-left: 4px solid #4caf50; }
        table { border-collapse: collapse; width: 100%; margin: 20px 0; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .fix-script { font-family: monospace; background-color: #f5f5f5; padding: 5px; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Index Troubleshooting Report</h1>
        <p>Generated: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')</p>
        <p>Server: $ServerInstance</p>
        <p>Database: $Database</p>
    </div>
    
    <div class="critical">
        <h2>Critical Issues ($($criticalIssues.Count) found)</h2>
        <p>These issues require immediate attention as they significantly impact performance.</p>
    </div>
    
    <div class="medium">
        <h2>Medium Priority Issues ($($mediumIssues.Count) found)</h2>
        <p>These issues should be addressed during the next maintenance window.</p>
    </div>
    
    <div class="low">
        <h2>Low Priority Issues ($($lowIssues.Count) found)</h2>
        <p>These issues can be addressed during regular maintenance activities.</p>
    </div>
    
    <h2>All Issues</h2>
    <table>
        <tr>
            <th>Severity</th>
            <th>Type</th>
            <th>Table</th>
            <th>Index</th>
            <th>Description</th>
            <th>Additional Info</th>
            <th>Recommended Fix</th>
        </tr>
"@
    
    foreach ($issue in $Issues) {
        $severityClass = $issue.Severity.ToLower()
        $htmlReport += @"
        <tr>
            <td class="$severityClass">$($issue.Severity)</td>
            <td>$($issue.IssueType)</td>
            <td>$($issue.TableName)</td>
            <td>$($issue.IndexName)</td>
            <td>$($issue.Description)</td>
            <td>$($issue.AdditionalInfo)</td>
            <td><div class="fix-script">$($issue.FixScript)</div></td>
        </tr>
"@
    }
    
    $htmlReport += @"
    </table>
</body>
</html>
"@
    
    $htmlReport | Out-File -FilePath $reportFile -Encoding UTF8
    
    # Generate fix scripts
    $fixScripts = @"
-- Index Troubleshooting Fix Scripts
-- Generated: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')
-- Server: $ServerInstance
-- Database: $Database

-- WARNING: Review and test these scripts in a development environment before running in production

"@
    
    foreach ($issue in $Issues) {
        $fixScripts += @"

-- $($issue.Severity) Priority: $($issue.IssueType)
-- Table: $($issue.TableName)
-- Index: $($issue.IndexName)
-- Description: $($issue.Description)
$($issue.FixScript)

"@
    }
    
    $fixScripts | Out-File -FilePath $fixScriptFile -Encoding UTF8
    
    return @{
        ReportFile = $reportFile
        FixScriptFile = $fixScriptFile
    }
}

# Main troubleshooting execution
try {
    Write-Log "Starting comprehensive index troubleshooting"
    
    # Get all issues
    $issues = Get-IndexIssues -Server $ServerInstance -DbName $Database
    
    if ($issues.Count -eq 0) {
        Write-Log "No index issues found - all indexes are performing optimally"
        exit 0
    }
    
    Write-Log "Found $($issues.Count) index issues"
    
    # Display summary
    $criticalCount = ($issues | Where-Object { $_.Severity -eq 'HIGH' }).Count
    $mediumCount = ($issues | Where-Object { $_.Severity -eq 'MEDIUM' }).Count
    $lowCount = ($issues | Where-Object { $_.Severity -eq 'LOW' }).Count
    
    Write-Log "=== Index Troubleshooting Summary ==="
    Write-Log "Critical issues: $criticalCount"
    Write-Log "Medium priority issues: $mediumCount"
    Write-Log "Low priority issues: $lowCount"
    
    # Generate report
    $files = New-TroubleshootingReport -Issues $issues -OutputPath $OutputPath
    
    Write-Log "Troubleshooting report saved to: $($files.ReportFile)"
    Write-Log "Fix scripts saved to: $($files.FixScriptFile)"
    
    # Display critical issues
    if ($criticalCount -gt 0) {
        Write-Log "=== Critical Issues Requiring Immediate Attention ===" 'WARNING'
        $issues | Where-Object { $_.Severity -eq 'HIGH' } | ForEach-Object {
            Write-Log "$($_.IssueType) - $($_.TableName).$($_.IndexName): $($_.Description)" 'WARNING'
        }
    }
    
    # Recommendations
    Write-Log "=== Recommendations ==="
    if ($criticalCount -gt 0) {
        Write-Log "1. Address all critical issues immediately" 'WARNING'
    }
    if ($mediumCount -gt 0) {
        Write-Log "2. Schedule medium priority issues for next maintenance window" 'INFO'
    }
    Write-Log "3. Test all fix scripts in development environment first" 'INFO'
    Write-Log "4. Monitor index performance after implementing fixes" 'INFO'
}
catch {
    Write-Log "Index troubleshooting failed: $($_.Exception.Message)" 'ERROR'
    exit 1
}
```

### Hands-On Lab: Complete Index Optimization Project

**Scenario**: Optimize indexing strategy for an e-commerce database system with performance issues.

**Tasks**:
1. Analyze current index usage and fragmentation levels
2. Identify missing indexes based on query patterns
3. Create optimal index strategy including covering and filtered indexes
4. Implement online index operations for zero-downtime maintenance
5. Set up automated index monitoring and maintenance
6. Create comprehensive performance baseline and improvement reports

**Deliverables**:
- Index analysis report with recommendations
- Optimized index scripts with covering indexes
- PowerShell automation scripts for monitoring
- Performance improvement measurements
- Index maintenance procedures and schedules

## Summary

This week covered comprehensive indexing strategies for SQL Server databases. You learned to:

- Understand different SQL Server index types and their optimal use cases
- Design effective indexing strategies based on query patterns and data characteristics
- Implement and optimize clustered, nonclustered, and columnstore indexes
- Use advanced indexing techniques like filtered indexes and covering indexes
- Perform index fragmentation analysis and maintenance
- Monitor index usage and performance impact
- Troubleshoot common index-related performance issues
- Automate index monitoring and maintenance processes

Mastering these indexing skills will enable you to optimize database performance, reduce query response times, and maintain efficient database operations in production environments.