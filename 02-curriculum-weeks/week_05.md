# Week 5: Backup and Recovery Basics

## Learning Objectives

By the end of this week, students will be able to:

1. **Understand Recovery Models**: Comprehend Simple, Full, and Bulk-Logged recovery models and their implications
2. **Design Backup Strategies**: Create comprehensive backup strategies based on business requirements and RTO/RPO objectives
3. **Execute Backup Operations**: Perform full, differential, and transaction log backups using various methods
4. **Implement Recovery Procedures**: Execute point-in-time recovery and database restoration processes
5. **Manage Backup Media**: Configure backup devices, file locations, and backup retention policies
6. **Plan for Disaster Recovery**: Develop disaster recovery plans and test recovery procedures

## Theoretical Content

### 1. SQL Server Recovery Models

#### 1.1 Understanding Recovery Models

Recovery models control how SQL Server logs transactions and determines backup and recovery capabilities. The choice of recovery model directly impacts:

- **Backup Capabilities**: What types of backups can be performed
- **Transaction Log Management**: How the transaction log is managed
- **Point-in-Time Recovery**: Ability to recover to specific points in time
- **Performance Impact**: Effect on database performance and transaction throughput

#### 1.2 Simple Recovery Model

**Characteristics**:
- **Log Management**: Transaction log space is automatically reused after checkpoint
- **Backup Types**: Only Full and Differential backups available
- **Recovery Point**: Can only recover to the end of a backup
- **Log Size**: Transaction log grows minimally and truncates automatically
- **Performance**: Best performance due to minimal logging

**Use Cases**:
- Development and testing databases
- Small databases with infrequent changes
- Databases where point-in-time recovery is not critical
- Non-critical applications

**Implementation**:
```sql
-- Set database to Simple recovery model
ALTER DATABASE SalesDatabase
SET RECOVERY SIMPLE;

-- Verify current recovery model
SELECT 
    name,
    recovery_model_desc,
    log_reuse_wait_desc
FROM sys.databases
WHERE name = 'SalesDatabase';

-- Check transaction log status
SELECT 
    DB_NAME() as DatabaseName,
    name as LogicalName,
    physical_name,
    (size*8)/1024.0 as SizeMB,
    max_size*8/1024.0 as MaxSizeMB,
    (FILEPROPERTY(name, 'SpaceUsed')*8)/1024.0 as UsedMB,
    type_desc
FROM sys.database_files
WHERE type_desc = 'LOG';
```

**Backup Strategy for Simple Recovery Model**:
```sql
-- Weekly full backup on Sunday at 2:00 AM
BACKUP DATABASE SalesDatabase 
TO DISK = 'C:\Backups\SalesDatabase_Full_Sun.bak'
WITH 
    INIT,
    NAME = 'SalesDatabase Full Backup',
    DESCRIPTION = 'Weekly full backup',
    EXPIRES = DATEADD(DAY, 30, GETDATE()),
    COMPRESSION,
    CHECKSUM;

-- Daily differential backup at 2:00 AM
BACKUP DATABASE SalesDatabase 
TO DISK = 'C:\Backups\SalesDatabase_Diff_Mon.bak'
WITH 
    DIFFERENTIAL,
    INIT,
    NAME = 'SalesDatabase Differential Backup',
    DESCRIPTION = 'Daily differential backup',
    COMPRESSION,
    CHECKSUM;
```

#### 1.3 Full Recovery Model

**Characteristics**:
- **Log Management**: Transaction log records all operations, requires regular log backups
- **Backup Types**: Full, Differential, and Transaction Log backups available
- **Recovery Point**: Can recover to any point in time using log backups
- **Log Size**: Transaction log grows until backed up or truncated
- **Performance**: Slight overhead due to full logging

**Use Cases**:
- Production databases with critical data
- Databases requiring point-in-time recovery
- High-transaction databases
- Applications with strict recovery requirements

**Implementation**:
```sql
-- Set database to Full recovery model
ALTER DATABASE SalesDatabase
SET RECOVERY FULL;

-- Full recovery model requires regular log backups
-- Set up log backup schedule (every 15 minutes)
BACKUP LOG SalesDatabase
TO DISK = 'C:\Backups\SalesDatabase_Log_14-00.bak'
WITH 
    NAME = 'SalesDatabase Transaction Log Backup',
    DESCRIPTION = '15-minute log backup',
    COMPRESSION,
    CHECKSUM;

-- Verify log backup chain
SELECT 
    database_name,
    type,
    backup_start_date,
    backup_finish_date,
    backup_size,
    first_lsn,
    last_lsn,
    checkpoint_lsn,
    database_backup_lsn
FROM msdb.dbo.backupset
WHERE database_name = 'SalesDatabase'
ORDER BY backup_start_date DESC;
```

**Backup Strategy for Full Recovery Model**:
```sql
-- Weekly full backup on Sunday at 2:00 AM
BACKUP DATABASE SalesDatabase 
TO DISK = 'C:\Backups\SalesDatabase_Full_Sun.bak'
WITH 
    INIT,
    FORMAT,
    NAME = 'SalesDatabase Full Backup',
    EXPIRES = DATEADD(DAY, 30, GETDATE()),
    COMPRESSION,
    CHECKSUM;

-- Daily differential backup at 2:00 AM (except Sunday)
BACKUP DATABASE SalesDatabase 
TO DISK = 'C:\Backups\SalesDatabase_Diff_Mon.bak'
WITH 
    DIFFERENTIAL,
    INIT,
    NAME = 'SalesDatabase Differential Backup',
    EXPIRES = DATEADD(DAY, 30, GETDATE()),
    COMPRESSION,
    CHECKSUM;

-- Hourly transaction log backups
BACKUP LOG SalesDatabase 
TO DISK = 'C:\Backups\SalesDatabase_Log_14-00.bak'
WITH 
    NAME = 'SalesDatabase Log Backup',
    EXPIRES = DATEADD(DAY, 7, GETDATE()),
    COMPRESSION,
    CHECKSUM;
```

#### 1.4 Bulk-Logged Recovery Model

**Characteristics**:
- **Log Management**: Uses minimal logging for bulk operations (SELECT INTO, BULK INSERT, etc.)
- **Backup Types**: Full, Differential, and Transaction Log backups available
- **Recovery Point**: Cannot perform point-in-time recovery for transactions that include minimally logged operations
- **Log Size**: Smaller transaction log than Full recovery model for bulk operations
- **Performance**: Better performance for bulk operations than Full recovery model

**Use Cases**:
- Databases with frequent bulk data loads
- Data warehouses with ETL processes
- Applications with both OLTP and bulk loading operations
- Situations where performance is critical during bulk operations

**Implementation**:
```sql
-- Set database to Bulk-Logged recovery model
ALTER DATABASE DataWarehouse
SET RECOVERY BULK_LOGGED;

-- After bulk operations, take immediate log backup
BACKUP LOG DataWarehouse
TO DISK = 'C:\Backups\DataWarehouse_Log_Post_Bulk.bak'
WITH 
    NAME = 'DataWarehouse Log Backup - Post Bulk',
    COMPRESSION,
    CHECKSUM;

-- Verify backup chain and check for bulk operations
SELECT 
    database_name,
    type,
    backup_start_date,
    backup_finish_date,
    backup_size/1024/1024 as BackupSizeMB,
    is_copy_only,
    is_damaged
FROM msdb.dbo.backupset
WHERE database_name = 'DataWarehouse'
ORDER BY backup_start_date DESC;
```

**Considerations for Bulk-Logged Recovery**:
```sql
-- Check for minimally logged operations
SELECT 
    name,
    log_reuse_wait_desc,
    user_access_desc
FROM sys.databases
WHERE name = 'DataWarehouse';

-- Monitor transaction log space usage
SELECT 
    DB_NAME() as DatabaseName,
    name as LogicalName,
    (size*8)/1024.0 as SizeMB,
    (FILEPROPERTY(name, 'SpaceUsed')*8)/1024.0 as UsedMB,
    (CAST(FILEPROPERTY(name, 'SpaceUsed') AS FLOAT) / size * 100) as PercentUsed
FROM sys.database_files
WHERE type_desc = 'LOG';

-- After bulk operations in Bulk-Logged mode, take log backup
-- to prevent transaction log from growing indefinitely
```

### 2. Backup Types and Strategies

#### 2.1 Full Database Backups

**Characteristics**:
- **Scope**: Complete copy of entire database
- **Time**: Longest backup time
- **Storage**: Largest backup size
- **Recovery**: Can be restored independently
- **Frequency**: Typically performed weekly or monthly

**Implementation**:
```sql
-- Basic full backup
BACKUP DATABASE ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_Full.bak';

-- Full backup with advanced options
BACKUP DATABASE ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_Full_2023_11_15.bak'
WITH 
    INIT,                          -- Initialize (overwrite) backup file
    FORMAT,                        -- Initialize media header
    NAME = 'ProductionDB Full Backup',
    DESCRIPTION = 'Weekly full backup - November 15, 2023',
    EXPIRES = DATEADD(DAY, 30, GETDATE()), -- Backup expires in 30 days
    COMPRESSION,                   -- Enable backup compression
    CHECKSUM,                      -- Calculate backup checksum
    STATS = 10,                    -- Show progress every 10%
    COPY_ONLY = 0;                 -- Regular backup (not copy-only)

-- Verify backup integrity
RESTORE VERIFYONLY 
FROM DISK = 'C:\Backups\ProductionDB_Full_2023_11_15.bak';

-- Full backup with multiple files for parallelism
BACKUP DATABASE ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_Full_01.bak',
     DISK = 'C:\Backups\ProductionDB_Full_02.bak',
     DISK = 'C:\Backups\ProductionDB_Full_03.bak'
WITH 
    INIT,
    COMPRESSION,
    CHECKSUM,
    MAXTRANSFERSIZE = 1048576,     -- 1MB transfer size
    BLOCKSIZE = 8192;              -- 8KB block size
```

**Best Practices for Full Backups**:
```sql
-- Always use compression for production backups
-- Enables CHECKSUM for data integrity verification
-- Set appropriate EXPIRES date for retention
-- Include verification step in backup process
-- Use meaningful naming conventions with timestamps

-- Example of complete backup with verification
DECLARE @BackupFile NVARCHAR(500) = 'C:\Backups\ProductionDB_Full_' + 
    REPLACE(CONVERT(VARCHAR(10), GETDATE(), 120), '-', '_') + '.bak';

BACKUP DATABASE ProductionDB 
TO DISK = @BackupFile
WITH 
    INIT,
    COMPRESSION,
    CHECKSUM,
    STATS = 10,
    NAME = 'ProductionDB Full Backup - ' + CONVERT(VARCHAR(10), GETDATE(), 120);

IF @@ERROR = 0
BEGIN
    -- Verify backup integrity
    RESTORE VERIFYONLY 
    FROM DISK = @BackupFile;
    
    IF @@ERROR = 0
        PRINT 'Backup and verification completed successfully';
    ELSE
        PRINT 'Backup completed but verification failed!';
END
ELSE
    PRINT 'Backup failed with error: ' + CAST(@@ERROR AS VARCHAR(10));
```

#### 2.2 Differential Backups

**Characteristics**:
- **Scope**: Only changes since last full backup
- **Time**: Faster than full backup
- **Storage**: Smaller than full backup
- **Recovery**: Must be combined with most recent full backup
- **Frequency**: Typically performed daily

**Implementation**:
```sql
-- Basic differential backup
BACKUP DATABASE ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_Diff.bak'
WITH 
    DIFFERENTIAL,
    INIT,
    COMPRESSION,
    CHECKSUM;

-- Differential backup with verification
DECLARE @BackupFile NVARCHAR(500) = 'C:\Backups\ProductionDB_Diff_' + 
    REPLACE(CONVERT(VARCHAR(10), GETDATE(), 120), '-', '_') + '.bak';

BACKUP DATABASE ProductionDB 
TO DISK = @BackupFile
WITH 
    DIFFERENTIAL,
    INIT,
    COMPRESSION,
    CHECKSUM,
    NAME = 'ProductionDB Differential Backup - ' + CONVERT(VARCHAR(10), GETDATE(), 120);

-- Verify differential backup
RESTORE VERIFYONLY 
FROM DISK = @BackupFile;

-- Check differential base for a database
SELECT 
    database_name,
    backup_start_date,
    backup_finish_date,
    type,
    database_backup_lsn,
    is_differential_base
FROM msdb.dbo.backupset
WHERE database_name = 'ProductionDB'
AND type = 'D'
ORDER BY backup_finish_date DESC;
```

**Differential Backup Strategy**:
```sql
-- Full backup on Sunday
-- Differential backups Monday through Saturday
-- Each differential backup contains all changes since the last full backup

-- Example restore sequence for disaster recovery:
-- 1. Restore Sunday full backup with NORECOVERY
-- 2. Restore Saturday differential backup with NORECOVERY  
-- 3. Restore all log backups from Saturday with RECOVERY
```

#### 2.3 Transaction Log Backups

**Characteristics**:
- **Scope**: Only transaction log entries since last log backup
- **Time**: Fastest backup type
- **Storage**: Varies based on transaction activity
- **Recovery**: Must be combined with full backup and subsequent differential backups
- **Frequency**: Typically every 15 minutes to 1 hour in Full recovery model

**Implementation**:
```sql
-- Basic transaction log backup
BACKUP LOG ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_Log.bak'
WITH 
    INIT,
    COMPRESSION,
    CHECKSUM;

-- Transaction log backup with advanced options
DECLARE @BackupFile NVARCHAR(500) = 'C:\Backups\ProductionDB_Log_' + 
    REPLACE(REPLACE(CONVERT(VARCHAR(20), GETDATE(), 120), '-', '_'), ':', '_') + '.bak';

BACKUP LOG ProductionDB 
TO DISK = @BackupFile
WITH 
    INIT,
    COMPRESSION,
    CHECKSUM,
    NAME = 'ProductionDB Transaction Log Backup - ' + 
           CONVERT(VARCHAR(20), GETDATE(), 120),
    EXPIRES = DATEADD(DAY, 7, GETDATE());

-- Verify log backup chain
SELECT 
    database_name,
    backup_start_date,
    type,
    first_lsn,
    last_lsn,
    backup_finish_date,
    backup_size/1024/1024 as BackupSizeMB
FROM msdb.dbo.backupset
WHERE database_name = 'ProductionDB'
AND type = 'L'  -- Log backups
ORDER BY backup_finish_date DESC;

-- Check log backup chain integrity
SELECT 
    'Log Chain Check' as CheckType,
    CASE 
        WHEN COUNT(*) > 0 THEN 'Log chain is continuous'
        ELSE 'Log chain may be broken'
    END as Status,
    COUNT(*) as BackupFiles,
    MIN(backup_finish_date) as FirstBackup,
    MAX(backup_finish_date) as LastBackup
FROM msdb.dbo.backupset
WHERE database_name = 'ProductionDB'
AND type = 'L'
AND backup_finish_date > DATEADD(DAY, -1, GETDATE());
```

