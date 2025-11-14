# SQL Server Database Installation and Configuration Guide

## Table of Contents
1. [Database Planning and Design](#planning)
2. [Creating Databases](#creating-databases)
3. [File and Filegroup Configuration](#filegroups)
4. [Security and Authentication Setup](#security)
5. [Performance Optimization](#performance)
6. [Backup and Recovery Configuration](#backup)
7. [Monitoring and Maintenance](#monitoring)
8. [Database Migration and Upgrade](#migration)
9. [Troubleshooting Database Issues](#troubleshooting)

## Database Planning and Design {#planning}

### Step 1: Requirements Analysis

**Business Requirements Documentation:**
```
Document the following for each database:
1. Purpose and scope of the database
2. Expected data volume and growth rate
3. Number of concurrent users
4. Performance requirements (response time, throughput)
5. Availability requirements (uptime, recovery time objectives)
6. Compliance and security requirements
7. Integration requirements with other systems
```

**Data Volume Estimation:**
```sql
-- Create estimation worksheet
CREATE TABLE #DatabaseEstimate (
    TableName NVARCHAR(128),
    ExpectedRows BIGINT,
    AvgRowSizeKB DECIMAL(10,2),
    InitialSizeMB DECIMAL(18,2),
    AnnualGrowthPercent DECIMAL(5,2),
    Notes NVARCHAR(500)
);

-- Example entries for estimation
INSERT INTO #DatabaseEstimate VALUES 
('Customers', 100000, 0.5, 50, 25, 'Moderate growth expected'),
('Orders', 500000, 1.2, 600, 30, 'High transaction volume'),
('Products', 10000, 2.0, 20, 15, 'Reference data, low growth');

-- Calculate total estimated size
SELECT 
    SUM(InitialSizeMB) AS InitialSizeMB,
    SUM(InitialSizeMB * AnnualGrowthPercent / 100) AS AnnualGrowthMB,
    SUM(InitialSizeMB) + SUM(InitialSizeMB * AnnualGrowthPercent / 100) AS Year1SizeMB
FROM #DatabaseEstimate;
```

### Step 2: Naming Conventions

**Database Naming Standards:**
```
1. Use PascalCase or camelCase for database names
2. Avoid special characters except underscore (_)
3. Include business context when relevant (e.g., FinanceDB, CRMData)
4. Use meaningful suffixes (e.g., Dev, Test, Prod)
5. Keep names under 128 characters
```

**File Naming Conventions:**
```
Database Files:
- Primary Data: [DatabaseName].mdf
- Log Files: [DatabaseName]_Log.ldf
- Secondary Data: [DatabaseName]_Secondary.ndf

Filegroups:
- Primary: FG_[DatabaseName]_Primary
- Data: FG_[DatabaseName]_Data
- Indexes: FG_[DatabaseName]_Indexes
- Temp: FG_[DatabaseName]_Temp
```

### Step 3: Capacity Planning

**Storage Requirements Calculator:**
```sql
-- Create comprehensive capacity planning script
CREATE PROCEDURE sp_CapacityPlanning
    @TableCount INT,
    @AvgTablesPerDatabase INT = 10,
    @PeakConcurrentConnections INT = 100,
    @AvailabilityRequirement NVARCHAR(50) = 'High'
AS
BEGIN
    DECLARE @BaseStorageGB DECIMAL(10,2) = 5.0;
    DECLARE @PerTableStorageGB DECIMAL(10,2) = 0.1;
    DECLARE @TempDBMultiplier DECIMAL(5,2) = 0.2;
    DECLARE @LogStorageGB DECIMAL(10,2) = 2.0;
    
    DECLARE @DataStorageGB DECIMAL(10,2);
    DECLARE @TotalStorageGB DECIMAL(10,2);
    
    -- Calculate data storage requirements
    SET @DataStorageGB = @BaseStorageGB + (@TableCount * @AvgTablesPerDatabase * @PerTableStorageGB);
    
    -- Add overhead for indexes (typically 20-30% of data)
    SET @DataStorageGB = @DataStorageGB * 1.25;
    
    -- Calculate total storage
    SET @TotalStorageGB = @DataStorageGB + (@DataStorageGB * @TempDBMultiplier) + @LogStorageGB;
    
    -- Display recommendations
    SELECT 
        @DataStorageGB AS RecommendedDataStorageGB,
        @TotalStorageGB AS RecommendedTotalStorageGB,
        CASE 
            WHEN @PeakConcurrentConnections < 50 THEN 'Standard'
            WHEN @PeakConcurrentConnections < 200 THEN 'Enhanced'
            ELSE 'High Performance'
        END AS RecommendedTier,
        CASE 
            WHEN @AvailabilityRequirement = 'Critical' THEN 'Always On, Full Backup Plan'
            WHEN @AvailabilityRequirement = 'High' THEN 'Log Shipping, Daily Backups'
            ELSE 'Standard Backup Plan'
        END AS RecommendedHighAvailability;
END;
```

## Creating Databases {#creating-databases}

### Step 1: Basic Database Creation

**Standard Database Creation:**
```sql
-- Create basic database with optimal defaults
CREATE DATABASE [ProductionDB]
ON (
    NAME = 'ProductionDB_Data',
    FILENAME = 'C:\SQLData\ProductionDB.mdf',
    SIZE = 100MB,
    MAXSIZE = UNLIMITED,
    FILEGROWTH = 50MB
)
LOG ON (
    NAME = 'ProductionDB_Log',
    FILENAME = 'C:\SQLLogs\ProductionDB_Log.ldf',
    SIZE = 25MB,
    MAXSIZE = UNLIMITED,
    FILEGROWTH = 25MB
);
GO

-- Configure database options
ALTER DATABASE [ProductionDB] SET RECOVERY FULL;
ALTER DATABASE [ProductionDB] SET COMPATIBILITY_LEVEL = 160; -- SQL Server 2022
ALTER DATABASE [ProductionDB] SET PAGE_VERIFY CHECKSUM;
ALTER DATABASE [ProductionDB] SET AUTO_CREATE_STATISTICS ON;
ALTER DATABASE [ProductionDB] SET AUTO_UPDATE_STATISTICS ON;
ALTER DATABASE [ProductionDB] SET AUTO_UPDATE_STATISTICS_ASYNC ON;
```

**Automated Database Creation Script:**
```sql
-- Create comprehensive database creation procedure
CREATE PROCEDURE sp_CreateApplicationDatabase
    @DatabaseName NVARCHAR(128),
    @DataPath NVARCHAR(260) = 'C:\SQLData\',
    @LogPath NVARCHAR(260) = 'C:\SQLLogs\',
    @InitialSizeMB INT = 100,
    @GrowthSizeMB INT = 50,
    @LogGrowthSizeMB INT = 25,
    @FileCount INT = 1,
    @CreateFilegroups BIT = 1,
    @RecoveryModel NVARCHAR(20) = 'FULL'
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @DataFile NVARCHAR(260);
    DECLARE @LogFile NVARCHAR(260);
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @FilegroupData NVARCHAR(128);
    DECLARE @FilegroupIndex NVARCHAR(128);
    DECLARE @i INT = 1;
    
    -- Validate inputs
    IF @DatabaseName IS NULL OR LEN(@DatabaseName) = 0
    BEGIN
        RAISERROR('Database name is required', 16, 1);
        RETURN;
    END
    
    IF @FileCount < 1 OR @FileCount > 8
    BEGIN
        RAISERROR('File count must be between 1 and 8', 16, 1);
        RETURN;
    END
    
    -- Set up file paths
    SET @DataFile = @DataPath + @DatabaseName + '.mdf';
    SET @LogFile = @LogPath + @DatabaseName + '_Log.ldf';
    SET @FilegroupData = 'FG_' + @DatabaseName + '_Data';
    SET @FilegroupIndex = 'FG_' + @DatabaseName + '_Index';
    
    -- Create filegroups if requested
    IF @CreateFilegroups = 1
    BEGIN
        -- Create data filegroup
        SET @SQL = N'
        ALTER DATABASE ' + QUOTENAME(@DatabaseName) + N'
        ADD FILEGROUP ' + QUOTENAME(@FilegroupData) + N';';
        
        PRINT 'Creating filegroup: ' + @FilegroupData;
        EXEC sp_executesql @SQL;
        
        -- Create index filegroup
        SET @SQL = N'
        ALTER DATABASE ' + QUOTENAME(@DatabaseName) + N'
        ADD FILEGROUP ' + QUOTENAME(@FilegroupIndex) + N';';
        
        PRINT 'Creating filegroup: ' + @FilegroupIndex;
        EXEC sp_executesql @SQL;
    END
    
    -- Create database with primary file
    SET @SQL = N'
    CREATE DATABASE ' + QUOTENAME(@DatabaseName) + N'
    ON (
        NAME = ''' + @DatabaseName + N'_Data'',
        FILENAME = ''' + @DataFile + N''',
        SIZE = ' + CAST(@InitialSizeMB AS NVARCHAR(10)) + N'MB,
        MAXSIZE = UNLIMITED,
        FILEGROWTH = ' + CAST(@GrowthSizeMB AS NVARCHAR(10)) + N'MB
    )
    LOG ON (
        NAME = ''' + @DatabaseName + N'_Log'',
        FILENAME = ''' + @LogFile + N''',
        SIZE = ' + CAST(@InitialSizeMB/4 AS NVARCHAR(10)) + N'MB,
        MAXSIZE = UNLIMITED,
        FILEGROWTH = ' + CAST(@LogGrowthSizeMB AS NVARCHAR(10)) + N'MB
    );';
    
    PRINT 'Creating database: ' + @DatabaseName;
    EXEC sp_executesql @SQL;
    
    -- Add additional data files if requested
    IF @FileCount > 1
    BEGIN
        WHILE @i <= @FileCount - 1
        BEGIN
            SET @DataFile = @DataPath + @DatabaseName + N'_' + CAST(@i AS NVARCHAR(10)) + N'.ndf';
            
            SET @SQL = N'
            ALTER DATABASE ' + QUOTENAME(@DatabaseName) + N'
            ADD FILE (
                NAME = ''' + @DatabaseName + N'_Data_' + CAST(@i AS NVARCHAR(10)) + N''',
                FILENAME = ''' + @DataFile + N''',
                SIZE = ' + CAST(@InitialSizeMB AS NVARCHAR(10)) + N'MB,
                MAXSIZE = UNLIMITED,
                FILEGROWTH = ' + CAST(@GrowthSizeMB AS NVARCHAR(10)) + N'MB
            );';
            
            PRINT 'Adding secondary file: ' + @DatabaseName + N'_' + CAST(@i AS NVARCHAR(10)) + N'.ndf';
            EXEC sp_executesql @SQL;
            
            SET @i = @i + 1;
        END
    END
    
    -- Configure database options
    SET @SQL = N'
    ALTER DATABASE ' + QUOTENAME(@DatabaseName) + N'
    SET RECOVERY ' + @RecoveryModel + N',
        COMPATIBILITY_LEVEL = 160,
        PAGE_VERIFY CHECKSUM,
        AUTO_CREATE_STATISTICS ON,
        AUTO_UPDATE_STATISTICS ON,
        AUTO_UPDATE_STATISTICS_ASYNC ON,
        AUTO_CLOSE OFF,
        AUTO_SHRINK OFF;';
    
    PRINT 'Configuring database options';
    EXEC sp_executesql @SQL;
    
    -- Create default schema
    SET @SQL = N'
    USE ' + QUOTENAME(@DatabaseName) + N';
    CREATE SCHEMA dbo;';
    
    EXEC sp_executesql @SQL;
    
    PRINT 'Database ' + @DatabaseName + N' created successfully.';
END;
```

### Step 2: Database with Multiple Filegroups

**Enterprise Database Creation:**
```sql
-- Create enterprise database with filegroups
EXEC sp_CreateApplicationDatabase 
    @DatabaseName = 'EnterpriseAppDB',
    @DataPath = 'D:\SQLData\',
    @LogPath = 'E:\SQLLogs\',
    @InitialSizeMB = 500,
    @GrowthSizeMB = 100,
    @LogGrowthSizeMB = 50,
    @FileCount = 4,
    @CreateFilegroups = 1,
    @RecoveryModel = 'FULL';

-- Move system tables to primary filegroup
USE EnterpriseAppDB;
GO

ALTER DATABASE EnterpriseAppDB
MODIFY FILEGROUP FG_EnterpriseAppDB_Data DEFAULT;
GO

-- Move user objects to appropriate filegroups
-- Create tables on data filegroup
CREATE TABLE dbo.Users (
    UserID INT IDENTITY(1,1) PRIMARY KEY,
    Username NVARCHAR(50) NOT NULL,
    Email NVARCHAR(100),
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    ModifiedDate DATETIME2 DEFAULT GETDATE(),
    Active BIT DEFAULT 1
) ON [FG_EnterpriseAppDB_Data];

CREATE TABLE dbo.Orders (
    OrderID INT IDENTITY(1,1) PRIMARY KEY,
    UserID INT NOT NULL,
    OrderDate DATETIME2 DEFAULT GETDATE(),
    TotalAmount DECIMAL(10,2),
    Status NVARCHAR(20) DEFAULT 'Pending'
) ON [FG_EnterpriseAppDB_Data];

-- Create indexes on separate filegroup
CREATE NONCLUSTERED INDEX IX_Orders_UserID ON dbo.Orders(UserID) ON [FG_EnterpriseAppDB_Index];
CREATE NONCLUSTERED INDEX IX_Users_Email ON dbo.Users(Email) ON [FG_EnterpriseAppDB_Index];

-- Create partition function and scheme for large tables
CREATE PARTITION FUNCTION PF_DateRange (DATETIME2)
AS RANGE RIGHT FOR VALUES ('2024-01-01', '2024-07-01', '2025-01-01');

CREATE PARTITION SCHEME PS_DateRange
AS PARTITION PF_DateRange
TO ([FG_EnterpriseAppDB_Data], [FG_EnterpriseAppDB_Data], [FG_EnterpriseAppDB_Data], [FG_EnterpriseAppDB_Data]);

-- Create partitioned table for large datasets
CREATE TABLE dbo.AuditLog (
    AuditID BIGINT IDENTITY(1,1) PRIMARY KEY,
    EventDate DATETIME2 NOT NULL,
    UserID INT,
    Action NVARCHAR(100),
    Details NVARCHAR(MAX)
) ON PS_DateRange(EventDate);
```

## File and Filegroup Configuration {#filegroups}

### Step 1: Filegroup Strategy

**Filegroup Architecture Design:**
```
Primary Filegroup:
- Contains system tables and critical user tables
- Recommended size: 20-30% of total database size
- High-performance storage (SSD preferred)

Data Filegroup:
- Contains user data tables
- Distribute I/O across multiple files
- Size: 50-60% of total database size

Index Filegroup:
- Contains all non-clustered indexes
- Separates index I/O from data I/O
- Size: 15-20% of total database size

Archive Filegroup:
- Contains historical or archive data
- Lower-cost storage suitable
- Size: Variable based on retention policy
```

### Step 2: Optimal File Placement

**File Placement Script:**
```sql
-- Script to move files to optimal locations
ALTER DATABASE [YourDatabase]
MODIFY FILE (
    NAME = 'YourDatabase_Data',
    FILENAME = 'D:\SQLData\YourDatabase_Data.mdf'  -- Fast SSD for primary data
);

ALTER DATABASE [YourDatabase]
ADD FILE (
    NAME = 'YourDatabase_Data2',
    FILENAME = 'E:\SQLData2\YourDatabase_Data2.ndf'  -- Different physical drive
);

ALTER DATABASE [YourDatabase]
ADD FILEGROUP [FG_YourDatabase_Indexes];

ALTER DATABASE [YourDatabase]
ADD FILE (
    NAME = 'YourDatabase_Index1',
    FILENAME = 'F:\SQLIndexes\YourDatabase_Index1.ndf'  -- Separate physical drive for indexes
) TO FILEGROUP [FG_YourDatabase_Indexes];

ALTER DATABASE [YourDatabase]
MODIFY FILEGROUP [FG_YourDatabase_Data] DEFAULT;
```

### Step 3: TempDB Optimization

**TempDB Configuration:**
```sql
-- Create optimized TempDB configuration
USE master;
GO

-- Remove existing TempDB files (for initial setup)
ALTER DATABASE tempdb
REMOVE FILE tempdev2;

ALTER DATABASE tempdb
REMOVE FILE tempdev3;

-- Add additional TempDB files based on CPU cores
DECLARE @CPUCount INT;
DECLARE @FileCount INT;
DECLARE @i INT = 2;

SET @CPUCount = (SELECT cpu_count FROM sys.dm_os_sys_info);
SET @FileCount = CASE 
    WHEN @CPUCount <= 4 THEN @CPUCount
    WHEN @CPUCount <= 8 THEN @CPUCount
    ELSE 8
END;

-- Add TempDB files up to 8 (or CPU count, whichever is less)
WHILE @i <= @FileCount
BEGIN
    DECLARE @FilePath NVARCHAR(500);
    SET @FilePath = 'C:\SQLTemp\tempdev' + CAST(@i AS NVARCHAR(10)) + '.ndf';
    
    EXEC('ALTER DATABASE tempdb ADD FILE (NAME = ''tempdev' + CAST(@i AS NVARCHAR(10)) + 
          ''', FILENAME = ''' + @FilePath + 
          ''', SIZE = 256MB, FILEGROWTH = 64MB)');
    
    SET @i = @i + 1;
END;

-- Configure TempDB options
ALTER DATABASE tempdb SET RECOVERY SIMPLE;
ALTER DATABASE tempdb SET AUTO_CREATE_STATISTICS OFF;
ALTER DATABASE tempdb SET AUTO_UPDATE_STATISTICS OFF;

PRINT 'TempDB optimized with ' + CAST(@FileCount AS NVARCHAR(10)) + ' data files';
```

### Step 4: Monitoring File Growth

**File Growth Monitoring Script:**
```sql
-- Create stored procedure to monitor file growth
CREATE PROCEDURE sp_MonitorFileGrowth
    @DatabaseName NVARCHAR(128) = NULL,
    @AlertThresholdPercent INT = 80
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX);
    
    IF @DatabaseName IS NOT NULL
        SET @SQL = N'SELECT * FROM ' + QUOTENAME(@DatabaseName) + N'.sys.dm_db_file_space_usage';
    ELSE
        SET @SQL = N'SELECT * FROM sys.dm_db_file_space_usage';
    
    -- Create table variable to store results
    DECLARE @FileStats TABLE (
        DatabaseID INT,
        DatabaseName NVARCHAR(128),
        FileID INT,
        FileName NVARCHAR(128),
        FileType TINYINT,
        CurrentSizeMB DECIMAL(18,2),
        UsedSpaceMB DECIMAL(18,2),
        FreeSpaceMB DECIMAL(18,2),
        PercentUsed DECIMAL(5,2)
    );
    
    -- Query for file statistics
    INSERT INTO @FileStats
    SELECT 
        d.database_id,
        d.name,
        f.file_id,
        f.name,
        f.type,
        (f.size * 8 / 1024.0) AS CurrentSizeMB,
        ((f.size - FILEPROPERTY(f.name, 'SpaceUsed')) * 8 / 1024.0) AS FreeSpaceMB,
        (FILEPROPERTY(f.name, 'SpaceUsed') * 8 / 1024.0) AS UsedSpaceMB,
        (FILEPROPERTY(f.name, 'SpaceUsed') * 100.0 / f.size) AS PercentUsed
    FROM sys.master_files f
    INNER JOIN sys.databases d ON f.database_id = d.database_id
    WHERE d.state = 0; -- Online databases only
    
    -- Display results
    SELECT 
        DatabaseName,
        FileName,
        CASE FileType 
            WHEN 0 THEN 'Data'
            WHEN 1 THEN 'Log'
            WHEN 2 THEN 'FILESTREAM'
            ELSE 'Other'
        END AS FileType,
        CurrentSizeMB,
        UsedSpaceMB,
        FreeSpaceMB,
        PercentUsed,
        CASE 
            WHEN PercentUsed > @AlertThresholdPercent THEN 'HIGH'
            WHEN PercentUsed > (@AlertThresholdPercent - 20) THEN 'MEDIUM'
            ELSE 'NORMAL'
        END AS AlertLevel
    FROM @FileStats
    ORDER BY DatabaseName, FileType, FileID;
    
    -- Alert for high usage
    IF EXISTS (SELECT 1 FROM @FileStats WHERE PercentUsed > @AlertThresholdPercent)
    BEGIN
        PRINT 'ALERT: Files with > ' + CAST(@AlertThresholdPercent AS NVARCHAR(10)) + '% usage detected!';
        SELECT 'ALERT: Check file growth planning' AS Message;
    END
END;
```

## Security and Authentication Setup {#security}

### Step 1: User and Role Management

**Create Application Users:**
```sql
-- Create database-specific users and roles
USE [YourDatabase];
GO

-- Create application roles
CREATE ROLE ApplicationReader;
CREATE ROLE ApplicationWriter;
CREATE ROLE ApplicationAdmin;

-- Create login for application
USE master;
GO

CREATE LOGIN [AppUser] WITH PASSWORD = 'SecurePassword123!',
    DEFAULT_DATABASE = [YourDatabase],
    CHECK_POLICY = ON,
    CHECK_EXPIRATION = OFF;

-- Create database user
USE [YourDatabase];
GO

CREATE USER [AppUser] FOR LOGIN [AppUser];
ALTER ROLE ApplicationReader ADD MEMBER [AppUser];
ALTER ROLE ApplicationWriter ADD MEMBER [AppUser];

-- Grant permissions
GRANT SELECT, INSERT, UPDATE, DELETE TO ApplicationWriter;
GRANT SELECT TO ApplicationReader;

-- Create admin user
CREATE LOGIN [AppAdmin] WITH PASSWORD = 'AdminPassword456!',
    DEFAULT_DATABASE = [YourDatabase],
    CHECK_POLICY = ON,
    CHECK_EXPIRATION = OFF;

CREATE USER [AppAdmin] FOR LOGIN [AppAdmin];
ALTER ROLE db_owner ADD MEMBER [AppAdmin];
```

### Step 2: Schema-Based Security

**Implement Schema Security:**
```sql
-- Create application schemas
USE [YourDatabase];
GO

-- Create schema for different application modules
CREATE SCHEMA Sales AUTHORIZATION dbo;
CREATE SCHEMA Inventory AUTHORIZATION dbo;
CREATE SCHEMA Reporting AUTHORIZATION dbo;
CREATE SCHEMA Admin AUTHORIZATION dbo;

-- Create role-based access
CREATE ROLE SalesApp;
CREATE ROLE InventoryApp;
CREATE ROLE ReportingApp;

-- Grant schema permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::Sales TO SalesApp;
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::Inventory TO InventoryApp;
GRANT SELECT ON SCHEMA::Sales TO ReportingApp;
GRANT SELECT ON SCHEMA::Inventory TO ReportingApp;

-- Assign roles to users
ALTER ROLE SalesApp ADD MEMBER [AppUser];
ALTER ROLE InventoryApp ADD MEMBER [AppUser];
ALTER ROLE ReportingApp ADD MEMBER [ReportingUser];
```

### Step 3: Data Encryption

**Implement Transparent Data Encryption (TDE):**
```sql
-- Enable TDE for database
USE master;
GO

-- Create master key if not exists
IF NOT EXISTS (SELECT 1 FROM sys.symmetric_keys WHERE name = '##MS_DatabaseMasterKey##')
BEGIN
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'MasterKeyPassword123!';
END

-- Create certificate for TDE
IF NOT EXISTS (SELECT 1 FROM sys.certificates WHERE name = 'TDECertificate')
BEGIN
    CREATE CERTIFICATE TDECertificate
    WITH SUBJECT = 'TDE Certificate',
    EXPIRY_DATE = '2030-12-31';
END

-- Enable TDE on database
USE [YourDatabase];
GO

CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM AES_256
ENCRYPTION BY SERVER CERTIFICATE TDECertificate;

ALTER DATABASE [YourDatabase]
SET ENCRYPTION ON;

-- Verify encryption status
SELECT 
    database_id,
    name,
    encryption_state,
    encryption_state_desc,
    percent_complete,
    encryption_scan_modify_date
FROM sys.dm_database_encryption_keys;
```

### Step 4: Row-Level Security

**Implement Row-Level Security:**
```sql
-- Create security policy for multi-tenant application
USE [YourDatabase];
GO

-- Create function to filter data based on user
CREATE FUNCTION fn_securitypredicate(@TenantID NVARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_securitypredicate
    WHERE @TenantID = CAST(SESSION_CONTEXT(N'TenantID') AS NVARCHAR(50))
    OR USER_NAME() = 'dbo';

-- Apply security policy to table
CREATE SECURITY POLICY TenantSecurityPolicy
ADD FILTER PREDICATE dbo.fn_securitypredicate(TenantID) ON dbo.Orders
AFTER INSERT
WITH (STATE = ON);

-- Set tenant context
EXEC sp_set_session_context 'TenantID', 'TENANT001';
```

## Performance Optimization {#performance}

### Step 1: Index Optimization

**Create Index Strategy:**
```sql
-- Comprehensive index optimization script
USE [YourDatabase];
GO

-- Create stored procedure for index maintenance
CREATE PROCEDURE sp_OptimizeIndexes
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @TableName NVARCHAR(128);
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @IndexCount INT;
    
    -- Cursor through all user tables
    DECLARE index_cursor CURSOR FOR
    SELECT DISTINCT t.name
    FROM sys.tables t
    INNER JOIN sys.indexes i ON t.object_id = i.object_id
    WHERE t.is_ms_shipped = 0
    AND i.type > 0
    ORDER BY t.name;
    
    OPEN index_cursor;
    
    FETCH NEXT FROM index_cursor INTO @TableName;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Update statistics for table
        SET @SQL = 'UPDATE STATISTICS ' + QUOTENAME(@TableName) + ' WITH FULLSCAN;';
        EXEC sp_executesql @SQL;
        
        -- Rebuild fragmented indexes (> 30% fragmentation)
        DECLARE @ObjectID INT;
        DECLARE @IndexID INT;
        DECLARE @IndexName NVARCHAR(128);
        DECLARE @Fragmentation DECIMAL(5,2);
        
        DECLARE index_detail_cursor CURSOR FOR
        SELECT 
            i.object_id,
            i.index_id,
            i.name,
            ps.avg_fragmentation_in_percent
        FROM sys.indexes i
        INNER JOIN sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID(@TableName), NULL, NULL, 'DETAILED') ps
            ON i.object_id = ps.object_id AND i.index_id = ps.index_id
        WHERE i.type > 0
        AND ps.avg_fragmentation_in_percent > 30
        ORDER BY ps.avg_fragmentation_in_percent DESC;
        
        OPEN index_detail_cursor;
        
        FETCH NEXT FROM index_detail_cursor INTO @ObjectID, @IndexID, @IndexName, @Fragmentation;
        
        WHILE @@FETCH_STATUS = 0
        BEGIN
            -- Rebuild index
            SET @SQL = 'ALTER INDEX ' + QUOTENAME(@IndexName) + 
                      ' ON ' + QUOTENAME(@TableName) + 
                      ' REBUILD WITH (ONLINE = ON, MAXDOP = 0);';
            
            PRINT 'Rebuilding index ' + @IndexName + ' on table ' + @TableName + 
                  ' (Fragmentation: ' + CAST(@Fragmentation AS NVARCHAR(10)) + '%)';
            
            EXEC sp_executesql @SQL;
            
            FETCH NEXT FROM index_detail_cursor INTO @ObjectID, @IndexID, @IndexName, @Fragmentation;
        END
        
        CLOSE index_detail_cursor;
        DEALLOCATE index_detail_cursor;
        
        FETCH NEXT FROM index_cursor INTO @TableName;
    END
    
    CLOSE index_cursor;
    DEALLOCATE index_cursor;
    
    PRINT 'Index optimization completed.';
END;
```

### Step 2: Query Optimization

**Create Query Performance Monitoring:**
```sql
-- Create table to store query performance metrics
CREATE TABLE QueryPerformanceLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    LogDate DATETIME2 DEFAULT GETDATE(),
    QueryText NVARCHAR(MAX),
    ExecutionCount BIGINT,
    AvgExecutionTimeMs DECIMAL(18,2),
    MinExecutionTimeMs DECIMAL(18,2),
    MaxExecutionTimeMs DECIMAL(18,2),
    LastExecution DATETIME2
);

-- Create procedure to log query performance
CREATE PROCEDURE sp_LogQueryPerformance
    @QueryText NVARCHAR(MAX),
    @ExecutionTimeMs DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @ExistingLogID INT;
    
    -- Check if query exists in log
    SELECT @ExistingLogID = LogID
    FROM QueryPerformanceLog
    WHERE QueryText = @QueryText;
    
    IF @ExistingLogID IS NOT NULL
    BEGIN
        -- Update existing record
        UPDATE QueryPerformanceLog
        SET 
            ExecutionCount = ExecutionCount + 1,
            AvgExecutionTimeMs = (AvgExecutionTimeMs * ExecutionCount + @ExecutionTimeMs) / (ExecutionCount + 1),
            MinExecutionTimeMs = CASE WHEN @ExecutionTimeMs < MinExecutionTimeMs THEN @ExecutionTimeMs ELSE MinExecutionTimeMs END,
            MaxExecutionTimeMs = CASE WHEN @ExecutionTimeMs > MaxExecutionTimeMs THEN @ExecutionTimeMs ELSE MaxExecutionTimeMs END,
            LastExecution = GETDATE()
        WHERE LogID = @ExistingLogID;
    END
    ELSE
    BEGIN
        -- Insert new record
        INSERT INTO QueryPerformanceLog (QueryText, ExecutionCount, AvgExecutionTimeMs, MinExecutionTimeMs, MaxExecutionTimeMs, LastExecution)
        VALUES (@QueryText, 1, @ExecutionTimeMs, @ExecutionTimeMs, @ExecutionTimeMs, GETDATE());
    END
END;
```

### Step 3: Database Statistics Management

**Automated Statistics Management:**
```sql
-- Create procedure to maintain statistics
CREATE PROCEDURE sp_MaintainStatistics
    @DatabaseName NVARCHAR(128) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @TableName NVARCHAR(128);
    DECLARE @SchemaName NVARCHAR(128);
    DECLARE @RowCount INT;
    DECLARE @StatsCount INT;
    
    IF @DatabaseName IS NOT NULL
        SET @SQL = 'USE ' + QUOTENAME(@DatabaseName);
    ELSE
        SET @SQL = 'USE ' + DB_NAME();
    
    EXEC sp_executesql @SQL;
    
    -- Cursor through all user tables
    DECLARE table_cursor CURSOR FOR
    SELECT DISTINCT 
        s.name AS SchemaName,
        t.name AS TableName
    FROM sys.tables t
    INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
    WHERE t.is_ms_shipped = 0
    ORDER BY s.name, t.name;
    
    OPEN table_cursor;
    
    FETCH NEXT FROM table_cursor INTO @SchemaName, @TableName;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Get row count
        SET @SQL = 'SELECT @Count = COUNT(*) FROM ' + QUOTENAME(@SchemaName) + '.' + QUOTENAME(@TableName);
        EXEC sp_executesql @SQL, N'@Count INT OUTPUT', @Count = @RowCount OUTPUT;
        
        -- Update statistics if table has data
        IF @RowCount > 0
        BEGIN
            SET @SQL = 'UPDATE STATISTICS ' + QUOTENAME(@SchemaName) + '.' + QUOTENAME(@TableName) + ' WITH FULLSCAN;';
            EXEC sp_executesql @SQL;
            
            PRINT 'Updated statistics for ' + @SchemaName + '.' + @TableName + ' (' + CAST(@RowCount AS NVARCHAR(10)) + ' rows)';
        END
        
        FETCH NEXT FROM table_cursor INTO @SchemaName, @TableName;
    END
    
    CLOSE table_cursor;
    DEALLOCATE table_cursor;
    
    PRINT 'Statistics maintenance completed for ' + ISNULL(@DatabaseName, 'current database');
END;
```

## Backup and Recovery Configuration {#backup}

### Step 1: Backup Strategy Design

**Comprehensive Backup Script:**
```sql
-- Create stored procedure for automated backup
CREATE PROCEDURE sp_AutomatedBackup
    @DatabaseName NVARCHAR(128),
    @BackupPath NVARCHAR(260) = 'D:\SQLBackup\',
    @RetentionDays INT = 30,
    @Compress BIT = 1,
    @Checksum BIT = 1
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @BackupFile NVARCHAR(500);
    DECLARE @BackupType NVARCHAR(10);
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @CurrentDate NVARCHAR(20);
    
    SET @CurrentDate = CONVERT(NVARCHAR(20), GETDATE(), 120);
    SET @CurrentDate = REPLACE(@CurrentDate, ':', '');
    SET @CurrentDate = REPLACE(@CurrentDate, '-', '');
    SET @CurrentDate = REPLACE(@CurrentDate, ' ', '');
    
    -- Full backup on Sunday or if database doesn't exist
    IF DATEPART(weekday, GETDATE()) = 1 OR NOT EXISTS (SELECT 1 FROM msdb.dbo.backupset WHERE database_name = @DatabaseName)
    BEGIN
        SET @BackupType = 'FULL';
        SET @BackupFile = @BackupPath + @DatabaseName + '_FULL_' + @CurrentDate + '.bak';
        
        SET @SQL = 'BACKUP DATABASE ' + QUOTENAME(@DatabaseName) + 
                  ' TO DISK = ''' + @BackupFile + '''' +
                  ' WITH FORMAT, INIT, SKIP, REWIND, NOUNLOAD, STATS = 10';
        
        IF @Compress = 1
            SET @SQL = @SQL + ', COMPRESSION';
        
        IF @Checksum = 1
            SET @SQL = @SQL + ', CHECKSUM';
            
        EXEC sp_executesql @SQL;
        
        PRINT 'Full backup completed: ' + @BackupFile;
    END
    ELSE
    BEGIN
        -- Differential backup
        SET @BackupType = 'DIFF';
        SET @BackupFile = @BackupPath + @DatabaseName + '_DIFF_' + @CurrentDate + '.bak';
        
        SET @SQL = 'BACKUP DATABASE ' + QUOTENAME(@DatabaseName) + 
                  ' TO DISK = ''' + @BackupFile + '''' +
                  ' WITH DIFFERENTIAL, FORMAT, INIT, SKIP, REWIND, NOUNLOAD, STATS = 10';
        
        IF @Compress = 1
            SET @SQL = @SQL + ', COMPRESSION';
        
        IF @Checksum = 1
            SET @SQL = @SQL + ', CHECKSUM';
            
        EXEC sp_executesql @SQL;
        
        PRINT 'Differential backup completed: ' + @BackupFile;
    END
    
    -- Transaction log backup every 4 hours during business hours
    IF DATEPART(hour, GETDATE()) BETWEEN 8 AND 18 AND DATEPART(minute, GETDATE()) < 5
    BEGIN
        SET @BackupType = 'LOG';
        SET @BackupFile = @BackupPath + @DatabaseName + '_LOG_' + @CurrentDate + '.trn';
        
        SET @SQL = 'BACKUP LOG ' + QUOTENAME(@DatabaseName) + 
                  ' TO DISK = ''' + @BackupFile + '''' +
                  ' WITH FORMAT, INIT, SKIP, REWIND, NOUNLOAD, STATS = 10';
        
        IF @Compress = 1
            SET @SQL = @SQL + ', COMPRESSION';
        
        IF @Checksum = 1
            SET @SQL = @SQL + ', CHECKSUM';
            
        EXEC sp_executesql @SQL;
        
        PRINT 'Transaction log backup completed: ' + @BackupFile;
    END
    
    -- Cleanup old backups
    DECLARE @DeleteBeforeDate DATETIME2;
    SET @DeleteBeforeDate = DATEADD(day, -@RetentionDays, GETDATE());
    
    SET @SQL = 'EXEC master.dbo.xp_delete_file 0, N''' + @BackupPath + ''', N''bak'', ''' + CONVERT(NVARCHAR(20), @DeleteBeforeDate, 120) + ''', 1';
    EXEC sp_executesql @SQL;
    
    PRINT 'Backup maintenance completed for ' + @DatabaseName;
END;
```

### Step 2: Recovery Testing

**Recovery Testing Script:**
```sql
-- Create procedure to test database recovery
CREATE PROCEDURE sp_TestRecovery
    @DatabaseName NVARCHAR(128),
    @TestBackupPath NVARCHAR(260) = 'D:\SQLBackup\',
    @TestDatabaseSuffix NVARCHAR(20) = '_RECOVERY_TEST'
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @TestDatabaseName NVARCHAR(128);
    DECLARE @RestoreSQL NVARCHAR(MAX);
    DECLARE @BackupFile NVARCHAR(500);
    DECLARE @LogFile NVARCHAR(500);
    
    SET @TestDatabaseName = @DatabaseName + @TestDatabaseSuffix;
    
    -- Get latest full backup
    SELECT TOP 1 @BackupFile = physical_device_name
    FROM msdb.dbo.backupset bs
    INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
    WHERE bs.database_name = @DatabaseName
    AND bs.type = 'D'
    ORDER BY bs.backup_start_date DESC;
    
    IF @BackupFile IS NULL
    BEGIN
        PRINT 'ERROR: No backup found for database ' + @DatabaseName;
        RETURN;
    END
    
    -- Check if test database exists and drop it
    IF EXISTS (SELECT 1 FROM sys.databases WHERE name = @TestDatabaseName)
    BEGIN
        SET @RestoreSQL = 'RESTORE DATABASE ' + QUOTENAME(@TestDatabaseName) + ' WITH RECOVERY, REPLACE;';
        EXEC sp_executesql @RestoreSQL;
        
        SET @RestoreSQL = 'DROP DATABASE ' + QUOTENAME(@TestDatabaseName) + ';';
        EXEC sp_executesql @RestoreSQL;
    END
    
    -- Restore to test database
    SET @RestoreSQL = 'RESTORE DATABASE ' + QUOTENAME(@TestDatabaseName) + 
                     ' FROM DISK = ''' + @BackupFile + '''' +
                     ' WITH MOVE ''' + @DatabaseName + ''' TO ''' + 
                     REPLACE(@BackupFile, '.bak', @TestDatabaseSuffix + '.mdf') + ''',' +
                     ' MOVE ''' + @DatabaseName + '_Log'' TO ''' + 
                     REPLACE(@BackupFile, '.bak', @TestDatabaseSuffix + '_Log.ldf') + ''',' +
                     ' REPLACE, RECOVERY;';
    
    PRINT 'Starting recovery test for database: ' + @DatabaseName;
    PRINT 'Test database will be: ' + @TestDatabaseName;
    PRINT 'Backup file: ' + @BackupFile;
    
    EXEC sp_executesql @RestoreSQL;
    
    -- Verify recovery
    IF EXISTS (SELECT 1 FROM sys.databases WHERE name = @TestDatabaseName)
    BEGIN
        SET @RestoreSQL = 'USE ' + QUOTENAME(@TestDatabaseName) + '; SELECT DB_NAME() AS DatabaseName, COUNT(*) AS TableCount FROM sys.tables;';
        
        PRINT 'Recovery test completed successfully.';
        PRINT 'Test database created: ' + @TestDatabaseName;
        
        -- Clean up test database after verification
        DECLARE @CleanupDelay INT = 300; -- 5 minutes
        WAITFOR DELAY '00:05:00';
        
        SET @RestoreSQL = 'DROP DATABASE ' + QUOTENAME(@TestDatabaseName) + ';';
        EXEC sp_executesql @RestoreSQL;
        
        PRINT 'Test database cleaned up.';
    END
    ELSE
    BEGIN
        PRINT 'ERROR: Recovery test failed for database ' + @DatabaseName;
    END
END;
```

This comprehensive guide covers database creation, configuration, security, performance optimization, and backup/recovery. Each section includes practical scripts and procedures that can be directly implemented in production environments. Remember to test all configurations in a development environment before applying to production databases.

For additional assistance with specific database scenarios or troubleshooting, consult the SQL Server documentation or contact your database administration team.