**Copy-Only Backups**:
```sql
-- Copy-only backup does not break log backup chain
-- Used for special purposes like testing or off-site copies
BACKUP DATABASE ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_CopyOnly.bak'
WITH 
    COPY_ONLY,
    INIT,
    COMPRESSION,
    CHECKSUM;

-- Copy-only log backup
BACKUP LOG ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_Log_CopyOnly.bak'
WITH 
    COPY_ONLY,
    INIT,
    COMPRESSION,
    CHECKSUM;

-- Verify copy-only backup
SELECT 
    database_name,
    backup_start_date,
    type,
    is_copy_only,
    name
FROM msdb.dbo.backupset
WHERE database_name = 'ProductionDB'
AND is_copy_only = 1
ORDER BY backup_finish_date DESC;
```

### 3. Backup Media Management

#### 3.1 Backup Devices and File Management

**Physical Backup Devices**:
```sql
-- Create logical backup device
EXEC sp_addumpdevice 
    @devtype = 'disk',
    @logicalname = 'ProductionDB_Full',
    @physicalname = 'C:\Backups\ProductionDB_Full.bak';

EXEC sp_addumpdevice 
    @devtype = 'disk',
    @logicalname = 'ProductionDB_Log',
    @physicalname = 'C:\Backups\ProductionDB_Log.bak';

-- List backup devices
SELECT 
    name as LogicalName,
    physical_name as PhysicalPath,
    type_desc,
    type
FROM sys.backup_devices;

-- Use logical backup device
BACKUP DATABASE ProductionDB 
TO ProductionDB_Full;

BACKUP LOG ProductionDB 
TO ProductionDB_Log;

-- Remove backup device
EXEC sp_dropdevice 'ProductionDB_Full';
```

**File-Based Backup Management**:
```sql
-- Backup with custom file naming
BACKUP DATABASE ProductionDB 
TO DISK = 'C:\Backups\ProductionDB\Full\ProductionDB_Full_' + 
    REPLACE(CONVERT(VARCHAR(10), GETDATE(), 120), '-', '_') + '.bak';

-- Backup with subdirectories by date
DECLARE @BackupPath NVARCHAR(500) = 
    'C:\Backups\ProductionDB\' + 
    CONVERT(VARCHAR(7), GETDATE(), 126) + '\' + -- YYYY-MM
    'ProductionDB_Full_' + 
    REPLACE(CONVERT(VARCHAR(10), GETDATE(), 120), '-', '_') + '.bak';

-- Create directory if it doesn't exist
EXEC xp_create_subdir @BackupPath;

BACKUP DATABASE ProductionDB 
TO DISK = @BackupPath
WITH 
    COMPRESSION,
    CHECKSUM;
```

**Backup Retention Policies**:
```sql
-- Query backup history to identify old backups
SELECT 
    database_name,
    type,
    backup_start_date,
    backup_finish_date,
    backup_size/1024/1024 as BackupSizeMB,
    physical_device_name,
    CASE 
        WHEN backup_start_date < DATEADD(DAY, -30, GETDATE()) THEN 'Older than 30 days'
        WHEN backup_start_date < DATEADD(DAY, -7, GETDATE()) THEN 'Older than 7 days'
        ELSE 'Current'
    END as RetentionCategory
FROM msdb.dbo.backupset bs
INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE database_name = 'ProductionDB'
ORDER BY backup_start_date DESC;

-- Clean up old backup files (manual process)
-- Note: Always verify backups before deletion
EXEC xp_delete_file 0, -- Delete files older than date
    'C:\Backups\ProductionDB\Full\' , -- Path
    0, -- Subdirectory recursion
    '.bak', -- File extension
    DATEADD(DAY, -30, GETDATE()), -- Delete files older than 30 days
    1; -- Check file timestamp
```

#### 3.2 Backup Compression and Encryption

**Backup Compression**:
```sql
-- Enable backup compression (SQL Server 2008+)
BACKUP DATABASE ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_Compressed.bak'
WITH 
    COMPRESSION,
    CHECKSUM;

-- Check compression settings
SELECT 
    serverproperty('ISUPGRADED') as IsUpgraded,
    SERVERPROPERTY('Edition') as Edition,
    CASE 
        WHEN CAST(SERVERPROPERTY('ProductVersion') AS DECIMAL(10,2)) >= 10.5 
        THEN 'Backup compression available'
        ELSE 'Backup compression not available'
    END as CompressionSupport;

-- Compare compressed vs non-compressed backup sizes
-- Monitor compression ratios
SELECT 
    database_name,
    type,
    backup_start_date,
    backup_size/1024/1024 as UncompressedSizeMB,
    compressed_backup_size/1024/1024 as CompressedSizeMB,
    CAST((1.0 - (compressed_backup_size * 1.0 / backup_size)) * 100 AS DECIMAL(5,2)) as CompressionRatio
FROM msdb.dbo.backupset
WHERE database_name = 'ProductionDB'
AND compressed_backup_size IS NOT NULL
ORDER BY backup_start_date DESC;
```

**Backup Encryption**:
```sql
-- Create certificate for backup encryption
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'MasterKey_P@ssw0rd123!';

CREATE CERTIFICATE BackupEncryptionCert
WITH SUBJECT = 'Backup Encryption Certificate',
EXPIRY_DATE = '2030-12-31';

-- Create asymmetric key for backup encryption
CREATE ASYMMETIC KEY BackupKey
WITH ALGORITHM = RSA_2048;

-- Encrypted backup
BACKUP DATABASE ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_Encrypted.bak'
WITH 
    ENCRYPTION (
        ALGORITHM = AES_256,
        SERVER CERTIFICATE = BackupEncryptionCert
    ),
    COMPRESSION,
    CHECKSUM;

-- Verify encrypted backup
RESTORE VERIFYONLY 
FROM DISK = 'C:\Backups\ProductionDB_Encrypted.bak';

-- Check encryption status in backup history
SELECT 
    database_name,
    backup_start_date,
    type,
    backup_size/1024/1024 as BackupSizeMB,
    CASE 
        WHEN is_encrypted = 1 THEN 'Encrypted'
        ELSE 'Not Encrypted'
    END as EncryptionStatus,
    encryption_algorithm,
    encryptor_type
FROM msdb.dbo.backupset
WHERE database_name = 'ProductionDB'
ORDER BY backup_start_date DESC;
```

#### 3.3 Backup Verification and Integrity Checks

**Backup Verification**:
```sql
-- Verify backup integrity after creation
BACKUP DATABASE ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_Full.bak'
WITH 
    INIT,
    COMPRESSION,
    CHECKSUM;

-- Immediate verification
IF @@ERROR = 0
BEGIN
    RESTORE VERIFYONLY 
    FROM DISK = 'C:\Backups\ProductionDB_Full.bak';
    
    IF @@ERROR = 0
        PRINT 'Backup verification passed';
    ELSE
        PRINT 'Backup verification failed - backup may be corrupt!';
END

-- Comprehensive backup verification script
DECLARE @BackupFile NVARCHAR(500) = 'C:\Backups\ProductionDB_Full.bak';

-- Check if backup file exists
IF EXISTS (SELECT 1 FROM sys.master_files WHERE physical_name = @BackupFile)
BEGIN
    -- Verify backup integrity
    RESTORE VERIFYONLY 
    FROM DISK = @BackupFile;
    
    IF @@ERROR = 0
    BEGIN
        -- Get backup header information
        RESTORE HEADERONLY 
        FROM DISK = @BackupFile;
        
        -- Get backup file list
        RESTORE FILELISTONLY 
        FROM DISK = @BackupFile;
        
        PRINT 'Backup verification completed successfully';
    END
    ELSE
    BEGIN
        PRINT 'ERROR: Backup verification failed!';
        -- Add error handling/logging here
    END
END
ELSE
BEGIN
    PRINT 'ERROR: Backup file does not exist!';
END
```

**Database Integrity Checks**:
```sql
-- Check database consistency before backup
DBCC CHECKDB('ProductionDB') WITH NO_INFOMSGS;

-- Check specific tables
DBCC CHECKTABLE('ProductionDB.dbo.Orders') WITH NO_INFOMSGS;

-- Check indexes
DBCC CHECKALLOC('ProductionDB') WITH NO_INFOMSGS;

-- Check catalog consistency
DBCC CHECKCATALOG('ProductionDB') WITH NO_INFOMSGS;

-- Run integrity check as part of backup process
BACKUP DATABASE ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_Full.bak'
WITH 
    INIT,
    COMPRESSION,
    CHECKSUM;

IF @@ERROR = 0
BEGIN
    -- Run integrity check
    DBCC CHECKDB('ProductionDB') WITH NO_INFOMSGS;
    
    IF @@ERROR = 0
        PRINT 'Backup and integrity check completed successfully';
    ELSE
        PRINT 'WARNING: Database integrity issues detected!';
END
```

### 4. Recovery Procedures

#### 4.1 Complete Database Restore

**Basic Database Restore**:
```sql
-- Basic restore to same database (overwrites existing data)
RESTORE DATABASE ProductionDB_Restored
FROM DISK = 'C:\Backups\ProductionDB_Full.bak'
WITH 
    MOVE 'ProductionDB' TO 'C:\Data\ProductionDB_Restored.mdf',
    MOVE 'ProductionDB_Log' TO 'C:\Data\ProductionDB_Restored.ldf',
    REPLACE,  -- Overwrite existing database
    RECOVERY;

-- Restore to different location or database name
RESTORE DATABASE ProductionDB_Development
FROM DISK = 'C:\Backups\ProductionDB_Full.bak'
WITH 
    MOVE 'ProductionDB' TO 'C:\Data\Development\ProductionDB_Development.mdf',
    MOVE 'ProductionDB_Log' TO 'C:\Data\Development\ProductionDB_Development.ldf',
    RECOVERY;

-- Restore with verification
RESTORE DATABASE ProductionDB_Test
FROM DISK = 'C:\Backups\ProductionDB_Full.bak'
WITH 
    MOVE 'ProductionDB' TO 'C:\Data\Test\ProductionDB_Test.mdf',
    MOVE 'ProductionDB_Log' TO 'C:\Data\Test\ProductionDB_Test.ldf',
    RECOVERY,
    STATS = 10;

-- Verify restored database
DBCC CHECKDB('ProductionDB_Test') WITH NO_INFOMSGS;
```

**Restore with NORECOVERY (for additional backups)**:
```sql
-- Restore full backup with NORECOVERY (leaves database in restoring state)
RESTORE DATABASE ProductionDB_Restored
FROM DISK = 'C:\Backups\ProductionDB_Full.bak'
WITH 
    MOVE 'ProductionDB' TO 'C:\Data\ProductionDB_Restored.mdf',
    MOVE 'ProductionDB_Log' TO 'C:\Data\ProductionDB_Restored.ldf',
    NORECOVERY,  -- Database remains in restoring state
    REPLACE;

-- Restore differential backup (if available)
RESTORE DATABASE ProductionDB_Restored
FROM DISK = 'C:\Backups\ProductionDB_Diff.bak'
WITH 
    MOVE 'ProductionDB' TO 'C:\Data\ProductionDB_Restored.mdf',
    MOVE 'ProductionDB_Log' TO 'C:\Data\ProductionDB_Restored.ldf',
    NORECOVERY;

-- Restore transaction log backups
RESTORE LOG ProductionDB_Restored
FROM DISK = 'C:\Backups\ProductionDB_Log_14-00.bak'
WITH 
    NORECOVERY;

RESTORE LOG ProductionDB_Restored
FROM DISK = 'C:\Backups\ProductionDB_Log_14-15.bak'
WITH 
    NORECOVERY;

-- Restore final log backup with RECOVERY
RESTORE LOG ProductionDB_Restored
FROM DISK = 'C:\Backups\ProductionDB_Log_14-30.bak'
WITH 
    RECOVERY;
```

#### 4.2 Point-in-Time Recovery

**Point-in-Time Recovery Process**:
```sql
-- 1. Restore full backup with NORECOVERY
RESTORE DATABASE ProductionDB
FROM DISK = 'C:\Backups\ProductionDB_Full_Sun.bak'
WITH 
    MOVE 'ProductionDB' TO 'C:\Data\ProductionDB.mdf',
    MOVE 'ProductionDB_Log' TO 'C:\Data\ProductionDB.ldf',
    NORECOVERY,
    REPLACE;

-- 2. Restore differential backup if available
RESTORE DATABASE ProductionDB
FROM DISK = 'C:\Backups\ProductionDB_Diff_Sat.bak'
WITH 
    MOVE 'ProductionDB' TO 'C:\Data\ProductionDB.mdf',
    MOVE 'ProductionDB_Log' TO 'C:\Data\ProductionDB.ldf',
    NORECOVERY;

-- 3. Restore transaction log backups up to point-in-time
-- Assume we want to recover to 2:30 PM on Saturday
RESTORE LOG ProductionDB
FROM DISK = 'C:\Backups\ProductionDB_Log_Sat_14-00.bak'
WITH 
    NORECOVERY;

RESTORE LOG ProductionDB
FROM DISK = 'C:\Backups\ProductionDB_Log_Sat_14-15.bak'
WITH 
    NORECOVERY;

RESTORE LOG ProductionDB
FROM DISK = 'C:\Backups\ProductionDB_Log_Sat_14-30.bak'
WITH 
    STOPAT = '2023-11-18 14:30:00',  -- Point-in-time recovery
    RECOVERY;

-- Alternative: Using CONTINUE_AFTER_ERROR if log backup is damaged
RESTORE LOG ProductionDB
FROM DISK = 'C:\Backups\ProductionDB_Log_Damaged.bak'
WITH 
    CONTINUE_AFTER_ERROR,
    RECOVERY;
```

**Monitoring Point-in-Time Recovery**:
```sql
-- Check database recovery state
SELECT 
    name,
    state_desc,
    user_access_desc,
    is_read_only,
    recovery_model_desc
FROM sys.databases
WHERE name = 'ProductionDB';

-- Check recovery progress
SELECT 
    percent_complete,
    estimated_completion_time,
    command,
    wait_type,
    wait_time,
    last_wait_type,
    status,
    database_id
FROM sys.dm_exec_requests
WHERE database_id = DB_ID('ProductionDB');

-- Check backup history to plan recovery
SELECT 
    type,
    CASE type
        WHEN 'D' THEN 'Full'
        WHEN 'I' THEN 'Differential'
        WHEN 'L' THEN 'Log'
        WHEN 'F' THEN 'File'
        WHEN 'G' THEN 'Filegroup'
        WHEN 'P' THEN 'Partial'
        WHEN 'Q' THEN 'Differential Partial'
        ELSE 'Unknown'
    END as BackupType,
    backup_start_date,
    backup_finish_date,
    database_backup_lsn,
    database_creation_date,
    backup_size/1024/1024 as BackupSizeMB
FROM msdb.dbo.backupset
WHERE database_name = 'ProductionDB'
AND backup_start_date >= DATEADD(DAY, -7, GETDATE())
ORDER BY backup_start_date;
```

#### 4.3 File and Filegroup Recovery

**Individual File Recovery**:
```sql
-- Restore individual data file
RESTORE DATABASE ProductionDB
FILE = 'ProductionDB_Data2'
FROM DISK = 'C:\Backups\ProductionDB_DataFile2.bak'
WITH 
    MOVE 'ProductionDB_Data2' TO 'C:\Data\ProductionDB_Data2.ndf',
    NORECOVERY;

-- Restore filegroup
RESTORE DATABASE ProductionDB
FILEGROUP = 'ProductionDB_FileGroup2'
FROM DISK = 'C:\Backups\ProductionDB_FileGroup2.bak'
WITH 
    MOVE 'ProductionDB_FileGroup2' TO 'C:\Data\ProductionDB_FileGroup2.ndf',
    NORECOVERY;

-- Restore multiple files
RESTORE DATABASE ProductionDB
FILE = 'ProductionDB_Data2',
       'ProductionDB_Data3'
FROM DISK = 'C:\Backups\ProductionDB_MultiFiles.bak'
WITH 
    MOVE 'ProductionDB_Data2' TO 'C:\Data\ProductionDB_Data2.ndf',
    MOVE 'ProductionDB_Data3' TO 'C:\Data\ProductionDB_Data3.ndf',
    NORECOVERY;
```

**Partial Backup and Restore**:
```sql
-- Create partial backup (primary filegroup + selected filegroups)
BACKUP DATABASE ProductionDB
FILEGROUP = 'PRIMARY',
            'ProductionDB_FileGroup1'
TO DISK = 'C:\Backups\ProductionDB_Partial.bak'
WITH 
    PARTIAL,
    INIT,
    COMPRESSION,
    CHECKSUM;

-- Restore partial backup
RESTORE DATABASE ProductionDB_Partial
FILEGROUP = 'PRIMARY',
            'ProductionDB_FileGroup1'
FROM DISK = 'C:\Backups\ProductionDB_Partial.bak'
WITH 
    MOVE 'ProductionDB' TO 'C:\Data\Partial\ProductionDB_Partial.mdf',
    MOVE 'ProductionDB_FileGroup1' TO 'C:\Data\Partial\ProductionDB_FileGroup1.ndf',
    RECOVERY;
```

### 5. Disaster Recovery Planning

#### 5.1 Recovery Objectives and Testing

**RTO/RPO Planning**:
```sql
-- Document RTO (Recovery Time Objective) and RPO (Recovery Point Objective)
CREATE TABLE dbo.DisasterRecoveryObjectives (
    DatabaseName NVARCHAR(128) NOT NULL,
    RTO_Minutes INT NOT NULL,          -- Recovery Time Objective
    RPO_Minutes INT NOT NULL,          -- Recovery Point Objective
    BusinessCriticality NVARCHAR(50) NOT NULL, -- CRITICAL, HIGH, MEDIUM, LOW
    LastUpdated DATETIME2 NOT NULL DEFAULT GETDATE(),
    UpdatedBy NVARCHAR(128) NOT NULL
);

-- Example RTO/RPO definitions
INSERT INTO dbo.DisasterRecoveryObjectives (DatabaseName, RTO_Minutes, RPO_Minutes, BusinessCriticality, UpdatedBy)
VALUES 
    ('ProductionDB', 60, 15, 'CRITICAL', 'DBA_Team'),
    ('ReportingDB', 240, 60, 'HIGH', 'DBA_Team'),
    ('DevelopmentDB', 480, 240, 'MEDIUM', 'DBA_Team'),
    ('TestDB', 480, 240, 'LOW', 'DBA_Team');

-- Query RTO/RPO for planning
SELECT 
    DatabaseName,
    RTO_Minutes,
    RPO_Minutes,
    BusinessCriticality,
    CASE 
        WHEN RTO_Minutes <= 60 AND RPO_Minutes <= 15 THEN 'HIGH_AVAILABILITY_REQUIRED'
        WHEN RTO_Minutes <= 240 AND RPO_Minutes <= 60 THEN 'STANDARD_BACKUP_REQUIRED'
        ELSE 'BASIC_BACKUP_SUFFICIENT'
    END as RequiredSolution
FROM dbo.DisasterRecoveryObjectives
ORDER BY RTO_Minutes, RPO_Minutes;
```

**Disaster Recovery Testing**:
```sql
-- Create disaster recovery test log
CREATE TABLE dbo.DRTestLog (
    TestID INT IDENTITY(1,1) PRIMARY KEY,
    TestDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    DatabaseName NVARCHAR(128) NOT NULL,
    TestType NVARCHAR(50) NOT NULL, -- FULL_RESTORE, POINT_IN_TIME, FILE_RECOVERY
    TestDurationMinutes INT,
    Success BIT NOT NULL,
    IssuesFound NVARCHAR(MAX),
    ActionsTaken NVARCHAR(MAX),
    TestedBy NVARCHAR(128) NOT NULL
);

-- Document recovery test procedures
CREATE PROCEDURE sp_ExecuteDRTest
    @DatabaseName NVARCHAR(128),
    @TestType NVARCHAR(50),
    @BackupLocation NVARCHAR(500),
    @RestoredDatabaseName NVARCHAR(128)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @StartTime DATETIME2 = GETDATE();
    DECLARE @Success BIT = 1;
    DECLARE @Issues NVARCHAR(MAX) = '';
    DECLARE @TestID INT;
    
    BEGIN TRY
        -- Create restored database name for testing
        SET @RestoredDatabaseName = @DatabaseName + '_DR_TEST_' + 
            REPLACE(CONVERT(VARCHAR(10), GETDATE(), 120), '-', '_');
        
        -- Execute appropriate test based on type
        IF @TestType = 'FULL_RESTORE'
        BEGIN
            -- Restore full backup
            RESTORE DATABASE @RestoredDatabaseName
            FROM DISK = @BackupLocation
            WITH 
                MOVE @DatabaseName TO 'C:\Data\DR_Test\' + @RestoredDatabaseName + '.mdf',
                MOVE @DatabaseName + '_Log' TO 'C:\Data\DR_Test\' + @RestoredDatabaseName + '.ldf',
                RECOVERY;
            
            -- Verify restored database
            DBCC CHECKDB(@RestoredDatabaseName) WITH NO_INFOMSGS;
        END
        -- Additional test types would be implemented here
        
        -- Log successful test
        INSERT INTO dbo.DRTestLog (
            DatabaseName, TestType, Success, IssuesFound, TestedBy
        )
        VALUES (
            @DatabaseName, @TestType, 1, '', SYSTEM_USER
        );
        
        SET @TestID = SCOPE_IDENTITY();
        
    END TRY
    BEGIN CATCH
        SET @Success = 0;
        SET @Issues = ERROR_MESSAGE();
        
        -- Log failed test
        INSERT INTO dbo.DRTestLog (
            DatabaseName, TestType, Success, IssuesFound, TestedBy
        )
        VALUES (
            @DatabaseName, @TestType, 0, @Issues, SYSTEM_USER
        );
        
        SET @TestID = SCOPE_IDENTITY();
        
        -- Clean up failed test database
        IF EXISTS (SELECT 1 FROM sys.databases WHERE name = @RestoredDatabaseName)
        BEGIN
            ALTER DATABASE @RestoredDatabaseName SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
            DROP DATABASE @RestoredDatabaseName;
        END
    END CATCH
    
    -- Update test with duration
    UPDATE dbo.DRTestLog
    SET TestDurationMinutes = DATEDIFF(MINUTE, @StartTime, GETDATE())
    WHERE TestID = @TestID;
    
    -- Return test results
    SELECT 
        TestID,
        @Success as Success,
        @Issues as Issues,
        DATEDIFF(MINUTE, @StartTime, GETDATE()) as DurationMinutes;
END;
```

#### 5.2 High Availability and Disaster Recovery Solutions

**Database Mirroring (Legacy)**:
```sql
-- Note: Database Mirroring is deprecated in SQL Server 2012+
-- Replaced by Always On Availability Groups

-- Create endpoint for database mirroring
CREATE ENDPOINT [Mirroring_Endpoint] 
    AS TCP (LISTENER_PORT = 5022)
    FOR DATA_MIRRORING (ROLE = ALL, AUTHENTICATION = WINDOWS NEGOTIATE);

-- Configure database mirroring
ALTER DATABASE ProductionDB
SET PARTNER = 'TCP://Server2.domain.com:5022';
```

**Log Shipping**:
```sql
-- Enable log shipping on primary server
-- Note: This requires SQL Server Agent to be configured

-- Configure log shipping on secondary server
RESTORE DATABASE ProductionDB
FROM DISK = 'C:\Backups\ProductionDB_Log.bak'
WITH 
    MOVE 'ProductionDB' TO 'C:\Data\ProductionDB.mdf',
    MOVE 'ProductionDB_Log' TO 'C:\Data\ProductionDB.ldf',
    STANDBY = 'C:\Data\ProductionDB_Undo.dat',  -- For read-only access
    NORECOVERY;

-- Configure transaction log backup job on primary
EXEC sp_add_job 
    @job_name = N'Log Shipping Backup',
    @enabled = 1,
    @description = N'Backup transaction logs for log shipping';

-- Configure log copy and restore jobs on secondary
EXEC sp_add_job 
    @job_name = N'Log Shipping Copy',
    @enabled = 1,
    @description = N'Copy transaction log backups from primary';

EXEC sp_add_job 
    @job_name = N'Log Shipping Restore',
    @enabled = 1,
    @description = N'Restore transaction log backups on secondary';
```

#### 5.3 Cloud-Based Backup and Recovery

**Azure SQL Database Backup**:
```sql
-- Automated backups (enabled by default)
-- Point-in-time restore: 7-35 days retention
-- Long-term retention: Up to 10 years

-- Restore to point in time
-- Use Azure Portal or Azure PowerShell
RESTORE DATABASE (Azure SQL Database)
FROM BACKUP_ID = 'backup_id'
WITH RESTORE_DDPDATA, RESTORE_WITH_NEW_LOGICAL_NAME;

-- Long-term retention backup
-- Configure using Azure Portal or T-SQL
BACKUP DATABASE [database_name]
TO URL = 'https://storageaccount.blob.core.windows.net/backupcontainer/database_backup.bak'
WITH 
    CREDENTIAL = 'AzureStorageCredential',
    STATS = 10;
```

**Hybrid Cloud Backup Strategy**:
```sql
-- On-premises backup with cloud archiving
-- Step 1: Local backup for fast recovery
BACKUP DATABASE ProductionDB 
TO DISK = 'C:\Backups\ProductionDB_Local.bak'
WITH 
    COMPRESSION,
    CHECKSUM;

-- Step 2: Archive to cloud storage
-- Use Azure Blob Storage, AWS S3, or similar
-- PowerShell script to upload backup to cloud
# Upload-LocalBackupToCloud -LocalPath "C:\Backups\ProductionDB_Local.bak" -CloudContainer "backupcontainer"

-- Step 3: Cloud backup verification
-- Verify backup in cloud storage
# Verify-CloudBackup -CloudURL "https://storageaccount.blob.core.windows.net/backupcontainer/ProductionDB_Local.bak"

-- Create backup verification procedure
CREATE PROCEDURE sp_VerifyCloudBackup
    @CloudBackupURL NVARCHAR(500),
    @ExpectedSizeMB INT
AS
BEGIN
    -- This would typically use PowerShell or .NET to verify cloud backup
    -- For demonstration purposes, this is a placeholder
    
    -- Log verification attempt
    INSERT INTO dbo.BackupVerificationLog (
        BackupLocation, VerificationTime, Success, SizeMB, BackupType
    )
    VALUES (
        @CloudBackupURL, GETDATE(), 1, @ExpectedSizeMB, 'CLOUD_ARCHIVE'
    );
    
    PRINT 'Cloud backup verification completed';
END;
```

## Practical Exercises

### Exercise 1: Comprehensive Backup Strategy Implementation

**Objective**: Design and implement a complete backup strategy for a production database with specific RTO/RPO requirements

**Scenario**: Implement backup strategy for TechCorp's production database with the following requirements:

**Business Requirements**:
- **Database**: CustomerDB (50GB, 10GB daily growth)
- **RTO**: 4 hours maximum
- **RPO**: 15 minutes maximum
- **Critical Hours**: Monday-Friday, 8:00 AM - 6:00 PM
- **Compliance**: SOX compliance requiring 7-year retention
- **Storage**: Local storage + Azure cloud archive
- **Performance**: Minimal impact during business hours

**Tasks**:

1. **Recovery Model Selection**
```sql
-- Analyze requirements and select appropriate recovery model
-- Based on RPO requirement (15 minutes), Full recovery model is required

-- Set database to Full recovery model
USE master;
ALTER DATABASE CustomerDB
SET RECOVERY FULL;

-- Verify recovery model and transaction log configuration
SELECT 
    name,
    recovery_model_desc,
    log_reuse_wait_desc,
    user_access_desc
FROM sys.databases
WHERE name = 'CustomerDB';
```

2. **Design Backup Schedule**
```sql
-- Create comprehensive backup schedule based on requirements
/*
Backup Strategy:
- Full Backup: Sunday at 2:00 AM (off-peak hours)
- Differential Backup: Daily at 2:00 AM (except Sunday)
- Transaction Log Backup: Every 15 minutes during business hours
- Transaction Log Backup: Every 60 minutes after business hours
- Monthly Full Backup: First Sunday of each month for compliance
- Retention: 30 days online, 7 years in archive
*/

-- Full backup schedule (Sunday)
DECLARE @FullBackupPath NVARCHAR(500) = 
    'C:\Backups\CustomerDB\Full\CustomerDB_Full_' + 
    CONVERT(VARCHAR(8), GETDATE(), 112) + '.bak';

BACKUP DATABASE CustomerDB 
TO DISK = @FullBackupPath
WITH 
    INIT,
    COMPRESSION,
    CHECKSUM,
    NAME = 'CustomerDB Full Backup - ' + CONVERT(VARCHAR(10), GETDATE(), 120),
    EXPIRES = DATEADD(DAY, 30, GETDATE()),
    DESCRIPTION = 'Weekly full backup for CustomerDB';

-- Differential backup schedule (Monday-Saturday)
DECLARE @DiffBackupPath NVARCHAR(500) = 
    'C:\Backups\CustomerDB\Differential\CustomerDB_Diff_' + 
    CONVERT(VARCHAR(8), GETDATE(), 112) + '.bak';

BACKUP DATABASE CustomerDB 
TO DISK = @DiffBackupPath
WITH 
    DIFFERENTIAL,
    INIT,
    COMPRESSION,
    CHECKSUM,
    NAME = 'CustomerDB Differential Backup - ' + CONVERT(VARCHAR(10), GETDATE(), 120),
    EXPIRES = DATEADD(DAY, 30, GETDATE());

-- Transaction log backup schedule (15 minutes during business hours)
DECLARE @LogBackupPath NVARCHAR(500) = 
    'C:\Backups\CustomerDB\Log\CustomerDB_Log_' + 
    REPLACE(REPLACE(CONVERT(VARCHAR(20), GETDATE(), 120), '-', '_'), ':', '_') + '.bak';

BACKUP LOG CustomerDB 
TO DISK = @LogBackupPath
WITH 
    INIT,
    COMPRESSION,
    CHECKSUM,
    NAME = 'CustomerDB Log Backup - ' + CONVERT(VARCHAR(20), GETDATE(), 120),
    EXPIRES = DATEADD(DAY, 7, GETDATE());
```

3. **Implement Backup Verification**
```sql
-- Create comprehensive backup verification procedure
CREATE PROCEDURE sp_BackupAndVerify
    @BackupType NVARCHAR(20), -- 'FULL', 'DIFFERENTIAL', 'LOG'
    @DatabaseName NVARCHAR(128)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @BackupFile NVARCHAR(500);
    DECLARE @StartTime DATETIME2 = GETDATE();
    DECLARE @BackupDuration INT;
    DECLARE @ErrorOccurred BIT = 0;
    DECLARE @ErrorMessage NVARCHAR(MAX) = '';
    
    -- Generate backup file name based on type and timestamp
    SET @BackupFile = 'C:\Backups\' + @DatabaseName + '\' + @BackupType + '\' + 
        @DatabaseName + '_' + @BackupType + '_' + 
        REPLACE(CONVERT(VARCHAR(20), GETDATE(), 120), ':', '_') + '.bak';
    
    BEGIN TRY
        -- Execute backup based on type
        IF @BackupType = 'FULL'
        BEGIN
            BACKUP DATABASE @DatabaseName 
            TO DISK = @BackupFile
            WITH 
                INIT,
                COMPRESSION,
                CHECKSUM,
                STATS = 10,
                NAME = @DatabaseName + ' Full Backup',
                DESCRIPTION = 'Automated full backup at ' + CONVERT(VARCHAR(20), GETDATE(), 120);
        END
        ELSE IF @BackupType = 'DIFFERENTIAL'
        BEGIN
            BACKUP DATABASE @DatabaseName 
            TO DISK = @BackupFile
            WITH 
                DIFFERENTIAL,
                INIT,
                COMPRESSION,
                CHECKSUM,
                STATS = 10,
                NAME = @DatabaseName + ' Differential Backup',
                DESCRIPTION = 'Automated differential backup at ' + CONVERT(VARCHAR(20), GETDATE(), 120);
        END
        ELSE IF @BackupType = 'LOG'
        BEGIN
            BACKUP LOG @DatabaseName 
            TO DISK = @BackupFile
            WITH 
                INIT,
                COMPRESSION,
                CHECKSUM,
                STATS = 10,
                NAME = @DatabaseName + ' Log Backup',
                DESCRIPTION = 'Automated log backup at ' + CONVERT(VARCHAR(20), GETDATE(), 120);
        END
        
        -- Verify backup integrity
        RESTORE VERIFYONLY 
        FROM DISK = @BackupFile;
        
        IF @@ERROR != 0
        BEGIN
            SET @ErrorOccurred = 1;
            SET @ErrorMessage = 'Backup verification failed';
        END
        
        -- Get backup information
        INSERT INTO dbo.BackupLog (
            DatabaseName, BackupType, BackupFile, BackupStartTime, BackupEndTime,
            Success, ErrorMessage, FileSizeMB, BackupDurationSeconds
        )
        SELECT 
            @DatabaseName,
            @BackupType,
            @BackupFile,
            @StartTime,
            GETDATE(),
            CASE WHEN @ErrorOccurred = 0 THEN 1 ELSE 0 END,
            @ErrorMessage,
            backup_size/1024/1024,
            DATEDIFF(SECOND, @StartTime, GETDATE())
        FROM msdb.dbo.backupset
        WHERE database_name = @DatabaseName
        AND backup_start_date >= @StartTime
        ORDER BY backup_start_date DESC
        OFFSET 0 ROWS FETCH NEXT 1 ROW ONLY;
        
        -- Return backup information
        SELECT 
            BackupType = @BackupType,
            DatabaseName = @DatabaseName,
            BackupFile = @BackupFile,
            Success = CASE WHEN @ErrorOccurred = 0 THEN 1 ELSE 0 END,
            DurationSeconds = DATEDIFF(SECOND, @StartTime, GETDATE()),
            ErrorMessage = @ErrorMessage;
        
    END TRY
    BEGIN CATCH
        -- Log backup failure
        INSERT INTO dbo.BackupLog (
            DatabaseName, BackupType, BackupFile, BackupStartTime, BackupEndTime,
            Success, ErrorMessage
        )
        VALUES (
            @DatabaseName,
            @BackupType,
            @BackupFile,
            @StartTime,
            GETDATE(),
            0,
            ERROR_MESSAGE()
        );
        
        -- Return error information
        SELECT 
            BackupType = @BackupType,
            DatabaseName = @DatabaseName,
            BackupFile = @BackupFile,
            Success = 0,
            DurationSeconds = DATEDIFF(SECOND, @StartTime, GETDATE()),
            ErrorMessage = ERROR_MESSAGE();
            
        THROW;  -- Re-throw the error
    END CATCH
END;
```

4. **Implement Retention and Cleanup**
```sql
-- Create backup cleanup procedure
CREATE PROCEDURE sp_CleanupOldBackups
    @DatabaseName NVARCHAR(128),
    @RetentionDays INT = 30
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CutoffDate DATETIME2 = DATEADD(DAY, -@RetentionDays, GETDATE());
    DECLARE @ArchiveCutoffDate DATETIME2 = DATEADD(YEAR, -1, GETDATE()); -- 1 year for archive
    
    -- Log cleanup operations
    DECLARE @CleanupLogID INT;
    
    BEGIN TRY
        -- Clean up local backup files older than retention period
        -- Note: This is a simplified example - production implementation would be more comprehensive
        
        -- Get list of old backup files from msdb
        SELECT 
            bs.backup_set_id,
            bs.database_name,
            bs.type,
            bs.backup_start_date,
            bmf.physical_device_name,
            bs.backup_size/1024/1024 as BackupSizeMB
        FROM msdb.dbo.backupset bs
        INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
        WHERE bs.database_name = @DatabaseName
        AND bs.backup_start_date < @CutoffDate
        AND bmf.device_type = 2  -- Disk devices
        ORDER BY bs.backup_start_date DESC;
        
        -- Archive backups to cloud storage before deletion
        -- This would be implemented using PowerShell or Azure PowerShell
        -- Example:
        -- PowerShell: Move-BackupToCloud -LocalPath $PhysicalDeviceName -CloudContainer "archive"
        
        -- Log cleanup action
        INSERT INTO dbo.BackupCleanupLog (
            DatabaseName, ActionType, BackupFile, BackupDate, CleanupDate, 
            CleanupMethod, Success, Notes
        )
        SELECT 
            bs.database_name,
            'LOCAL_CLEANUP',
            bmf.physical_device_name,
            bs.backup_start_date,
            GETDATE(),
            'FILE_DELETION',
            1,
            'Backup moved to cloud archive before deletion'
        FROM msdb.dbo.backupset bs
        INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
        WHERE bs.database_name = @DatabaseName
        AND bs.backup_start_date < @CutoffDate
        AND bmf.device_type = 2
        AND NOT EXISTS (
            SELECT 1 FROM dbo.BackupCleanupLog bcl
            WHERE bcl.BackupFile = bmf.physical_device_name
        );
        
        -- Return cleanup statistics
        SELECT 
            DatabaseName = @DatabaseName,
            TotalBackupsCleaned = @@ROWCOUNT,
            CutoffDate = @CutoffDate,
            CleanupDate = GETDATE();
            
    END TRY
    BEGIN CATCH
        -- Log cleanup failure
        INSERT INTO dbo.BackupCleanupLog (
            DatabaseName, ActionType, BackupFile, CleanupDate, 
            CleanupMethod, Success, Notes
        )
        VALUES (
            @DatabaseName,
            'CLEANUP_ERROR',
            'Multiple files',
            GETDATE(),
            'AUTOMATED_CLEANUP',
            0,
            ERROR_MESSAGE()
        );
        
        THROW;
    END CATCH
END;
```

**Deliverable**: Complete backup strategy implementation with verification, monitoring, and automated cleanup procedures

### Exercise 2: Disaster Recovery Testing

**Objective**: Design and execute comprehensive disaster recovery testing procedures

**Scenario**: Conduct full disaster recovery testing for TechCorp's CustomerDB following established RTO/RPO objectives

**Requirements**:
- **Test Environment**: Separate from production
- **Test Database**: CustomerDB_Restore_Test
- **Backup Files**: Use existing backup files
- **Recovery Time**: Measure and document
- **Data Integrity**: Verify all data after restore
- **Documentation**: Complete test report

**Tasks**:

1. **Preparation for DR Test**
```sql
-- Create test environment database name
DECLARE @TestDBName NVARCHAR(128) = 'CustomerDB_DR_TEST_' + 
    REPLACE(CONVERT(VARCHAR(8), GETDATE(), 112), '-', '_');

-- Identify available backup files
SELECT 
    database_name,
    type,
    backup_start_date,
    backup_finish_date,
    backup_size/1024/1024 as BackupSizeMB,
    is_damaged,
    is_copy_only,
    physical_device_name
FROM msdb.dbo.backupset bs
INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE database_name = 'CustomerDB'
AND backup_finish_date >= DATEADD(DAY, -7, GETDATE())
ORDER BY backup_start_date DESC;

-- Create DR test log table
CREATE TABLE dbo.DRTestResults (
    TestID INT IDENTITY(1,1) PRIMARY KEY,
    TestDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    DatabaseName NVARCHAR(128) NOT NULL,
    TestType NVARCHAR(50) NOT NULL,
    RTO_Target INT NOT NULL,  -- Target RTO in minutes
    RPO_Target INT NOT NULL,  -- Target RPO in minutes
    ActualRecoveryTime INT,   -- Actual recovery time in minutes
    DataLossMinutes INT,      -- Estimated data loss in minutes
    Success BIT NOT NULL,
    IssuesFound NVARCHAR(MAX),
    ActionsTaken NVARCHAR(MAX),
    TestedBy NVARCHAR(128) NOT NULL,
    TestEnvironment NVARCHAR(100) NOT NULL
);
```

2. **Full Database Recovery Test**
```sql
-- Full disaster recovery test
CREATE PROCEDURE sp_ExecuteFullDRTest
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @TestStartTime DATETIME2 = GETDATE();
    DECLARE @TestDBName NVARCHAR(128);
    DECLARE @RTO_Target INT = 240;  -- 4 hours
    DECLARE @Success BIT = 1;
    DECLARE @ErrorMessage NVARCHAR(MAX) = '';
    
    -- Generate unique test database name
    SET @TestDBName = 'CustomerDB_DR_TEST_' + 
        REPLACE(CONVERT(VARCHAR(8), GETDATE(), 112), '-', '_');
    
    BEGIN TRY
        -- Step 1: Get latest full backup
        DECLARE @FullBackupFile NVARCHAR(500);
        
        SELECT TOP 1 @FullBackupFile = bmf.physical_device_name
        FROM msdb.dbo.backupset bs
        INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
        WHERE bs.database_name = 'CustomerDB'
        AND bs.type = 'D'  -- Full backup
        ORDER BY bs.backup_finish_date DESC;
        
        -- Step 2: Restore full backup
        PRINT 'Starting full database restore...';
        RESTORE DATABASE @TestDBName
        FROM DISK = @FullBackupFile
        WITH 
            MOVE 'CustomerDB' TO 'C:\Data\DR_Test\' + @TestDBName + '.mdf',
            MOVE 'CustomerDB_Log' TO 'C:\Data\DR_Test\' + @TestDBName + '.ldf',
            NORECOVERY,
            REPLACE,
            STATS = 10;
        
        -- Step 3: Restore differential backup if available
        DECLARE @DiffBackupFile NVARCHAR(500);
        
        SELECT TOP 1 @DiffBackupFile = bmf.physical_device_name
        FROM msdb.dbo.backupset bs
        INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
        WHERE bs.database_name = 'CustomerDB'
        AND bs.type = 'I'  -- Differential backup
        AND bs.database_backup_lsn = (
            SELECT database_backup_lsn 
            FROM msdb.dbo.backupset 
            WHERE backup_set_id = (SELECT backup_set_id FROM msdb.dbo.backupset WHERE database_name = 'CustomerDB' AND physical_device_name = @FullBackupFile)
        )
        ORDER BY bs.backup_finish_date DESC;
        
        IF @DiffBackupFile IS NOT NULL
        BEGIN
            PRINT 'Restoring differential backup...';
            RESTORE DATABASE @TestDBName
            FROM DISK = @DiffBackupFile
            WITH 
                MOVE 'CustomerDB' TO 'C:\Data\DR_Test\' + @TestDBName + '.mdf',
                MOVE 'CustomerDB_Log' TO 'C:\Data\DR_Test\' + @TestDBName + '.ldf',
                NORECOVERY,
                STATS = 10;
        END
        
        -- Step 4: Restore transaction log backups
        DECLARE @LogBackupFile NVARCHAR(500);
        DECLARE log_cursor CURSOR FOR
        SELECT TOP 10 bmf.physical_device_name
        FROM msdb.dbo.backupset bs
        INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
        WHERE bs.database_name = 'CustomerDB'
        AND bs.type = 'L'  -- Log backup
        AND bs.backup_finish_date > (
            SELECT ISNULL(MAX(backup_finish_date), '1900-01-01')
            FROM msdb.dbo.backupset
            WHERE database_name = 'CustomerDB'
            AND type IN ('D', 'I')  -- Full or differential
        )
        ORDER BY bs.backup_finish_date ASC;
        
        OPEN log_cursor;
        FETCH NEXT FROM log_cursor INTO @LogBackupFile;
        
        WHILE @@FETCH_STATUS = 0
        BEGIN
            PRINT 'Restoring log backup: ' + @LogBackupFile;
            RESTORE DATABASE @TestDBName
            FROM DISK = @LogBackupFile
            WITH 
                NORECOVERY,
                STATS = 10;
            
            FETCH NEXT FROM log_cursor INTO @LogBackupFile;
        END
        
        CLOSE log_cursor;
        DEALLOCATE log_cursor;
        
        -- Step 5: Complete recovery
        RESTORE DATABASE @TestDBName WITH RECOVERY;
        
        -- Step 6: Verify database integrity
        PRINT 'Verifying database integrity...';
        DBCC CHECKDB(@TestDBName) WITH NO_INFOMSGS;
        
        -- Step 7: Verify data consistency
        PRINT 'Verifying data consistency...';
        DECLARE @RecordCount INT;
        SELECT @RecordCount = COUNT(*) FROM @TestDBName.dbo.Customers;
        
        -- Step 8: Calculate recovery time
        DECLARE @RecoveryTime INT = DATEDIFF(MINUTE, @TestStartTime, GETDATE());
        
        -- Log successful test
        INSERT INTO dbo.DRTestResults (
            DatabaseName, TestType, RTO_Target, RPO_Target,
            ActualRecoveryTime, Success, ActionsTaken, TestedBy, TestEnvironment
        )
        VALUES (
            'CustomerDB', 'FULL_RESTORE', @RTO_Target, 15,
            @RecoveryTime, 1, 'Full recovery test completed successfully', 
            SYSTEM_USER, 'DR_TEST_ENVIRONMENT'
        );
        
        PRINT 'DR Test completed successfully in ' + CAST(@RecoveryTime AS VARCHAR(10)) + ' minutes';
        
    END TRY
    BEGIN CATCH
        SET @Success = 0;
        SET @ErrorMessage = ERROR_MESSAGE();
        
        -- Log failed test
        INSERT INTO dbo.DRTestResults (
            DatabaseName, TestType, RTO_Target, RPO_Target,
            ActualRecoveryTime, Success, IssuesFound, ActionsTaken, TestedBy, TestEnvironment
        )
        VALUES (
            'CustomerDB', 'FULL_RESTORE', @RTO_Target, 15,
            DATEDIFF(MINUTE, @TestStartTime, GETDATE()), 0, @ErrorMessage,
            'DR test failed - see error details', SYSTEM_USER, 'DR_TEST_ENVIRONMENT'
        );
        
        -- Clean up failed test database
        IF EXISTS (SELECT 1 FROM sys.databases WHERE name = @TestDBName)
        BEGIN
            ALTER DATABASE @TestDBName SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
            DROP DATABASE @TestDBName;
        END
        
        PRINT 'DR Test failed: ' + @ErrorMessage;
        THROW;
    END CATCH
    
    -- Return test results
    SELECT 
        TestDate,
        DatabaseName,
        TestType,
        RTO_Target,
        RPO_Target,
        ActualRecoveryTime,
        Success,
        IssuesFound,
        ActionsTaken
    FROM dbo.DRTestResults
    WHERE TestDate = @TestStartTime;
END;
```

3. **Point-in-Time Recovery Test**
```sql
-- Point-in-time recovery test
CREATE PROCEDURE sp_ExecutePITRTest
    @PointInTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @TestStartTime DATETIME2 = GETDATE();
    DECLARE @TestDBName NVARCHAR(128);
    DECLARE @RTO_Target INT = 240;  -- 4 hours
    DECLARE @RPO_Target INT = 15;   -- 15 minutes
    DECLARE @Success BIT = 1;
    DECLARE @ErrorMessage NVARCHAR(MAX) = '';
    
    -- Set default point-in-time if not specified
    IF @PointInTime IS NULL
        SET @PointInTime = DATEADD(HOUR, -1, GETDATE());
    
    -- Generate unique test database name
    SET @TestDBName = 'CustomerDB_PITR_TEST_' + 
        REPLACE(CONVERT(VARCHAR(8), GETDATE(), 112), '-', '_');
    
    BEGIN TRY
        PRINT 'Executing point-in-time recovery test...';
        PRINT 'Target recovery time: ' + CONVERT(VARCHAR(20), @PointInTime, 120);
        
        -- Get latest full backup before target time
        DECLARE @FullBackupFile NVARCHAR(500);
        
        SELECT TOP 1 @FullBackupFile = bmf.physical_device_name
        FROM msdb.dbo.backupset bs
        INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
        WHERE bs.database_name = 'CustomerDB'
        AND bs.type = 'D'  -- Full backup
        AND bs.backup_finish_date <= @PointInTime
        ORDER BY bs.backup_finish_date DESC;
        
        IF @FullBackupFile IS NULL
        BEGIN
            RAISERROR('No suitable full backup found for point-in-time recovery', 16, 1);
        END
        
        -- Restore full backup
        PRINT 'Restoring full backup...';
        RESTORE DATABASE @TestDBName
        FROM DISK = @FullBackupFile
        WITH 
            MOVE 'CustomerDB' TO 'C:\Data\DR_Test\' + @TestDBName + '.mdf',
            MOVE 'CustomerDB_Log' TO 'C:\Data\DR_Test\' + @TestDBName + '.ldf',
            NORECOVERY,
            REPLACE,
            STATS = 10;
        
        -- Restore differential backup if available and before target time
        DECLARE @DiffBackupFile NVARCHAR(500);
        
        SELECT TOP 1 @DiffBackupFile = bmf.physical_device_name
        FROM msdb.dbo.backupset bs
        INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
        WHERE bs.database_name = 'CustomerDB'
        AND bs.type = 'I'  -- Differential backup
        AND bs.backup_finish_date <= @PointInTime
        ORDER BY bs.backup_finish_date DESC;
        
        IF @DiffBackupFile IS NOT NULL
        BEGIN
            PRINT 'Restoring differential backup...';
            RESTORE DATABASE @TestDBName
            FROM DISK = @DiffBackupFile
            WITH 
                MOVE 'CustomerDB' TO 'C:\Data\DR_Test\' + @TestDBName + '.mdf',
                MOVE 'CustomerDB_Log' TO 'C:\Data\DR_Test\' + @TestDBName + '.ldf',
                NORECOVERY,
                STATS = 10;
        END
        
        -- Restore log backups up to point-in-time
        DECLARE @LogBackupFile NVARCHAR(500);
        DECLARE @IsLastLog BIT = 0;
        
        DECLARE log_cursor CURSOR FOR
        SELECT bmf.physical_device_name,
               bs.backup_finish_date,
               ROW_NUMBER() OVER (ORDER BY bs.backup_finish_date DESC) as RowNum
        FROM msdb.dbo.backupset bs
        INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
        WHERE bs.database_name = 'CustomerDB'
        AND bs.type = 'L'  -- Log backup
        AND bs.backup_finish_date > @PointInTime
        AND bs.backup_finish_date <= @PointInTime
        ORDER BY bs.backup_finish_date DESC;
        
        OPEN log_cursor;
        FETCH NEXT FROM log_cursor INTO @LogBackupFile, @PointInTime, @IsLastLog;
        
        WHILE @@FETCH_STATUS = 0
        BEGIN
            PRINT 'Restoring log backup: ' + @LogBackupFile;
            
            RESTORE DATABASE @TestDBName
            FROM DISK = @LogBackupFile
            WITH 
                NORECOVERY,
                STATS = 10;
            
            FETCH NEXT FROM log_cursor INTO @LogBackupFile, @PointInTime, @IsLastLog;
        END
        
        CLOSE log_cursor;
        DEALLOCATE log_cursor;
        
        -- Complete recovery with point-in-time
        RESTORE DATABASE @TestDBName 
        WITH RECOVERY, 
             STOPAT = @PointInTime;
        
        -- Calculate recovery time and data loss
        DECLARE @RecoveryTime INT = DATEDIFF(MINUTE, @TestStartTime, GETDATE());
        DECLARE @DataLossMinutes INT = DATEDIFF(MINUTE, @PointInTime, GETDATE());
        
        -- Log successful test
        INSERT INTO dbo.DRTestResults (
            DatabaseName, TestType, RTO_Target, RPO_Target,
            ActualRecoveryTime, DataLossMinutes, Success, ActionsTaken, TestedBy, TestEnvironment
        )
        VALUES (
            'CustomerDB', 'POINT_IN_TIME_RECOVERY', @RTO_Target, @RPO_Target,
            @RecoveryTime, @DataLossMinutes, 1,
            'Point-in-time recovery test completed successfully at ' + CONVERT(VARCHAR(20), @PointInTime, 120),
            SYSTEM_USER, 'DR_TEST_ENVIRONMENT'
        );
        
        PRINT 'PITR Test completed successfully in ' + CAST(@RecoveryTime AS VARCHAR(10)) + ' minutes';
        PRINT 'Data loss: ' + CAST(@DataLossMinutes AS VARCHAR(10)) + ' minutes';
        
    END TRY
    BEGIN CATCH
        SET @Success = 0;
        SET @ErrorMessage = ERROR_MESSAGE();
        
        -- Log failed test
        INSERT INTO dbo.DRTestResults (
            DatabaseName, TestType, RTO_Target, RPO_Target,
            ActualRecoveryTime, Success, IssuesFound, ActionsTaken, TestedBy, TestEnvironment
        )
        VALUES (
            'CustomerDB', 'POINT_IN_TIME_RECOVERY', @RTO_Target, @RPO_Target,
            DATEDIFF(MINUTE, @TestStartTime, GETDATE()), 0, @ErrorMessage,
            'PITR test failed - see error details', SYSTEM_USER, 'DR_TEST_ENVIRONMENT'
        );
        
        -- Clean up failed test database
        IF EXISTS (SELECT 1 FROM sys.databases WHERE name = @TestDBName)
        BEGIN
            ALTER DATABASE @TestDBName SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
            DROP DATABASE @TestDBName;
        END
        
        PRINT 'PITR Test failed: ' + @ErrorMessage;
        THROW;
    END CATCH
END;
```

**Deliverable**: Complete disaster recovery test procedures with documentation of RTO/RPO achievement and data integrity verification

### Exercise 3: Advanced Recovery Scenarios

**Objective**: Implement and test advanced recovery scenarios including file recovery, filegroup recovery, and partial database recovery

**Scenario**: Handle various database failures and implement appropriate recovery strategies

**Requirements**:
- **File Failure Recovery**: Simulate data file corruption and implement file-level recovery
- **Filegroup Recovery**: Test recovery of specific filegroups
- **Partial Recovery**: Demonstrate partial database restore scenarios
- **Emergency Procedures**: Implement emergency recovery procedures

**Tasks**:

1. **File-Level Recovery**
```sql
-- Simulate file-level recovery scenario
-- Step 1: Identify files in database
SELECT 
    database_id,
    file_id,
    type_desc,
    name as LogicalName,
    physical_name as PhysicalPath,
    (size*8)/1024.0 as SizeMB,
    max_size*8/1024.0 as MaxSizeMB,
    growth*8/1024.0 as GrowthMB,
    is_percent_growth
FROM sys.database_files
WHERE database_id = DB_ID('CustomerDB')
ORDER BY file_id;

-- Create backup of specific files for testing
BACKUP DATABASE CustomerDB
FILE = 'CustomerDB_Data2'
TO DISK = 'C:\Backups\CustomerDB_FileBackup.bak'
WITH 
    INIT,
    COMPRESSION,
    CHECKSUM,
    NAME = 'CustomerDB File Backup',
    DESCRIPTION = 'Backup of CustomerDB_Data2 file';

-- Simulate file corruption and recovery
-- Note: In production, this would be preceded by proper diagnostic procedures

-- Step 1: Restore specific file with NORECOVERY
RESTORE DATABASE CustomerDB_FileRestore
FILE = 'CustomerDB_Data2'
FROM DISK = 'C:\Backups\CustomerDB_FileBackup.bak'
WITH 
    MOVE 'CustomerDB_Data2' TO 'C:\Data\FileRestore\CustomerDB_Data2.ndf',
    NORECOVERY,
    REPLACE;

-- Step 2: Restore transaction log backup to bring file online
RESTORE LOG CustomerDB_FileRestore
FROM DISK = 'C:\Backups\CustomerDB_Log_Latest.bak'
WITH 
    NORECOVERY;

-- Complete recovery
RESTORE DATABASE CustomerDB_FileRestore WITH RECOVERY;

-- Verify recovered file
SELECT 
    name,
    state_desc,
    is_read_only,
    size*8/1024.0 as SizeMB
FROM sys.database_files
WHERE database_id = DB_ID('CustomerDB_FileRestore')
AND type_desc = 'ROWS';
```

2. **Filegroup Recovery**
```sql
-- Create filegroup backup scenario
-- Assume database has multiple filegroups
-- Primary filegroup + Secondary filegroup

-- Backup specific filegroup
BACKUP DATABASE CustomerDB
FILEGROUP = 'CustomerDB_SecondaryFG'
TO DISK = 'C:\Backups\CustomerDB_FileGroupBackup.bak'
WITH 
    INIT,
    COMPRESSION,
    CHECKSUM,
    NAME = 'CustomerDB Secondary FileGroup Backup';

-- Restore filegroup backup
RESTORE DATABASE CustomerDB_FileGroupRestore
FILEGROUP = 'CustomerDB_SecondaryFG'
FROM DISK = 'C:\Backups\CustomerDB_FileGroupBackup.bak'
WITH 
    MOVE 'CustomerDB_SecondaryFG' TO 'C:\Data\FileGroupRestore\CustomerDB_SecondaryFG.ndf',
    NORECOVERY,
    REPLACE;

-- Restore transaction logs to make filegroup consistent
RESTORE LOG CustomerDB_FileGroupRestore
FROM DISK = 'C:\Backups\CustomerDB_Log_1.bak'
WITH 
    NORECOVERY;

RESTORE LOG CustomerDB_FileGroupRestore
FROM DISK = 'C:\Backups\CustomerDB_Log_2.bak'
WITH 
    NORECOVERY;

RESTORE LOG CustomerDB_FileGroupRestore
FROM DISK = 'C:\Backups\CustomerDB_Log_3.bak'
WITH 
    RECOVERY;

-- Verify filegroup recovery
SELECT 
    fg.name as FileGroupName,
    df.name as FileName,
    df.state_desc,
    (df.size*8)/1024.0 as FileSizeMB
FROM sys.filegroups fg
INNER JOIN sys.database_files df ON fg.data_space_id = df.data_space_id
WHERE fg.name = 'CustomerDB_SecondaryFG'
AND fg.database_id = DB_ID('CustomerDB_FileGroupRestore');
```

3. **Partial Database Recovery**
```sql
-- Create partial backup scenario
-- Backup primary filegroup and critical tables

-- Create partial backup (primary + critical filegroups)
BACKUP DATABASE CustomerDB
FILEGROUP = 'PRIMARY',
            'CustomerDB_CriticalFG'
TO DISK = 'C:\Backups\CustomerDB_Partial.bak'
WITH 
    PARTIAL,
    INIT,
    COMPRESSION,
    CHECKSUM,
    NAME = 'CustomerDB Partial Backup',
    DESCRIPTION = 'Primary + Critical Filegroups';

-- Restore partial backup
RESTORE DATABASE CustomerDB_Partial
FILEGROUP = 'PRIMARY',
            'CustomerDB_CriticalFG'
FROM DISK = 'C:\Backups\CustomerDB_Partial.bak'
WITH 
    MOVE 'CustomerDB' TO 'C:\Data\PartialRestore\CustomerDB_Partial.mdf',
    MOVE 'CustomerDB_CriticalFG' TO 'C:\Data\PartialRestore\CustomerDB_CriticalFG.ndf',
    RECOVERY;

-- Verify partial restore
SELECT 
    name,
    state_desc,
    is_read_only,
    filegroup_id
FROM sys.database_files
WHERE database_id = DB_ID('CustomerDB_Partial')
ORDER BY filegroup_id, file_id;

-- Check which filegroups are online
SELECT 
    fg.name as FileGroupName,
    fg.type_desc,
    fg.is_read_only,
    COUNT(df.file_id) as FileCount
FROM sys.filegroups fg
LEFT JOIN sys.database_files df ON fg.data_space_id = df.data_space_id
WHERE fg.database_id = DB_ID('CustomerDB_Partial')
GROUP BY fg.name, fg.type_desc, fg.is_read_only
ORDER BY fg.name;
```

**Deliverable**: Advanced recovery procedures for various failure scenarios with testing and documentation

## Real-World Scenarios

### Scenario 1: E-commerce Platform Disaster Recovery

**Background**: ShopFast Online operates a critical e-commerce platform with 24/7 availability requirements. During Black Friday, their database server experiences catastrophic hardware failure requiring immediate disaster recovery.

**Current Situation**:
- **Database**: CustomerOrderDB (2TB, 50GB daily transaction volume)
- **Peak Hours**: 8:00 AM - 11:00 PM daily, with Black Friday peak
- **RTO**: 2 hours maximum
- **RPO**: 30 minutes maximum
- **Backup Strategy**: Full backup Sunday 2:00 AM, Log backups every 15 minutes
- **Disaster**: Primary server hardware failure, database files corrupted

**Recovery Challenge**:
The primary database server suffered complete hardware failure. The DBA team must restore the database to a standby server within 2 hours while minimizing data loss and ensuring no loss of transactions during Black Friday peak hours.

**DBA Solution Implementation**:

1. **Immediate Response and Assessment**
```sql
-- Step 1: Assess available backup files
-- This would be executed on the standby recovery server
SELECT 
    bs.database_name,
    bs.type,
    CASE bs.type
        WHEN 'D' THEN 'Full Backup'
        WHEN 'I' THEN 'Differential Backup'
        WHEN 'L' THEN 'Log Backup'
        ELSE 'Unknown'
    END as BackupType,
    bs.backup_start_date,
    bs.backup_finish_date,
    bs.backup_size/1024/1024 as BackupSizeMB,
    bmf.physical_device_name,
    bs.first_lsn,
    bs.last_lsn,
    bs.database_backup_lsn
FROM msdb.dbo.backupset bs
INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.database_name = 'CustomerOrderDB'
AND bs.backup_finish_date >= DATEADD(DAY, -1, GETDATE()) -- Last 24 hours
ORDER BY bs.backup_finish_date DESC;

-- Step 2: Identify the recovery point
-- Determine latest full backup and log chain
DECLARE @LatestFullBackup NVARCHAR(500);
DECLARE @LatestDiffBackup NVARCHAR(500);
DECLARE @PointInTime DATETIME2;
DECLARE @TotalRecoveryTime INT;
DECLARE @StartTime DATETIME2 = GETDATE();

-- Get latest full backup
SELECT TOP 1 @LatestFullBackup = bmf.physical_device_name
FROM msdb.dbo.backupset bs
INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.database_name = 'CustomerOrderDB'
AND bs.type = 'D'
ORDER BY bs.backup_finish_date DESC;

-- Get latest differential backup
SELECT TOP 1 @LatestDiffBackup = bmf.physical_device_name
FROM msdb.dbo.backupset bs
INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.database_name = 'CustomerOrderDB'
AND bs.type = 'I'
ORDER BY bs.backup_finish_date DESC;

-- Calculate point-in-time (current time - 30 minutes RPO)
SET @PointInTime = DATEADD(MINUTE, -30, GETDATE());
```

2. **Disaster Recovery Execution**
```sql
-- Step 3: Execute full disaster recovery
-- This would be performed as a single transaction to minimize downtime

RESTORE DATABASE CustomerOrderDB_DR
FROM DISK = @LatestFullBackup
WITH 
    MOVE 'CustomerOrderDB' TO 'D:\SQLData\CustomerOrderDB_DR.mdf',
    MOVE 'CustomerOrderDB_Log' TO 'D:\SQLData\CustomerOrderDB_DR.ldf',
    MOVE 'CustomerOrderDB_Data2' TO 'E:\SQLData\CustomerOrderDB_Data2.ndf',
    MOVE 'CustomerOrderDB_Data3' TO 'E:\SQLData\CustomerOrderDB_Data3.ndf',
    NORECOVERY,
    REPLACE,
    STATS = 5; -- Show progress every 5%

-- Restore differential backup if available and beneficial
IF @LatestDiffBackup IS NOT NULL
BEGIN
    RESTORE DATABASE CustomerOrderDB_DR
    FROM DISK = @LatestDiffBackup
    WITH 
        MOVE 'CustomerOrderDB' TO 'D:\SQLData\CustomerOrderDB_DR.mdf',
        MOVE 'CustomerOrderDB_Log' TO 'D:\SQLData\CustomerOrderDB_DR.ldf',
        MOVE 'CustomerOrderDB_Data2' TO 'E:\SQLData\CustomerOrderDB_Data2.ndf',
        MOVE 'CustomerOrderDB_Data3' TO 'E:\SQLData\CustomerOrderDB_Data3.ndf',
        NORECOVERY,
        STATS = 5;
END

-- Step 4: Restore transaction log backups up to point-in-time
-- Get all log backups needed for recovery
DECLARE @LogBackupFile NVARCHAR(500);

DECLARE log_recovery_cursor CURSOR FOR
SELECT bmf.physical_device_name
FROM msdb.dbo.backupset bs
INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.database_name = 'CustomerOrderDB'
AND bs.type = 'L'
AND bs.backup_finish_date > @PointInTime
AND bs.backup_finish_date <= @PointInTime
ORDER BY bs.backup_finish_date;

OPEN log_recovery_cursor;
FETCH NEXT FROM log_recovery_cursor INTO @LogBackupFile;

WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT 'Restoring log backup: ' + @LogBackupFile;
    
    RESTORE DATABASE CustomerOrderDB_DR
    FROM DISK = @LogBackupFile
    WITH 
        NORECOVERY,
        STATS = 5;
    
    FETCH NEXT FROM log_recovery_cursor INTO @LogBackupFile;
END

CLOSE log_recovery_cursor;
DEALLOCATE log_recovery_cursor;

-- Step 5: Complete recovery to point-in-time
RESTORE DATABASE CustomerOrderDB_DR
WITH RECOVERY,
     STOPAT = @PointInTime;

-- Calculate total recovery time
SET @TotalRecoveryTime = DATEDIFF(MINUTE, @StartTime, GETDATE());

PRINT 'Disaster recovery completed in ' + CAST(@TotalRecoveryTime AS VARCHAR(10)) + ' minutes';
```

3. **Post-Recovery Validation and Optimization**
```sql
-- Step 6: Validate recovered database
-- Quick integrity check
DBCC CHECKDB('CustomerOrderDB_DR') WITH NO_INFOMSGS;

-- Check data consistency
SELECT 
    'Order Table' as TableName,
    COUNT(*) as RecordCount
FROM CustomerOrderDB_DR.dbo.Orders
UNION ALL
SELECT 
    'Customer Table' as TableName,
    COUNT(*) as RecordCount
FROM CustomerOrderDB_DR.dbo.Customers
UNION ALL
SELECT 
    'OrderItems Table' as TableName,
    COUNT(*) as RecordCount
FROM CustomerOrderDB_DR.dbo.OrderItems;

-- Update statistics for performance
UPDATE STATISTICS CustomerOrderDB_DR.dbo.Orders WITH FULLSCAN;
UPDATE STATISTICS CustomerOrderDB_DR.dbo.Customers WITH FULLSCAN;
UPDATE STATISTICS CustomerOrderDB_DR.dbo.OrderItems WITH FULLSCAN;

-- Check index fragmentation and rebuild if needed
SELECT 
    OBJECT_NAME(ips.object_id) as TableName,
    i.name as IndexName,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    CASE 
        WHEN ips.avg_fragmentation_in_percent < 10 THEN 'No Action'
        WHEN ips.avg_fragmentation_in_percent < 30 THEN 'Reorganize'
        ELSE 'Rebuild'
    END as RecommendedAction
FROM sys.dm_db_index_physical_stats(DB_ID('CustomerOrderDB_DR'), NULL, NULL, NULL, 'DETAILED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 10
AND ips.page_count > 100
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

4. **Business Continuity Planning**
```sql
-- Step 7: Document recovery statistics
INSERT INTO dbo.DisasterRecoveryLog (
    RecoveryDate,
    DatabaseName,
    RecoveryType,
    RTO_Target_Minutes,
    RPO_Target_Minutes,
    ActualRecoveryTime_Minutes,
    EstimatedDataLoss_Minutes,
    Success,
    BackupMethod,
    RecoveryMethod,
    IssuesEncountered,
    PostRecoveryActions
)
VALUES (
    GETDATE(),
    'CustomerOrderDB',
    'FULL_DISASTER_RECOVERY',
    120,  -- 2 hour RTO
    30,   -- 30 minute RPO
    @TotalRecoveryTime,
    30,   -- Estimated 30 minutes data loss based on RPO
    1,    -- Success
    'FULL_DIFFERENTIAL_LOG',
    'POINT_IN_TIME_RECOVERY',
    'None',
    'Database integrity verified, statistics updated, ready for production'
);

-- Step 8: Implement temporary monitoring
CREATE PROCEDURE sp_MonitorRecoveredDatabase
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Monitor critical performance metrics for first 4 hours
    SELECT 
        GETDATE() as CheckTime,
        'CustomerOrderDB_DR' as DatabaseName,
        -- Memory usage
        (SELECT SUM(pages_kb) 
         FROM sys.dm_os_memory_clerks 
         WHERE database_id = DB_ID('CustomerOrderDB_DR'))/1024.0 as MemoryMB,
        -- CPU usage
        (SELECT SUM(cpu_time) 
         FROM sys.dm_exec_requests 
         WHERE database_id = DB_ID('CustomerOrderDB_DR'))/1000.0 as CPU_MS,
        -- Active connections
        (SELECT COUNT(*) 
         FROM sys.dm_exec_sessions 
         WHERE database_id = DB_ID('CustomerOrderDB_DR')) as ActiveConnections,
        -- Transaction log usage
        (SELECT CAST(FILEPROPERTY(name, 'SpaceUsed') AS FLOAT) * 8 / 1024.0
         FROM sys.database_files 
         WHERE type_desc = 'LOG') as LogUsedMB,
        -- Longest running query
        (SELECT TOP 1 DATEDIFF(SECOND, start_time, GETDATE())
         FROM sys.dm_exec_requests
         WHERE database_id = DB_ID('CustomerOrderDB_DR')
         ORDER BY start_time) as LongestQuerySeconds
    FROM sys.dm_os_sys_info;
END;
```

**Learning Outcome**: Executing rapid disaster recovery while maintaining business continuity and minimizing data loss during critical business periods

### Scenario 2: Healthcare System Compliance and Recovery

**Background**: MedCare Health System must maintain HIPAA compliance while implementing a comprehensive backup and recovery strategy that supports both routine operations and regulatory requirements.

**Requirements**:
- **Compliance**: HIPAA, SOX, and state healthcare regulations
- **Data Retention**: 7 years minimum for medical records
- **Audit Trail**: Complete audit of all backup and recovery operations
- **Security**: Encrypted backups, secure storage locations
- **Testing**: Monthly disaster recovery tests
- **Recovery**: 4-hour RTO, 1-hour RPO for critical patient data

**Challenges**:
- Large database size (500GB patient records)
- Complex regulatory requirements
- Multiple compliance audits
- Security and encryption requirements
- Regular testing requirements

**DBA Solution Implementation**:

1. **HIPAA-Compliant Backup Strategy**
```sql
-- Create compliant backup procedures
CREATE PROCEDURE sp_HIPAACompliantBackup
    @DatabaseName NVARCHAR(128)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @BackupFile NVARCHAR(500);
    DECLARE @ArchiveFile NVARCHAR(500);
    DECLARE @BackupStartTime DATETIME2 = GETDATE();
    DECLARE @BackupEndTime DATETIME2;
    DECLARE @BackupSuccess BIT = 1;
    DECLARE @ErrorMessage NVARCHAR(MAX) = '';
    
    -- Generate HIPAA-compliant backup file name
    SET @BackupFile = 'D:\HIPAA_Backups\' + @DatabaseName + '\Encrypted\' +
        @DatabaseName + '_HIPAA_' + 
        REPLACE(CONVERT(VARCHAR(8), GETDATE(), 112), '-', '_') + '.bak';
    
    -- Create directory if it doesn't exist
    EXEC xp_create_subdir 'D:\HIPAA_Backups\' + @DatabaseName + '\Encrypted';
    
    BEGIN TRY
        -- Step 1: Perform encrypted backup
        BACKUP DATABASE @DatabaseName
        TO DISK = @BackupFile
        WITH 
            INIT,
            ENCRYPTION (
                ALGORITHM = AES_256,
                SERVER CERTIFICATE = HIPAA_Backup_Cert
            ),
            COMPRESSION,
            CHECKSUM,
            STATS = 10,
            NAME = @DatabaseName + ' HIPAA Compliant Backup',
            DESCRIPTION = 'Encrypted backup for HIPAA compliance - ' + 
                         CONVERT(VARCHAR(20), GETDATE(), 120),
            EXPIRES = DATEADD(YEAR, 7, GETDATE());  -- 7-year retention
        
        -- Step 2: Verify backup integrity
        RESTORE VERIFYONLY
        FROM DISK = @BackupFile;
        
        SET @BackupEndTime = GETDATE();
        
        -- Step 3: Log to HIPAA audit trail
        INSERT INTO dbo.HIPAA_AuditLog (
            EventType,
            DatabaseName,
            BackupFile,
            BackupStartTime,
            BackupEndTime,
            BackupDurationSeconds,
            Success,
            ErrorMessage,
            ComplianceFlags,
            SecurityLevel,
            BackupType,
            RetentionPeriodYears,
            PerformedBy
        )
        VALUES (
            'DATABASE_BACKUP',
            @DatabaseName,
            @BackupFile,
            @BackupStartTime,
            @BackupEndTime,
            DATEDIFF(SECOND, @BackupStartTime, @BackupEndTime),
            1,
            '',
            '{"HIPAA": true, "SOX": true, "ENCRYPTED": true, "VERIFIED": true}',
            'HIGH',
            'FULL_DATABASE_BACKUP',
            7,
            SYSTEM_USER
        );
        
        -- Step 4: Archive to secure location
        -- Note: This would be implemented using PowerShell with encryption
        -- Archive-BackupToSecureLocation -SourcePath $BackupFile -DestinationPath "\\secure-archive\hipaa-backups\"
        
        PRINT 'HIPAA-compliant backup completed successfully';
        
    END TRY
    BEGIN CATCH
        SET @BackupSuccess = 0;
        SET @ErrorMessage = ERROR_MESSAGE();
        SET @BackupEndTime = GETDATE();
        
        -- Log failed backup attempt
        INSERT INTO dbo.HIPAA_AuditLog (
            EventType,
            DatabaseName,
            BackupFile,
            BackupStartTime,
            BackupEndTime,
            BackupDurationSeconds,
            Success,
            ErrorMessage,
            ComplianceFlags,
            SecurityLevel,
            PerformedBy
        )
        VALUES (
            'DATABASE_BACKUP_FAILED',
            @DatabaseName,
            @BackupFile,
            @BackupStartTime,
            @BackupEndTime,
            DATEDIFF(SECOND, @BackupStartTime, @BackupEndTime),
            0,
            @ErrorMessage,
            '{"HIPAA": true, "FAILED": true}',
            'CRITICAL',
            SYSTEM_USER
        );
        
        PRINT 'HIPAA-compliant backup failed: ' + @ErrorMessage;
        THROW;
    END CATCH
    
    -- Return backup summary
    SELECT 
        DatabaseName = @DatabaseName,
        BackupFile = @BackupFile,
        BackupStartTime = @BackupStartTime,
        BackupEndTime = @BackupEndTime,
        DurationSeconds = DATEDIFF(SECOND, @BackupStartTime, @BackupEndTime),
        Success = @BackupSuccess,
        ComplianceLevel = 'HIPAA_COMPLIANT',
        RetentionYears = 7;
END;
```

2. **Automated Compliance Monitoring**
```sql
-- Create compliance monitoring table
CREATE TABLE dbo.HIPAA_ComplianceMonitoring (
    MonitoringID BIGINT IDENTITY(1,1) PRIMARY KEY,
    CheckDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    DatabaseName NVARCHAR(128) NOT NULL,
    CheckType NVARCHAR(50) NOT NULL,
    CheckResult NVARCHAR(20) NOT NULL, -- PASS, FAIL, WARNING
    Details NVARCHAR(MAX),
    RemediationRequired BIT DEFAULT 0,
    FollowUpDate DATETIME2,
    VerifiedBy NVARCHAR(128) NOT NULL
);

-- Create compliance checking procedure
CREATE PROCEDURE sp_HIPAAComplianceCheck
    @DatabaseName NVARCHAR(128)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @ComplianceScore INT = 0;
    DECLARE @TotalChecks INT = 0;
    
    -- Check 1: Verify backup encryption
    SET @TotalChecks = @TotalChecks + 1;
    IF EXISTS (
        SELECT 1 
        FROM msdb.dbo.backupset bs
        INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
        WHERE bs.database_name = @DatabaseName
        AND bs.backup_start_date >= DATEADD(DAY, -1, GETDATE())
        AND bs.is_encrypted = 1
    )
    BEGIN
        SET @ComplianceScore = @ComplianceScore + 1;
        
        INSERT INTO dbo.HIPAA_ComplianceMonitoring (
            DatabaseName, CheckType, CheckResult, Details, VerifiedBy
        )
        VALUES (
            @DatabaseName, 'BACKUP_ENCRYPTION', 'PASS',
            'Recent backups are encrypted', SYSTEM_USER
        );
    END
    ELSE
    BEGIN
        INSERT INTO dbo.HIPAA_ComplianceMonitoring (
            DatabaseName, CheckType, CheckResult, Details, RemediationRequired, VerifiedBy
        )
        VALUES (
            @DatabaseName, 'BACKUP_ENCRYPTION', 'FAIL',
            'Recent backups are not encrypted', 1, SYSTEM_USER
        );
    END
    
    -- Check 2: Verify backup retention compliance
    SET @TotalChecks = @TotalChecks + 1;
    DECLARE @OldestRequiredBackup DATETIME2 = DATEADD(YEAR, -7, GETDATE()); -- 7 years
    
    IF EXISTS (
        SELECT 1 
        FROM msdb.dbo.backupset
        WHERE database_name = @DatabaseName
        AND backup_start_date >= @OldestRequiredBackup
        AND type = 'D'
    )
    BEGIN
        SET @ComplianceScore = @ComplianceScore + 1;
        
        INSERT INTO dbo.HIPAA_ComplianceMonitoring (
            DatabaseName, CheckType, CheckResult, Details, VerifiedBy
        )
        VALUES (
            @DatabaseName, 'RETENTION_COMPLIANCE', 'PASS',
            'Backup retention meets 7-year requirement', SYSTEM_USER
        );
    END
    ELSE
    BEGIN
        INSERT INTO dbo.HIPAA_ComplianceMonitoring (
            DatabaseName, CheckType, CheckResult, Details, RemediationRequired, VerifiedBy
        )
        VALUES (
            @DatabaseName, 'RETENTION_COMPLIANCE', 'FAIL',
            'Backup retention does not meet 7-year requirement', 1, SYSTEM_USER
        );
    END
    
    -- Check 3: Verify audit trail completeness
    SET @TotalChecks = @TotalChecks + 1;
    IF EXISTS (
        SELECT 1 
        FROM dbo.HIPAA_AuditLog
        WHERE DatabaseName = @DatabaseName
        AND EventDate >= DATEADD(DAY, -7, GETDATE())
    )
    BEGIN
        SET @ComplianceScore = @ComplianceScore + 1;
        
        INSERT INTO dbo.HIPAA_ComplianceMonitoring (
            DatabaseName, CheckType, CheckResult, Details, VerifiedBy
        )
        VALUES (
            @DatabaseName, 'AUDIT_TRAIL', 'PASS',
            'Audit trail is complete and recent', SYSTEM_USER
        );
    END
    
    -- Check 4: Verify disaster recovery testing
    SET @TotalChecks = @TotalChecks + 1;
    IF EXISTS (
        SELECT 1 
        FROM dbo.DRTestResults
        WHERE DatabaseName = @DatabaseName
        AND TestDate >= DATEADD(MONTH, -1, GETDATE())
        AND Success = 1
    )
    BEGIN
        SET @ComplianceScore = @ComplianceScore + 1;
        
        INSERT INTO dbo.HIPAA_ComplianceMonitoring (
            DatabaseName, CheckType, CheckResult, Details, VerifiedBy
        )
        VALUES (
            @DatabaseName, 'DR_TESTING', 'PASS',
            'Recent disaster recovery testing completed', SYSTEM_USER
        );
    END
    
    -- Calculate overall compliance score
    DECLARE @CompliancePercentage DECIMAL(5,2) = (@ComplianceScore * 100.0 / @TotalChecks);
    
    -- Log overall compliance check
    INSERT INTO dbo.HIPAA_ComplianceMonitoring (
        DatabaseName, CheckType, CheckResult, Details, VerifiedBy
    )
    VALUES (
        @DatabaseName, 'OVERALL_COMPLIANCE',
        CASE 
            WHEN @CompliancePercentage >= 90 THEN 'PASS'
            WHEN @CompliancePercentage >= 75 THEN 'WARNING'
            ELSE 'FAIL'
        END,
        'Overall compliance score: ' + CAST(@CompliancePercentage AS VARCHAR(10)) + '% (' + 
        CAST(@ComplianceScore AS VARCHAR(10)) + '/' + CAST(@TotalChecks AS VARCHAR(10)) + ' checks passed)',
        SYSTEM_USER
    );
    
    -- Return compliance summary
    SELECT 
        DatabaseName = @DatabaseName,
        ComplianceScore = @ComplianceScore,
        TotalChecks = @TotalChecks,
        CompliancePercentage = @CompliancePercentage,
        Status = CASE 
            WHEN @CompliancePercentage >= 90 THEN 'COMPLIANT'
            WHEN @CompliancePercentage >= 75 THEN 'NEEDS_ATTENTION'
            ELSE 'NON_COMPLIANT'
        END,
        CheckDate = GETDATE();
END;
```

3. **Monthly DR Testing for Compliance**
```sql
-- Create monthly DR test procedure for compliance
CREATE PROCEDURE sp_MonthlyHIPAA_DRTest
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @TestStartTime DATETIME2 = GETDATE();
    DECLARE @TestDBName NVARCHAR(128);
    DECLARE @Success BIT = 1;
    DECLARE @ErrorMessage NVARCHAR(MAX) = '';
    
    -- Generate test database name
    SET @TestDBName = 'HIPAA_DR_TEST_' + 
        REPLACE(CONVERT(VARCHAR(8), GETDATE(), 112), '-', '_');
    
    BEGIN TRY
        -- Step 1: Use most recent encrypted backup
        DECLARE @LatestBackupFile NVARCHAR(500);
        
        SELECT TOP 1 @LatestBackupFile = bmf.physical_device_name
        FROM msdb.dbo.backupset bs
        INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
        WHERE bs.database_name = 'PatientDB'
        AND bs.is_encrypted = 1  -- Ensure encrypted backup
        ORDER BY bs.backup_finish_date DESC;
        
        IF @LatestBackupFile IS NULL
            RAISERROR('No encrypted backup found for DR test', 16, 1);
        
        -- Step 2: Restore encrypted backup to test environment
        RESTORE DATABASE @TestDBName
        FROM DISK = @LatestBackupFile
        WITH 
            MOVE 'PatientDB' TO 'D:\DR_Testing\' + @TestDBName + '.mdf',
            MOVE 'PatientDB_Log' TO 'D:\DR_Testing\' + @TestDBName + '.ldf',
            NORECOVERY,
            REPLACE,
            STATS = 10;
        
        -- Step 3: Restore transaction logs if needed for point-in-time recovery
        -- This would follow the same pattern as disaster recovery
        
        RESTORE DATABASE @TestDBName WITH RECOVERY;
        
        -- Step 4: Comprehensive data integrity verification
        DBCC CHECKDB(@TestDBName) WITH NO_INFOMSGS, DATA_PURITY;
        
        -- Step 5: Verify HIPAA compliance in test environment
        -- Ensure patient data is properly protected
        
        -- Step 6: Measure recovery time and log results
        DECLARE @RecoveryTime INT = DATEDIFF(MINUTE, @TestStartTime, GETDATE());
        
        -- Log DR test results
        INSERT INTO dbo.HIPAA_AuditLog (
            EventType,
            DatabaseName,
            EventDate,
            Details,
            SecurityLevel,
            ComplianceFlags,
            PerformedBy
        )
        VALUES (
            'MONTHLY_DR_TEST',
            'PatientDB',
            GETDATE(),
            'Monthly DR test completed successfully. Recovery time: ' + 
            CAST(@RecoveryTime AS VARCHAR(10)) + ' minutes. Test database: ' + @TestDBName,
            'HIGH',
            '{"HIPAA_COMPLIANCE": true, "DR_TESTING": true, "DATA_INTEGRITY": true}',
            SYSTEM_USER
        );
        
        -- Generate compliance report
        SELECT 
            TestDate = GETDATE(),
            DatabaseName = 'PatientDB',
            TestType = 'MONTHLY_HIPAA_DR',
            RecoveryTimeMinutes = @RecoveryTime,
            Success = 1,
            TestDatabase = @TestDBName,
            ComplianceStatus = 'PASS',
            Notes = 'Monthly DR test completed successfully for HIPAA compliance';
        
    END TRY
    BEGIN CATCH
        SET @Success = 0;
        SET @ErrorMessage = ERROR_MESSAGE();
        
        -- Log failed DR test
        INSERT INTO dbo.HIPAA_AuditLog (
            EventType,
            DatabaseName,
            EventDate,
            Details,
            SecurityLevel,
            ComplianceFlags,
            PerformedBy
        )
        VALUES (
            'MONTHLY_DR_TEST_FAILED',
            'PatientDB',
            GETDATE(),
            'Monthly DR test failed: ' + @ErrorMessage,
            'CRITICAL',
            '{"HIPAA_COMPLIANCE": false, "DR_TESTING_FAILED": true}',
            SYSTEM_USER
        );
        
        -- Clean up failed test database
        IF EXISTS (SELECT 1 FROM sys.databases WHERE name = @TestDBName)
        BEGIN
            ALTER DATABASE @TestDBName SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
            DROP DATABASE @TestDBName;
        END
        
        THROW;
    END CATCH
    
    -- Return test results
    SELECT 
        TestDate = GETDATE(),
        DatabaseName = 'PatientDB',
        TestType = 'MONTHLY_HIPAA_DR',
        RecoveryTimeMinutes = DATEDIFF(MINUTE, @TestStartTime, GETDATE()),
        Success = @Success,
        ErrorMessage = @ErrorMessage;
END;
```

**Learning Outcome**: Implementing HIPAA-compliant backup and recovery procedures with comprehensive auditing, encryption, and regular testing to meet regulatory requirements

### Scenario 3: Multi-Region Financial Services Recovery

**Background**: GlobalFinance operates across 8 regions with strict financial regulations (SOX, PCI DSS, Basel III). They need a sophisticated backup and recovery strategy that supports cross-border compliance and disaster recovery across multiple time zones.

**Requirements**:
- **Global Operations**: 8 regional data centers
- **Regulatory Compliance**: SOX (US), PCI DSS (Payment), Basel III (International)
- **Recovery Strategy**: Local recovery within 1 hour, global failover within 4 hours
- **Data Sovereignty**: Regional data must remain within regulatory boundaries
- **Audit Trail**: Complete transaction audit trail for compliance
- **Testing**: Quarterly global DR exercises

**Challenges**:
- Different regulatory requirements per region
- Data residency constraints
- Cross-border data transfer restrictions
- Real-time synchronization requirements
- Multiple backup technologies and procedures

**DBA Solution Implementation**:

1. **Multi-Region Backup Strategy**
```sql
-- Create region-specific backup procedures
CREATE PROCEDURE sp_RegionalBackup
    @RegionCode NVARCHAR(3), -- US, EU, APAC, etc.
    @DatabaseName NVARCHAR(128),
    @BackupType NVARCHAR(20) -- FULL, DIFF, LOG
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @BackupPath NVARCHAR(500);
    DECLARE @ArchivePath NVARCHAR(500);
    DECLARE @ComplianceLevel NVARCHAR(50);
    DECLARE @DataSovereignty NVARCHAR(100);
    
    -- Determine paths based on region and compliance requirements
    SET @BackupPath = CASE @RegionCode
        WHEN 'US' THEN 'E:\RegionalBackups\US\' + @DatabaseName + '\'
        WHEN 'EU' THEN 'F:\RegionalBackups\EU\' + @DatabaseName + '\'
        WHEN 'APAC' THEN 'G:\RegionalBackups\APAC\' + @DatabaseName + '\'
        ELSE 'H:\RegionalBackups\Other\' + @DatabaseName + '\'
    END;
    
    SET @ComplianceLevel = CASE @RegionCode
        WHEN 'US' THEN 'SOX_COMPLIANT'
        WHEN 'EU' THEN 'GDPR_COMPLIANT'
        WHEN 'APAC' THEN 'BASEL_III_COMPLIANT'
        ELSE 'BASIC_COMPLIANCE'
    END;
    
    SET @DataSovereignty = CASE @RegionCode
        WHEN 'US' THEN 'DATA_MUST_REMAIN_US'
        WHEN 'EU' THEN 'GDPR_DATA_RESIDENCY'
        ELSE 'REGIONAL_DATA_SOVEREIGNTY'
    END;
    
    -- Generate backup file name with region code
    DECLARE @BackupFile NVARCHAR(500) = @BackupPath + 
        @DatabaseName + '_' + @RegionCode + '_' + @BackupType + '_' + 
        REPLACE(CONVERT(VARCHAR(8), GETDATE(), 112), '-', '_') + '.bak';
    
    -- Create regional directory structure
    EXEC xp_create_subdir @BackupPath;
    
    -- Generate encryption certificate based on region
    DECLARE @CertName NVARCHAR(128) = 'RegionalBackup_' + @RegionCode + '_Cert';
    
    BEGIN TRY
        -- Execute backup with region-specific encryption
        IF @BackupType = 'FULL'
        BEGIN
            BACKUP DATABASE @DatabaseName
            TO DISK = @BackupFile
            WITH 
                INIT,
                ENCRYPTION (
                    ALGORITHM = AES_256,
                    SERVER CERTIFICATE = @CertName
                ),
                COMPRESSION,
                CHECKSUM,
                STATS = 10,
                NAME = @DatabaseName + ' ' + @RegionCode + ' Regional Backup',
                DESCRIPTION = @BackupType + ' backup for ' + @RegionCode + ' region - ' + @ComplianceLevel,
                EXPIRES = DATEADD(YEAR, 7, GETDATE()); -- 7-year retention for financial records
        END
        ELSE IF @BackupType = 'LOG'
        BEGIN
            BACKUP LOG @DatabaseName
            TO DISK = @BackupFile
            WITH 
                INIT,
                ENCRYPTION (
                    ALGORITHM = AES_256,
                    SERVER CERTIFICATE = @CertName
                ),
                COMPRESSION,
                CHECKSUM,
                STATS = 10,
                NAME = @DatabaseName + ' ' + @RegionCode + ' Transaction Log Backup',
                DESCRIPTION = 'Log backup for ' + @RegionCode + ' region - ' + @DataSovereignty,
                EXPIRES = DATEADD(YEAR, 7, GETDATE());
        END
        
        -- Log regional backup to compliance audit
        INSERT INTO dbo.GlobalComplianceAudit (
            EventType,
            RegionCode,
            DatabaseName,
            BackupFile,
            ComplianceLevel,
            DataSovereignty,
            BackupType,
            EncryptionAlgorithm,
            RetentionPeriodYears,
            RegionalBackupLocation,
            BackupStartTime,
            BackupEndTime,
            Success,
            PerformedBy,
            RegulatoryFlags
        )
        VALUES (
            'REGIONAL_BACKUP',
            @RegionCode,
            @DatabaseName,
            @BackupFile,
            @ComplianceLevel,
            @DataSovereignty,
            @BackupType,
            'AES_256',
            7,
            @BackupPath,
            GETDATE(),
            GETDATE(),
            1,
            SYSTEM_USER,
            JSON_QUERY(
                OBJECT_QUERY(
                    '{SOX: @sox, PCI_DSS: @pci, GDPR: @gdpr, Basel_III: @basel}',
                    '$.sox', CASE WHEN @RegionCode = 'US' THEN true ELSE false END,
                    '$.pci', CASE WHEN @RegionCode = 'US' THEN true ELSE false END,
                    '$.gdpr', CASE WHEN @RegionCode = 'EU' THEN true ELSE false END,
                    '$.basel', CASE WHEN @RegionCode IN ('EU', 'APAC') THEN true ELSE false END
                )
            )
        );
        
        -- Create cross-region replication log entry
        -- Note: Actual replication would be implemented via log shipping or Always On
        INSERT INTO dbo.CrossRegionReplicationLog (
            SourceRegion,
            TargetRegion,
            DatabaseName,
            ReplicationType,
            StartTime,
            CompletionTime,
            Status,
            DataTransferMB,
            NetworkLatencyMS
        )
        VALUES (
            @RegionCode,
            'GLOBAL_ARCHIVE', -- Global archive region
            @DatabaseName,
            'BACKUP_ARCHIVAL',
            GETDATE(),
            GETDATE(),
            'COMPLETED',
            (SELECT backup_size/1024/1024 
             FROM msdb.dbo.backupset 
             WHERE database_name = @DatabaseName 
             AND backup_start_date >= DATEADD(SECOND, -60, GETDATE())),
            150 -- Simulated network latency
        );
        
        PRINT @BackupType + ' backup completed for ' + @RegionCode + ' region: ' + @BackupFile;
        
    END TRY
    BEGIN CATCH
        -- Log backup failure
        INSERT INTO dbo.GlobalComplianceAudit (
            EventType,
            RegionCode,
            DatabaseName,
            BackupFile,
            ComplianceLevel,
            BackupType,
            BackupStartTime,
            BackupEndTime,
            Success,
            ErrorMessage,
            PerformedBy,
            SeverityLevel
        )
        VALUES (
            'REGIONAL_BACKUP_FAILED',
            @RegionCode,
            @DatabaseName,
            @BackupFile,
            @ComplianceLevel,
            @BackupType,
            GETDATE(),
            GETDATE(),
            0,
            ERROR_MESSAGE(),
            SYSTEM_USER,
            'CRITICAL'
        );
        
        -- Alert regional compliance team
        EXEC sp_SendComplianceAlert 
            @RegionCode, 
            'BACKUP_FAILURE', 
            'Regional backup failed for ' + @DatabaseName + ': ' + ERROR_MESSAGE();
        
        THROW;
    END CATCH
END;
```

2. **Global Disaster Recovery Coordination**
```sql
-- Create global DR coordination procedure
CREATE PROCEDURE sp_GlobalDisasterRecovery
    @PrimaryRegion NVARCHAR(3),
    @FailoverRegion NVARCHAR(3),
    @DatabaseName NVARCHAR(128),
    @RecoveryType NVARCHAR(20) -- LOCAL_FAILOVER, REGIONAL_FAILOVER, GLOBAL_FAILOVER
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @DRSessionID INT = NEXT VALUE FOR dbo.GlobalDRSessionSequence;
    DECLARE @StartTime DATETIME2 = GETDATE();
    DECLARE @RecoveryCompleteTime DATETIME2;
    DECLARE @Success BIT = 1;
    DECLARE @ErrorMessage NVARCHAR(MAX) = '';
    DECLARE @TotalRecoveryTime INT;
    DECLARE @DataLossMinutes INT;
    
    -- Begin global DR session
    BEGIN TRY
        -- Step 1: Log DR session start
        INSERT INTO dbo.GlobalDRSession (
            DRSessionID,
            PrimaryRegion,
            FailoverRegion,
            DatabaseName,
            RecoveryType,
            SessionStartTime,
            SessionStatus,
            InitiatedBy,
            RegulatoryFlags
        )
        VALUES (
            @DRSessionID,
            @PrimaryRegion,
            @FailoverRegion,
            @DatabaseName,
            @RecoveryType,
            @StartTime,
            'IN_PROGRESS',
            SYSTEM_USER,
            JSON_QUERY('{"SOX": true, "PCI_DSS": true, "GLOBAL_COMPLIANCE": true}')
        );
        
        -- Step 2: Check cross-region backup availability
        DECLARE @PrimaryBackupFile NVARCHAR(500);
        DECLARE @FailoverBackupFile NVARCHAR(500);
        
        -- Get latest backup from primary region
        SELECT TOP 1 @PrimaryBackupFile = gca.BackupFile
        FROM dbo.GlobalComplianceAudit gca
        WHERE gca.RegionCode = @PrimaryRegion
        AND gca.DatabaseName = @DatabaseName
        AND gca.BackupType = 'FULL'
        AND gca.Success = 1
        ORDER BY gca.BackupStartTime DESC;
        
        -- Get latest backup from failover region (if cross-region replication is working)
        SELECT TOP 1 @FailoverBackupFile = gca.BackupFile
        FROM dbo.GlobalComplianceAudit gca
        WHERE gca.RegionCode = @FailoverRegion
        AND gca.DatabaseName = @DatabaseName
        AND gca.BackupType = 'FULL'
        AND gca.Success = 1
        ORDER BY gca.BackupStartTime DESC;
        
        -- Step 3: Determine backup source (prefer local, use cross-region if necessary)
        DECLARE @SelectedBackupFile NVARCHAR(500);
        DECLARE @BackupSourceRegion NVARCHAR(3);
        
        IF @FailoverBackupFile IS NOT NULL AND 
           DATEDIFF(HOUR, (SELECT MAX(BackupStartTime) FROM dbo.GlobalComplianceAudit 
                           WHERE BackupFile = @PrimaryBackupFile), GETDATE()) > 2
        BEGIN
            -- Primary region backup is too old, use cross-region backup
            SET @SelectedBackupFile = @FailoverBackupFile;
            SET @BackupSourceRegion = @FailoverRegion;
        END
        ELSE
        BEGIN
            -- Use primary region backup
            SET @SelectedBackupFile = @PrimaryBackupFile;
            SET @BackupSourceRegion = @PrimaryRegion;
        END
        
        -- Step 4: Restore database to failover region
        RESTORE DATABASE @DatabaseName + '_DR_' + @DRSessionID
        FROM DISK = @SelectedBackupFile
        WITH 
            MOVE @DatabaseName TO 'D:\DR_Failover\' + @DatabaseName + '_DR_' + @DRSessionID + '.mdf',
            MOVE @DatabaseName + '_Log' TO 'D:\DR_Failover\' + @DatabaseName + '_DR_' + @DRSessionID + '.ldf',
            NORECOVERY,
            REPLACE,
            STATS = 5;
        
        -- Step 5: Restore transaction logs up to point of failure
        DECLARE @LogBackupFile NVARCHAR(500);
        
        -- Get transaction logs since last full backup
        DECLARE log_restoration_cursor CURSOR FOR
        SELECT gca.BackupFile
        FROM dbo.GlobalComplianceAudit gca
        WHERE gca.RegionCode = @BackupSourceRegion
        AND gca.DatabaseName = @DatabaseName
        AND gca.BackupType = 'LOG'
        AND gca.BackupStartTime > (
            SELECT MAX(BackupStartTime)
            FROM dbo.GlobalComplianceAudit
            WHERE DatabaseName = @DatabaseName
            AND BackupType = 'FULL'
            AND Success = 1
        )
        AND gca.Success = 1
        ORDER BY gca.BackupStartTime;
        
        OPEN log_restoration_cursor;
        FETCH NEXT FROM log_restoration_cursor INTO @LogBackupFile;
        
        WHILE @@FETCH_STATUS = 0
        BEGIN
            RESTORE LOG @DatabaseName + '_DR_' + @DRSessionID
            FROM DISK = @LogBackupFile
            WITH 
                NORECOVERY,
                STATS = 5;
            
            FETCH NEXT FROM log_restoration_cursor INTO @LogBackupFile;
        END
        
        CLOSE log_restoration_cursor;
        DEALLOCATE log_restoration_cursor;
        
        -- Step 6: Complete recovery
        RESTORE DATABASE @DatabaseName + '_DR_' + @DRSessionID WITH RECOVERY;
        
        SET @RecoveryCompleteTime = GETDATE();
        SET @TotalRecoveryTime = DATEDIFF(MINUTE, @StartTime, @RecoveryCompleteTime);
        SET @DataLossMinutes = DATEDIFF(MINUTE, (SELECT MAX(BackupStartTime) 
                                                FROM dbo.GlobalComplianceAudit 
                                                WHERE BackupFile = @LogBackupFile), GETDATE());
        
        -- Step 7: Verify database integrity and compliance
        DBCC CHECKDB(@DatabaseName + '_DR_' + @DRSessionID) WITH NO_INFOMSGS, DATA_PURITY;
        
        -- Step 8: Update DR session status
        UPDATE dbo.GlobalDRSession
        SET SessionEndTime = @RecoveryCompleteTime,
            SessionStatus = 'COMPLETED',
            ActualRecoveryTimeMinutes = @TotalRecoveryTime,
            EstimatedDataLossMinutes = @DataLossMinutes,
            RecoveryDetails = 'Database restored to ' + @FailoverRegion + ' region successfully'
        WHERE DRSessionID = @DRSessionID;
        
        -- Step 9: Log global DR completion
        INSERT INTO dbo.GlobalComplianceAudit (
            EventType,
            RegionCode,
            DatabaseName,
            ComplianceLevel,
            DataSovereignty,
            EventDate,
            Details,
            SecurityLevel,
            RegulatoryFlags,
            PerformedBy,
            DRSessionID
        )
        VALUES (
            'GLOBAL_DISASTER_RECOVERY',
            @FailoverRegion,
            @DatabaseName,
            'GLOBAL_COMPLIANCE',
            'CROSS_REGION_FAILOVER',
            GETDATE(),
            'Global disaster recovery completed. Recovery time: ' + CAST(@TotalRecoveryTime AS VARCHAR(10)) + 
            ' minutes. Data loss: ' + CAST(@DataLossMinutes AS VARCHAR(10)) + ' minutes.',
            'CRITICAL',
            '{"GLOBAL_FAILOVER": true, "CROSS_REGION_COMPLIANCE": true}',
            SYSTEM_USER,
            @DRSessionID
        );
        
        PRINT 'Global disaster recovery completed successfully in ' + 
              CAST(@TotalRecoveryTime AS VARCHAR(10)) + ' minutes';
        
    END TRY
    BEGIN CATCH
        SET @Success = 0;
        SET @ErrorMessage = ERROR_MESSAGE();
        SET @RecoveryCompleteTime = GETDATE();
        
        -- Log DR failure
        UPDATE dbo.GlobalDRSession
        SET SessionEndTime = @RecoveryCompleteTime,
            SessionStatus = 'FAILED',
            ErrorDetails = @ErrorMessage,
            ActualRecoveryTimeMinutes = DATEDIFF(MINUTE, @StartTime, @RecoveryCompleteTime)
        WHERE DRSessionID = @DRSessionID;
        
        -- Log critical failure
        INSERT INTO dbo.GlobalComplianceAudit (
            EventType,
            RegionCode,
            DatabaseName,
            EventDate,
            Details,
            SecurityLevel,
            SeverityLevel,
            PerformedBy,
            DRSessionID
        )
        VALUES (
            'GLOBAL_DISASTER_RECOVERY_FAILED',
            @FailoverRegion,
            @DatabaseName,
            GETDATE(),
            'Global disaster recovery failed: ' + @ErrorMessage,
            'CRITICAL',
            'CRITICAL',
            SYSTEM_USER,
            @DRSessionID
        );
        
        -- Trigger global alert system
        EXEC sp_TriggerGlobalAlert 
            @PrimaryRegion, 
            @FailoverRegion, 
            'DISASTER_RECOVERY_FAILURE', 
            @ErrorMessage;
        
        THROW;
    END CATCH
    
    -- Return DR session results
    SELECT 
        DRSessionID,
        DatabaseName = @DatabaseName,
        PrimaryRegion = @PrimaryRegion,
        FailoverRegion = @FailoverRegion,
        RecoveryType = @RecoveryType,
        TotalRecoveryTimeMinutes = @TotalRecoveryTime,
        EstimatedDataLossMinutes = @DataLossMinutes,
        Success = @Success,
        CompletionTime = @RecoveryCompleteTime,
        Status = CASE WHEN @Success = 1 THEN 'COMPLETED' ELSE 'FAILED' END;
END;
```

**Learning Outcome**: Implementing sophisticated multi-region backup and disaster recovery strategies that maintain compliance across different regulatory environments while ensuring business continuity

## Summary and Key Takeaways

This week provided comprehensive coverage of SQL Server backup and recovery fundamentals:

1. **Recovery Models**: Understanding Simple, Full, and Bulk-Logged recovery models and their implications for backup strategies and recovery capabilities
2. **Backup Types**: Full, differential, and transaction log backups and when to use each type
3. **Backup Media Management**: File organization, compression, encryption, and retention policies
4. **Recovery Procedures**: Complete restore, point-in-time recovery, and file-level recovery techniques
5. **Disaster Recovery Planning**: RTO/RPO planning, testing procedures, and high availability considerations
6. **Real-World Applications**: Implementing backup strategies for e-commerce, healthcare, and financial services environments

Key backup and recovery principles:
- **Know Your RTO/RPO**: Design backup strategies around business recovery requirements
- **Test Regularly**: Regular disaster recovery testing ensures procedures work when needed
- **Encrypt Everything**: Protect sensitive data both in transit and at rest
- **Document Everything**: Comprehensive documentation of procedures, decisions, and test results
- **Monitor Continuously**: Ongoing monitoring of backup success, storage usage, and recovery times
- **Plan for Growth**: Backup strategies must accommodate database growth and changing requirements

Backup and recovery is not just a technical task but a critical business function that requires careful planning, regular testing, and continuous improvement.

## Additional Resources

### Microsoft Documentation
- SQL Server Backup and Restore: https://docs.microsoft.com/sql/relational-databases/backup-restore/sql-server-backup-and-restore
- Recovery Models: https://docs.microsoft.com/sql/relational-databases/backup-restore/recovery-models-sql-server
- Always On Backup Preferences: https://docs.microsoft.com/sql/database-engine/availability-groups/windows/backup-on-secondary-replicas-always-on-availability-groups

### Backup Strategy Resources
- SQL Server Backup Best Practices: Industry whitepapers and best practice guides
- Disaster Recovery Planning Templates: Available from various consulting firms
- Backup Monitoring Tools: SQL Server tools and third-party solutions

### Testing and Validation
- Database Consistency Check (DBCC CHECKDB): For data integrity verification
- Backup Verification Procedures: RESTORE VERIFYONLY and other validation methods
- Performance Testing Tools: For measuring backup and recovery performance

---

**Next Week Preview**: Monitoring and Performance - We'll explore SQL Server monitoring techniques, performance optimization strategies, and proactive maintenance procedures for optimal database performance.