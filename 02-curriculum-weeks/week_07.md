# Week 07: Advanced Backup Strategies

## Overview

This week focuses on implementing comprehensive backup and recovery strategies for SQL Server databases. As a Database Administrator, you'll master backup automation, offsite backup solutions, and disaster recovery planning to ensure business continuity and data protection.

## Learning Objectives

By the end of this week, you will be able to:
- Design and implement automated backup strategies for various database scenarios
- Configure and manage offsite backup solutions for disaster recovery
- Develop comprehensive disaster recovery plans with recovery time objectives (RTO) and recovery point objectives (RPO)
- Implement backup compression and encryption for data security
- Monitor backup operations and troubleshoot backup failures
- Create backup verification and testing procedures

## Monday - Backup Fundamentals and Strategy Design

### Understanding Backup Types and Strategies

#### Full Backups
Full backups create a complete copy of your database, including all data, schema, and transaction log information. These backups serve as the foundation for any disaster recovery strategy.

**Key Characteristics:**
- Contains all database objects and data
- Provides complete point-in-time recovery capability
- Serves as baseline for differential and transaction log backups
- Can be restored independently without other backup files

**Implementation Example:**
```sql
-- Basic full backup
BACKUP DATABASE SalesDB
TO DISK = 'C:\Backups\SalesDB_Full_20250101.bak'
WITH FORMAT,
     COMPRESSION,
     CHECKSUM,
     STATS = 10;

-- Full backup with custom options
BACKUP DATABASE SalesDB
TO DISK = 'C:\Backups\SalesDB_Full_20250101.bak'
WITH 
    FORMAT,
    COMPRESSION,
    CHECKSUM,
    STATS = 10,
    DESCRIPTION = 'Weekly Full Backup - Sales Database',
    EXPIREDATE = '2025-02-01',
    RETAINDAYS = 30;
```

#### Differential Backups
Differential backups capture only the changes made since the last full backup, making them faster and smaller than full backups while providing quicker recovery times.

**Key Characteristics:**
- Contains changes since last full backup
- Faster to create than full backups
- Reduces recovery time compared to transaction log restore sequence
- Must be used with the most recent full backup

**Implementation Example:**
```sql
-- Daily differential backup
BACKUP DATABASE SalesDB
TO DISK = 'C:\Backups\SalesDB_Diff_20250102.bak'
WITH 
    DIFFERENTIAL,
    COMPRESSION,
    CHECKSUM,
    STATS = 10,
    DESCRIPTION = 'Daily Differential Backup - Sales Database';
```

#### Transaction Log Backups
Transaction log backups are crucial for Point-in-Time Recovery (PITR) and enable you to recover data to any specific point in time.

**Key Characteristics:**
- Captures all transactions since last log backup
- Enables point-in-time recovery
- Required for databases in FULL or BULK_LOGGED recovery models
- Smallest backup type for the same time period

**Implementation Example:**
```sql
-- Transaction log backup every 15 minutes during business hours
BACKUP LOG SalesDB
TO DISK = 'C:\Backups\SalesDB_Log_20250102_0900.trn'
WITH 
    COMPRESSION,
    CHECKSUM,
    STATS = 10,
    DESCRIPTION = 'Transaction Log Backup - 9:00 AM';
```

### Backup Strategy Design Principles

#### Recovery Model Selection

**Full Recovery Model:**
- Use for production databases where data loss is unacceptable
- Supports point-in-time recovery
- Requires regular transaction log backups
- Highest overhead but maximum protection

```sql
-- Set database to full recovery model
ALTER DATABASE SalesDB
SET RECOVERY FULL;
```

**Simple Recovery Model:**
- Use for development/test databases or where data loss is acceptable
- Automatic truncation of transaction logs
- Cannot perform point-in-time recovery
- Lowest administrative overhead

```sql
-- Set database to simple recovery model
ALTER DATABASE SalesDB
SET RECOVERY SIMPLE;
```

**Bulk-Logged Recovery Model:**
- Use for databases with large bulk operations
- Better performance for bulk imports
- Limited point-in-time recovery capabilities
- Good balance between performance and protection

```sql
-- Set database to bulk-logged recovery model
ALTER DATABASE SalesDB
SET RECOVERY BULK_LOGGED;
```

#### Backup Frequency Planning

**High-Availability Databases:**
- Full backup: Weekly on Sunday
- Differential backup: Daily at 6:00 AM
- Transaction log backup: Every 15 minutes during business hours
- Log backup: Every 30 minutes during off-hours

**Standard Production Databases:**
- Full backup: Weekly on Sunday
- Differential backup: Daily at 2:00 AM
- Transaction log backup: Every 30 minutes
- Log backup: Hourly during off-hours

**Development/Test Databases:**
- Full backup: Weekly on Sunday
- No differential backups
- Simple recovery model (no log backups needed)

### Backup Media Strategy

#### Local Disk Storage
Store backups on local high-speed storage for quick access and verification.

```sql
-- Backup to local disk
BACKUP DATABASE SalesDB
TO DISK = 'C:\SQLBackups\SalesDB_Full_20250101.bak',
   DISK = 'E:\SQLBackups\SalesDB_Full_20250101.bak'
WITH 
    FORMAT,
    COMPRESSION,
    CHECKSUM,
    STATS = 10;
```

#### Network Storage
Use network-attached storage (NAS) or Storage Area Network (SAN) for shared backup access.

```sql
-- Backup to network share
BACKUP DATABASE SalesDB
TO DISK = '\\backupserver\backups\SalesDB_Full_20250101.bak'
WITH 
    FORMAT,
    COMPRESSION,
    CHECKSUM,
    STATS = 10;
```

## Tuesday - Backup Automation and Scheduling

### SQL Server Agent Jobs for Backup Automation

#### Creating Automated Backup Jobs

**Daily Full Backup Job:**
```sql
USE msdb;
GO

-- Create job category
EXEC sp_addcategory 
    @name = N'Database Maintenance';

-- Create daily full backup job
DECLARE @jobId BINARY(16);
EXEC sp_add_job
    @job_name = N'Daily Full Backup - All Databases',
    @enabled = 1,
    @description = N'Perform daily full backup of all user databases',
    @category_name = N'Database Maintenance',
    @owner_login_name = N'sa',
    @job_id = @jobId OUTPUT;

-- Add job step
EXEC sp_add_jobstep
    @job_id = @jobId,
    @step_name = N'Full Backup All Databases',
    @step_id = 1,
    @subsystem = N'CmdExec',
    @command = N'sqlcmd -S localhost -Q "EXEC sp_BackupAllDatabases @BackupType = ''FULL'', @BackupLocation = ''C:\Backups\DBA\'''" -E',
    @on_success_action = 1,
    @retry_attempts = 3,
    @retry_interval = 1;
```

**Transaction Log Backup Job:**
```sql
-- Create transaction log backup job for production databases
DECLARE @jobId BINARY(16);
EXEC sp_add_job
    @job_name = N'Transaction Log Backup - Production',
    @enabled = 1,
    @description = N'Backup transaction logs every 15 minutes for production databases',
    @category_name = N'Database Maintenance',
    @owner_login_name = N'sa',
    @job_id = @jobId OUTPUT;

-- Add job step
EXEC sp_add_jobstep
    @job_id = @jobId,
    @step_name = N'Log Backup Production Databases',
    @step_id = 1,
    @subsystem = N'CmdExec',
    @command = N'sqlcmd -S localhost -Q "EXEC sp_BackupProductionDatabases @BackupType = ''LOG'''" -E',
    @on_success_action = 1,
    @retry_attempts = 3,
    @retry_interval = 1;

-- Schedule job (every 15 minutes)
EXEC sp_add_schedule
    @schedule_name = N'Every 15 Minutes',
    @enabled = 1,
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15,
    @active_start_time = 0;

EXEC sp_attach_schedule
    @job_name = N'Transaction Log Backup - Production',
    @schedule_name = N'Every 15 Minutes';

-- Assign job to server
EXEC sp_add_jobserver
    @job_name = N'Transaction Log Backup - Production',
    @server_name = N'(LOCAL)';
```

#### Stored Procedure for Automated Backups

```sql
-- Create stored procedure for flexible backup operations
CREATE OR ALTER PROCEDURE sp_BackupAllDatabases
    @BackupType VARCHAR(10), -- FULL, DIFF, LOG
    @BackupLocation NVARCHAR(500) = 'C:\Backups\DBA\',
    @IncludeSystemDatabases BIT = 0,
    @Compression BIT = 1,
    @Checksum BIT = 1,
    @Verify BIT = 1
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @DatabaseName NVARCHAR(128);
    DECLARE @BackupFile NVARCHAR(500);
    DECLARE @BackupSQL NVARCHAR(MAX);
    DECLARE @DateString NVARCHAR(20);
    
    -- Generate date string for backup file naming
    SET @DateString = CONVERT(NVARCHAR(20), GETDATE(), 112) + '_' + 
                     REPLACE(CONVERT(NVARCHAR(20), GETDATE(), 108), ':', '');
    
    -- Cursor for all user databases
    DECLARE db_cursor CURSOR FOR
    SELECT name 
    FROM sys.databases 
    WHERE state = 0 -- Online databases
    AND database_id > 4 -- Exclude system databases unless specified
    AND name NOT IN ('ReportServer', 'ReportServerTempDB')
    ORDER BY name;
    
    OPEN db_cursor;
    FETCH NEXT FROM db_cursor INTO @DatabaseName;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Skip system databases unless explicitly requested
        IF @IncludeSystemDatabases = 0 AND @DatabaseName IN ('master', 'msdb', 'model', 'tempdb')
        BEGIN
            FETCH NEXT FROM db_cursor INTO @DatabaseName;
            CONTINUE;
        END
        
        -- Construct backup file path
        SET @BackupFile = @BackupLocation + @DatabaseName + '_' + @BackupType + '_' + @DateString + '.bak';
        
        -- Generate backup command
        SET @BackupSQL = 'BACKUP ' + 
            CASE @BackupType
                WHEN 'FULL' THEN 'DATABASE'
                WHEN 'DIFF' THEN 'DATABASE'
                WHEN 'LOG' THEN 'LOG'
            END + ' [' + @DatabaseName + '] ' +
            'TO DISK = ''' + @BackupFile + '''';
        
        -- Add backup options
        IF @Compression = 1
            SET @BackupSQL = @BackupSQL + ' WITH COMPRESSION';
        
        IF @Checksum = 1
            SET @BackupSQL = @BackupSQL + ', CHECKSUM';
        
        -- Add differential option for differential backups
        IF @BackupType = 'DIFF'
            SET @BackupSQL = @BackupSQL + ', DIFFERENTIAL';
        
        -- Add log-specific options for transaction log backups
        IF @BackupType = 'LOG'
            SET @BackupSQL = @BackupSQL + ', NORECOVER';
        
        -- Add backup description
        SET @BackupSQL = @BackupSQL + ', DESCRIPTION = ''Automated ' + @BackupType + ' backup - ' + 
                        @DatabaseName + ' - ' + CONVERT(NVARCHAR(20), GETDATE(), 120) + '''';
        
        -- Execute backup
        PRINT 'Backing up ' + @DatabaseName + ' (' + @BackupType + ') to ' + @BackupFile;
        
        BEGIN TRY
            EXEC sp_executesql @BackupSQL;
            
            -- Verify backup if requested
            IF @Verify = 1 AND @BackupType = 'FULL'
            BEGIN
                DECLARE @VerifySQL NVARCHAR(500);
                SET @VerifySQL = 'RESTORE VERIFYONLY FROM DISK = ''' + @BackupFile + '''';
                EXEC sp_executesql @VerifySQL;
                PRINT 'Backup verified for ' + @DatabaseName;
            END
            
            PRINT 'Backup completed successfully for ' + @DatabaseName;
        END TRY
        BEGIN CATCH
            PRINT 'ERROR backing up ' + @DatabaseName + ': ' + ERROR_MESSAGE();
        END CATCH
        
        FETCH NEXT FROM db_cursor INTO @DatabaseName;
    END
    
    CLOSE db_cursor;
    DEALLOCATE db_cursor;
    
    PRINT 'Backup process completed.';
END;
GO
```

### PowerShell Scripts for Advanced Backup Automation

#### Cross-Platform Backup Script
```powershell
# PowerShell script for advanced backup automation
# Save as: Backup-ProductionDatabases.ps1

param(
    [Parameter(Mandatory=$true)]
    [ValidateSet('Full', 'Differential', 'Log')]
    [string]$BackupType,
    
    [Parameter(Mandatory=$true)]
    [string]$BackupRootPath,
    
    [Parameter(Mandatory=$false)]
    [string]$InstanceName = 'localhost',
    
    [Parameter(Mandatory=$false)]
    [switch]$Compress,
    
    [Parameter(Mandatory=$false)]
    [switch]$Encrypt,
    
    [Parameter(Mandatory=$false)]
    [string]$EncryptionKeyPath,
    
    [Parameter(Mandatory=$false)]
    [switch]$Verify
)

# Function to log messages
function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
    
    # Also log to file
    $logFile = Join-Path $BackupRootPath "Backup_$(Get-Date -Format 'yyyyMMdd').log"
    Add-Content -Path $logFile -Value $logMessage
}

# Function to test SQL Server connection
function Test-SQLConnection {
    param([string]$Server, [string]$Database = 'master')
    try {
        $connectionString = "Server=$Server;Database=$Database;Integrated Security=True;Connection Timeout=5;"
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

# Function to get databases to backup
function Get-DatabasesToBackup {
    param([string]$Server, [string]$DatabaseType = 'User')
    
    $query = @"
    SELECT name, database_id, state_desc, recovery_model_desc
    FROM sys.databases
    WHERE state = 0 -- Online
    AND database_id > 4 -- Exclude system databases
"@
    
    if ($DatabaseType -eq 'All') {
        $query = @"
        SELECT name, database_id, state_desc, recovery_model_desc
        FROM sys.databases
        WHERE state = 0 -- Online
"@
    }
    
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
        Write-Log "Failed to retrieve database list: $($_.Exception.Message)" 'ERROR'
        return $null
    }
}

# Function to create backup
function New-DatabaseBackup {
    param(
        [string]$Server,
        [string]$Database,
        [string]$BackupType,
        [string]$BackupPath,
        [bool]$Compress = $false,
        [bool]$Encrypt = $false,
        [string]$EncryptionKey = '',
        [bool]$Verify = $false
    )
    
    $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    $backupFile = Join-Path $BackupPath "$Database`_$BackupType`_$timestamp.bak"
    
    # Create backup path if it doesn't exist
    if (-not (Test-Path $BackupPath)) {
        New-Item -ItemType Directory -Path $BackupPath -Force | Out-Null
    }
    
    # Build backup query
    $backupQuery = "BACKUP $BackupType [$Database] TO DISK = '$backupFile'"
    
    if ($Compress) {
        $backupQuery += " WITH COMPRESSION"
    }
    
    if ($Encrypt -and $EncryptionKey) {
        $backupQuery += " WITH ENCRYPTION (ALGORITHM = AES_256, SERVER CERTIFICATE = BackupEncryptionCert)"
    }
    
    if ($BackupType -eq 'DIFFERENTIAL') {
        $backupQuery += ", DIFFERENTIAL"
    }
    
    $backupQuery += ", STATS = 10"
    
    try {
        Write-Log "Starting backup of $Database ($BackupType) to $backupFile"
        
        $connectionString = "Server=$Server;Database=master;Integrated Security=True;"
        $connection = New-Object System.Data.SqlClient.SqlConnection($connectionString)
        $command = $connection.CreateCommand()
        $command.CommandText = $backupQuery
        $command.CommandTimeout = 0 # No timeout for backups
        $connection.Open()
        $command.ExecuteNonQuery() | Out-Null
        $connection.Close()
        
        # Verify backup if requested
        if ($Verify) {
            $verifyQuery = "RESTORE VERIFYONLY FROM DISK = '$backupFile'"
            $verifyConnection = New-Object System.Data.SqlClient.SqlConnection($connectionString)
            $verifyCommand = $verifyConnection.CreateCommand()
            $verifyCommand.CommandText = $verifyQuery
            $verifyConnection.Open()
            $verifyCommand.ExecuteNonQuery() | Out-Null
            $verifyConnection.Close()
        }
        
        Write-Log "Backup completed successfully for $Database" 'SUCCESS'
        return $backupFile
    }
    catch {
        Write-Log "Backup failed for $Database`: $($_.Exception.Message)" 'ERROR'
        return $null
    }
}

# Main backup process
try {
    Write-Log "Starting backup process - Type: $BackupType, Instance: $InstanceName"
    
    # Test connection
    if (-not (Test-SQLConnection -Server $InstanceName)) {
        exit 1
    }
    
    # Get databases to backup
    $databases = Get-DatabasesToBackup -Server $InstanceName
    
    if (-not $databases) {
        Write-Log "No databases found to backup" 'ERROR'
        exit 1
    }
    
    # Create backup directory for today
    $backupPath = Join-Path $BackupRootPath (Get-Date -Format 'yyyy-MM-dd')
    New-Item -ItemType Directory -Path $backupPath -Force | Out-Null
    
    # Perform backups
    $successfulBackups = @()
    $failedBackups = @()
    
    foreach ($db in $databases) {
        $backupFile = New-DatabaseBackup -Server $InstanceName -Database $db.name -BackupType $BackupType -BackupPath $backupPath -Compress $Compress.IsPresent -Encrypt $Encrypt.IsPresent -EncryptionKey $EncryptionKeyPath -Verify $Verify.IsPresent
        
        if ($backupFile) {
            $successfulBackups += [PSCustomObject]@{
                Database = $db.name
                BackupFile = $backupFile
                Size = (Get-Item $backupFile).Length / 1MB
                Timestamp = Get-Date
            }
        }
        else {
            $failedBackups += $db.name
        }
    }
    
    # Generate backup summary
    Write-Log "=== Backup Summary ==="
    Write-Log "Successful backups: $($successfulBackups.Count)"
    Write-Log "Failed backups: $($failedBackups.Count)"
    
    if ($successfulBackups.Count -gt 0) {
        Write-Log "Backup Details:"
        foreach ($backup in $successfulBackups) {
            Write-Log "  - $($backup.Database): $($backup.Size.ToString('F2')) MB"
        }
    }
    
    if ($failedBackups.Count -gt 0) {
        Write-Log "Failed Databases:"
        foreach ($db in $failedBackups) {
            Write-Log "  - $db"
        }
    }
    
    Write-Log "Backup process completed"
}
catch {
    Write-Log "Backup process failed: $($_.Exception.Message)" 'ERROR'
    exit 1
}
```

## Wednesday - Offsite Backup Solutions and Cloud Integration

### Azure Blob Storage Backup Integration

#### Configuring Azure Storage for SQL Server Backups

**Step 1: Create Azure Storage Account and Container**
```sql
-- Create credential for Azure Storage
CREATE CREDENTIAL AzureBackupCredentials
WITH IDENTITY = 'your_storage_account_name',
SECRET = 'your_access_key_here';
```

**Step 2: Backup to Azure Blob Storage**
```sql
-- Full backup to Azure Blob Storage
BACKUP DATABASE SalesDB
TO URL = 'https://mystorageaccount.blob.core.windows.net/sqlbackups/SalesDB_Full_20250101.bak'
WITH 
    FORMAT,
    COMPRESSION,
    CHECKSUM,
    STATS = 10,
    CREDENTIAL = 'AzureBackupCredentials',
    DESCRIPTION = 'Full backup to Azure Blob Storage';
    
-- Differential backup to Azure
BACKUP DATABASE SalesDB
TO URL = 'https://mystorageaccount.blob.core.windows.net/sqlbackups/SalesDB_Diff_20250101.bak'
WITH 
    DIFFERENTIAL,
    COMPRESSION,
    CHECKSUM,
    CREDENTIAL = 'AzureBackupCredentials';
    
-- Transaction log backup to Azure
BACKUP LOG SalesDB
TO URL = 'https://mystorageaccount.blob.core.windows.net/sqlbackups/SalesDB_Log_20250101_0900.trn'
WITH 
    COMPRESSION,
    CREDENTIAL = 'AzureBackupCredentials';
```

#### Automated Azure Backup Script
```powershell
# Azure SQL Server Backup Script
# Save as: Azure-SQLBackup.ps1

param(
    [Parameter(Mandatory=$true)]
    [string]$ServerName,
    
    [Parameter(Mandatory=$true)]
    [string]$StorageAccountName,
    
    [Parameter(Mandatory=$true)]
    [string]$StorageAccountKey,
    
    [Parameter(Mandatory=$true)]
    [string]$ContainerName,
    
    [Parameter(Mandatory=$true)]
    [ValidateSet('Full', 'Differential', 'Log')]
    [string]$BackupType,
    
    [Parameter(Mandatory=$false)]
    [string[]]$ExcludeDatabases = @(),
    
    [Parameter(Mandatory=$false)]
    [bool]$Compress = $true
)

# Import SQL Server module
Import-Module SqlServer -ErrorAction SilentlyContinue

# Function to create SAS token for Azure Blob Storage
function New-SAStoken {
    param(
        [string]$AccountName,
        [string]$AccountKey,
        [string]$ContainerName,
        [int]$ExpiryHours = 24
    )
    
    # Create SAS token using Azure Storage library
    $storageContext = New-AzureStorageContext -StorageAccountName $AccountName -StorageAccountKey $AccountKey
    $sasToken = New-AzureStorageBlobSASToken -Container $ContainerName -Permission "racw" -ExpiryTime (Get-Date).AddHours($ExpiryHours) -Context $storageContext
    
    return $sasToken
}

# Function to upload backup file to Azure
function Upload-BackupToAzure {
    param(
        [string]$LocalBackupFile,
        [string]$BlobName,
        [string]$StorageAccountName,
        [string]$SasToken,
        [string]$ContainerName
    )
    
    $blobUrl = "https://$StorageAccountName.blob.core.windows.net/$ContainerName/$BlobName"
    $uploadUrl = "$blobUrl$SasToken"
    
    try {
        $context = New-AzureStorageContext -StorageAccountName $StorageAccountName -SasToken $SasToken
        Set-AzureStorageBlobContent -File $LocalBackupFile -Blob $BlobName -Container $ContainerName -Context $context -Force
        Write-Log "Successfully uploaded $BlobName to Azure Blob Storage"
        return $blobUrl
    }
    catch {
        Write-Log "Failed to upload $BlobName`: $($_.Exception.Message)" 'ERROR'
        return $null
    }
}

# Function to backup database and upload to Azure
function Backup-DatabaseToAzure {
    param(
        [string]$Server,
        [string]$Database,
        [string]$BackupType,
        [string]$LocalBackupPath,
        [string]$BlobName,
        [string]$StorageAccount,
        [string]$SasToken,
        [string]$Container,
        [bool]$Compress
    )
    
    $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    $backupFile = Join-Path $LocalBackupPath "$Database`_$BackupType`_$timestamp.bak"
    
    # Create backup directory
    if (-not (Test-Path $LocalBackupPath)) {
        New-Item -ItemType Directory -Path $LocalBackupPath -Force | Out-Null
    }
    
    # Build backup T-SQL
    $backupQuery = "BACKUP $BackupType [$Database] TO DISK = '$backupFile'"
    
    if ($Compress) {
        $backupQuery += " WITH COMPRESSION"
    }
    
    $backupQuery += ", STATS = 10"
    
    if ($BackupType -eq 'DIFFERENTIAL') {
        $backupQuery += ", DIFFERENTIAL"
    }
    
    try {
        # Execute backup
        Write-Log "Starting backup of $Database to $backupFile"
        Invoke-Sqlcmd -ServerInstance $Server -Database master -Query $backupQuery -ErrorAction Stop
        
        # Upload to Azure
        $blobUrl = Upload-BackupToAzure -LocalBackupFile $backupFile -BlobName $BlobName -StorageAccountName $StorageAccount -SasToken $SasToken -ContainerName $Container
        
        # Clean up local file
        Remove-Item $backupFile -Force
        
        if ($blobUrl) {
            Write-Log "Backup and upload completed for $Database"
            return $blobUrl
        }
        else {
            return $null
        }
    }
    catch {
        Write-Log "Backup failed for $Database`: $($_.Exception.Message)" 'ERROR'
        return $null
    }
}

# Main execution
try {
    Write-Log "Starting Azure SQL Server backup process"
    
    # Generate SAS token
    $sasToken = New-SAStoken -AccountName $StorageAccountName -AccountKey $StorageAccountKey -ContainerName $ContainerName
    
    # Local backup directory
    $localBackupPath = "C:\Temp\Backups\$(Get-Date -Format 'yyyy-MM-dd')"
    
    # Get databases to backup
    $databases = Invoke-Sqlcmd -ServerInstance $ServerName -Database master -Query @"
    SELECT name FROM sys.databases
    WHERE state = 0 -- Online
    AND database_id > 4 -- Exclude system databases
    AND name NOT IN ($(($ExcludeDatabases | ForEach-Object { "'$_'" }) -join ','))
    ORDER BY name
"@
    
    $successfulBackups = @()
    $failedBackups = @()
    
    foreach ($db in $databases) {
        $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
        $blobName = "$($db.name)_$BackupType`_$timestamp.bak"
        
        $blobUrl = Backup-DatabaseToAzure -Server $ServerName -Database $db.name -BackupType $BackupType -LocalBackupPath $localBackupPath -BlobName $blobName -StorageAccount $StorageAccountName -SasToken $sasToken -Container $ContainerName -Compress $Compress
        
        if ($blobUrl) {
            $successfulBackups += [PSCustomObject]@{
                Database = $db.name
                BlobUrl = $blobUrl
                Timestamp = Get-Date
            }
        }
        else {
            $failedBackups += $db.name
        }
    }
    
    # Summary
    Write-Log "=== Azure Backup Summary ==="
    Write-Log "Successful uploads: $($successfulBackups.Count)"
    Write-Log "Failed uploads: $($failedBackups.Count)"
    
    if ($successfulBackups.Count -gt 0) {
        Write-Log "Successful Azure Uploads:"
        foreach ($backup in $successfulBackups) {
            Write-Log "  - $($backup.Database): $($backup.BlobUrl)"
        }
    }
}
catch {
    Write-Log "Azure backup process failed: $($_.Exception.Message)" 'ERROR'
}
```

### Amazon S3 Backup Integration

#### Configuring S3 for SQL Server Backups
```sql
-- Create credential for Amazon S3
CREATE CREDENTIAL AWSCredentials
WITH IDENTITY = 'S3',
SECRET = 'AccessKeyID:SecretAccessKey';

-- Backup to Amazon S3
BACKUP DATABASE SalesDB
TO URL = 's3://mybucket/sqlbackups/SalesDB_Full_20250101.bak'
WITH 
    FORMAT,
    COMPRESSION,
    CHECKSUM,
    CREDENTIAL = 'AWSCredentials',
    STATS = 10;
```

#### Comprehensive Offsite Backup Script
```powershell
# Comprehensive offsite backup script with multiple cloud providers
# Save as: Offsite-BackupManager.ps1

enum BackupDestination {
    Local = 1
    Azure = 2
    AWS = 3
    Google = 4
}

class BackupJob {
    [string]$Database
    [string]$Server
    [BackupDestination]$Destination
    [string]$BackupPath
    [string]$BackupFile
    [datetime]$StartTime
    [datetime]$EndTime
    [bool]$Success
    [string]$ErrorMessage
    [long]$FileSize
}

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
    Add-Content -Path "C:\Logs\OffsiteBackup_$(Get-Date -Format 'yyyyMMdd').log" -Value $logMessage
}

function Start-SQLBackup {
    param(
        [string]$Server,
        [string]$Database,
        [string]$BackupType,
        [string]$BackupPath,
        [bool]$Compress = $true
    )
    
    $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    $backupFile = Join-Path $BackupPath "$Database`_$BackupType`_$timestamp.bak"
    
    if (-not (Test-Path $BackupPath)) {
        New-Item -ItemType Directory -Path $BackupPath -Force | Out-Null
    }
    
    $backupQuery = "BACKUP $BackupType [$Database] TO DISK = '$backupFile'"
    if ($Compress) {
        $backupQuery += " WITH COMPRESSION"
    }
    $backupQuery += ", STATS = 10"
    
    if ($BackupType -eq 'DIFFERENTIAL') {
        $backupQuery += ", DIFFERENTIAL"
    }
    
    try {
        $result = Invoke-Sqlcmd -ServerInstance $Server -Database master -Query $backupQuery -ErrorAction Stop
        $fileInfo = Get-Item $backupFile
        return @{
            Success = $true
            FilePath = $backupFile
            FileSize = $fileInfo.Length
            StartTime = $script:startTime
            EndTime = Get-Date
        }
    }
    catch {
        return @{
            Success = $false
            ErrorMessage = $_.Exception.Message
            StartTime = $script:startTime
            EndTime = Get-Date
        }
    }
}

function Send-ToAzure {
    param(
        [string]$FilePath,
        [string]$BlobName,
        [string]$StorageAccount,
        [string]$Container,
        [string]$SasToken
    )
    
    try {
        $context = New-AzureStorageContext -StorageAccountName $StorageAccount -SasToken $SasToken
        Set-AzureStorageBlobContent -File $FilePath -Blob $BlobName -Container $Container -Context $context -Force
        return $true
    }
    catch {
        Write-Log "Azure upload failed: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

function Send-ToAWS {
    param(
        [string]$FilePath,
        [string]$S3Key,
        [string]$BucketName,
        [string]$AccessKey,
        [string]$SecretKey,
        [string]$Region = 'us-east-1'
    )
    
    try {
        $credentials = New-AWSCredentials -AccessKey $AccessKey -SecretKey $SecretKey
        Set-S3Object -BucketName $BucketName -Key $S3Key -File $FilePath -Region $Region -Credential $credentials
        return $true
    }
    catch {
        Write-Log "AWS upload failed: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

# Main backup orchestrator
function Start-OrchestratedBackup {
    param(
        [string]$Server = 'localhost',
        [BackupDestination[]]$Destinations = @([BackupDestination]::Local, [BackupDestination]::Azure),
        [string]$BackupType = 'Full',
        [string]$LocalBackupPath = 'C:\SQLBackups\Daily',
        [hashtable]$CloudCredentials = @{}
    )
    
    Write-Log "Starting orchestrated backup process"
    
    # Get databases to backup
    $databases = Invoke-Sqlcmd -ServerInstance $Server -Database master -Query @"
    SELECT name FROM sys.databases
    WHERE state = 0 -- Online
    AND database_id > 4 -- Exclude system databases
    ORDER BY name
"@
    
    $backupJobs = @()
    
    foreach ($db in $databases) {
        Write-Log "Processing database: $($db.name)"
        
        # Perform local backup
        $script:startTime = Get-Date
        $localBackup = Start-SQLBackup -Server $Server -Database $db.name -BackupType $BackupType -BackupPath $LocalBackupPath
        
        $job = [BackupJob]::new()
        $job.Database = $db.name
        $job.Server = $Server
        $job.Destination = [BackupDestination]::Local
        $job.BackupPath = $LocalBackupPath
        $job.BackupFile = $localBackup.FilePath
        $job.StartTime = $localBackup.StartTime
        $job.EndTime = $localBackup.EndTime
        $job.Success = $localBackup.Success
        $job.ErrorMessage = $localBackup.ErrorMessage
        $job.FileSize = $localBackup.FileSize
        
        if ($localBackup.Success) {
            # Upload to cloud destinations
            foreach ($destination in $Destinations) {
                if ($destination -eq [BackupDestination]::Local) { continue }
                
                switch ($destination) {
                    ([BackupDestination]::Azure) {
                        if ($CloudCredentials.ContainsKey('Azure')) {
                            $azureCreds = $CloudCredentials['Azure']
                            $blobName = "$($db.name)/$BackupType/$(Split-Path $localBackup.FilePath -Leaf)"
                            $uploadSuccess = Send-ToAzure -FilePath $localBackup.FilePath -BlobName $blobName -StorageAccount $azureCreds.AccountName -Container $azureCreds.Container -SasToken $azureCreds.SasToken
                            
                            if ($uploadSuccess) {
                                Write-Log "Azure upload successful for $($db.name)"
                            }
                            else {
                                Write-Log "Azure upload failed for $($db.name)" 'WARNING'
                            }
                        }
                    }
                    
                    ([BackupDestination]::AWS) {
                        if ($CloudCredentials.ContainsKey('AWS')) {
                            $awsCreds = $CloudCredentials['AWS']
                            $s3Key = "$($db.name)/$BackupType/$(Split-Path $localBackup.FilePath -Leaf)"
                            $uploadSuccess = Send-ToAWS -FilePath $localBackup.FilePath -S3Key $s3Key -BucketName $awsCreds.BucketName -AccessKey $awsCreds.AccessKey -SecretKey $awsCreds.SecretKey
                            
                            if ($uploadSuccess) {
                                Write-Log "AWS upload successful for $($db.name)"
                            }
                            else {
                                Write-Log "AWS upload failed for $($db.name)" 'WARNING'
                            }
                        }
                    }
                }
            }
            
            # Clean up local file after successful upload
            if (Test-Path $localBackup.FilePath) {
                Remove-Item $localBackup.FilePath -Force
            }
        }
        
        $backupJobs += $job
    }
    
    # Generate backup report
    $successfulJobs = $backupJobs | Where-Object { $_.Success }
    $failedJobs = $backupJobs | Where-Object { -not $_.Success }
    
    Write-Log "=== Backup Process Summary ==="
    Write-Log "Total databases processed: $($backupJobs.Count)"
    Write-Log "Successful backups: $($successfulJobs.Count)"
    Write-Log "Failed backups: $($failedJobs.Count)"
    
    if ($failedJobs.Count -gt 0) {
        Write-Log "Failed databases:"
        foreach ($job in $failedJobs) {
            Write-Log "  - $($job.Database): $($job.ErrorMessage)" 'ERROR'
        }
    }
    
    return $backupJobs
}

# Example usage
$cloudCredentials = @{
    Azure = @{
        AccountName = 'mybackupstorage'
        Container = 'sqlserverbackups'
        SasToken = 'sv=2020-08-04&ss=bfqt&srt=sco&sp=rwdlacupx&se=2025-12-31T23:59:59Z&st=2021-01-01T00:00:00Z&spr=https,http&sig=XXXXX'
    }
    AWS = @{
        BucketName = 'my-sql-backups'
        AccessKey = 'AKIAIOSFODNN7EXAMPLE'
        SecretKey = 'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY'
        Region = 'us-east-1'
    }
}

$backupResults = Start-OrchestratedBackup -Server 'localhost' -Destinations @([BackupDestination]::Local, [BackupDestination]::Azure, [BackupDestination]::AWS) -CloudCredentials $cloudCredentials
```

## Thursday - Disaster Recovery Planning and Testing

### Disaster Recovery Planning Framework

#### Risk Assessment and Business Impact Analysis

**Step 1: Identify Critical Systems and Data**
```sql
-- Identify critical databases and their business impact
SELECT 
    DB.name as DatabaseName,
    CASE 
        WHEN DB.database_id <= 4 THEN 'System Database'
        WHEN DB.name LIKE '%PROD%' THEN 'Production Critical'
        WHEN DB.name LIKE '%TEST%' THEN 'Test Environment'
        ELSE 'Development/Support'
    END as Criticality,
    SUM(mf.size * 8.0 / 1024) as SizeMB,
    DB.recovery_model_desc,
    MAX(bs.backup_finish_date) as LastBackupDate,
    SUM(CASE WHEN bs.type = 'D' THEN 1 ELSE 0 END) as FullBackups,
    SUM(CASE WHEN bs.type = 'I' THEN 1 ELSE 0 END) as DiffBackups,
    SUM(CASE WHEN bs.type = 'L' THEN 1 ELSE 0 END) as LogBackups
FROM sys.databases DB
LEFT JOIN sys.master_files mf ON DB.database_id = mf.database_id
LEFT JOIN msdb.dbo.backupset bs ON DB.name = bs.database_name
WHERE DB.state = 0 -- Online
GROUP BY DB.name, DB.database_id, DB.recovery_model_desc, DB.is_read_only
ORDER BY DatabaseName;
```

**Step 2: Calculate Recovery Time Objectives (RTO) and Recovery Point Objectives (RPO)**
```sql
-- Create table to track RTO/RPO requirements
CREATE TABLE DatabaseRTO_RPO (
    DatabaseName NVARCHAR(128) PRIMARY KEY,
    BusinessFunction NVARCHAR(200),
    MaxDowntimeMinutes INT,
    MaxDataLossMinutes INT,
    BackupFrequency VARCHAR(50),
    DisasterRecoveryStrategy NVARCHAR(200),
    TestingFrequency VARCHAR(50),
    LastTestDate DATETIME,
    NextTestDate DATETIME,
    CreatedDate DATETIME DEFAULT GETDATE(),
    ModifiedDate DATETIME DEFAULT GETDATE()
);

-- Insert sample RTO/RPO data
INSERT INTO DatabaseRTO_RPO (DatabaseName, BusinessFunction, MaxDowntimeMinutes, MaxDataLossMinutes, BackupFrequency, DisasterRecoveryStrategy)
VALUES 
('SalesDB', 'Order Processing', 15, 5, 'Log every 15 min, Full weekly', 'Always On Availability Group'),
('FinanceDB', 'Financial Reporting', 30, 10, 'Full daily, Log every 30 min', 'Log Shipping'),
('HRDB', 'Human Resources', 60, 30, 'Full weekly, Log daily', 'Backup/Restore'),
('InventoryDB', 'Inventory Management', 30, 15, 'Full weekly, Log every 30 min', 'Database Mirroring');
```

#### Disaster Recovery Testing Procedures

**Step 1: Create Disaster Recovery Testing Procedure**
```sql
-- Create stored procedure for DR testing
CREATE OR ALTER PROCEDURE sp_TestDisasterRecovery
    @TestDatabase NVARCHAR(128),
    @TestBackupPath NVARCHAR(500),
    @ExpectedRecoveryTime INT OUTPUT,
    @ActualRecoveryTime INT OUTPUT,
    @TestResult NVARCHAR(MAX) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @StartTime DATETIME;
    DECLARE @EndTime DATETIME;
    DECLARE @BackupFile NVARCHAR(500);
    DECLARE @RecoverySQL NVARCHAR(MAX);
    DECLARE @ErrorCount INT = 0;
    
    SET @StartTime = GETDATE();
    SET @TestResult = 'DR Test started for ' + @TestDatabase + ' at ' + CONVERT(VARCHAR, @StartTime, 120);
    
    BEGIN TRY
        -- Step 1: Verify backup file exists and is readable
        SET @BackupFile = @TestBackupPath + @TestDatabase + '_Full.bak';
        
        IF NOT EXISTS (SELECT 1 FROM sys.dm_os_hadr_database_replica_states WHERE database_name = @TestDatabase)
        BEGIN
            SET @TestResult = @TestResult + CHAR(13) + 'Checking backup file: ' + @BackupFile;
            
            -- Test RESTORE VERIFYONLY
            SET @RecoverySQL = 'RESTORE VERIFYONLY FROM DISK = ''' + @BackupFile + '''';
            EXEC(@RecoverySQL);
            
            IF @@ERROR = 0
                SET @TestResult = @TestResult + CHAR(13) + 'Backup file verification: SUCCESS';
            ELSE
                SET @TestResult = @TestResult + CHAR(13) + 'Backup file verification: FAILED';
        END
        
        -- Step 2: Test restore to staging database
        DECLARE @StagingDB NVARCHAR(128) = @TestDatabase + '_DR_Test_' + CONVERT(VARCHAR, GETDATE(), 112);
        
        -- Drop staging database if exists
        IF DB_ID(@StagingDB) IS NOT NULL
        BEGIN
            ALTER DATABASE @StagingDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
            DROP DATABASE @StagingDB;
        END
        
        -- Restore to staging database
        SET @RecoverySQL = 'RESTORE DATABASE [' + @StagingDB + '] FROM DISK = ''' + @BackupFile + ''' WITH REPLACE, MOVE ''' + 
                          @TestDatabase + '''_Data TO ''' + @TestBackupPath + @StagingDB + '.mdf'', MOVE ''' + 
                          @TestDatabase + '''_Log TO ''' + @TestBackupPath + @StagingDB + '.ldf''';
        
        EXEC(@RecoverySQL);
        
        IF @@ERROR = 0
        BEGIN
            SET @TestResult = @TestResult + CHAR(13) + 'Database restore: SUCCESS';
            
            -- Step 3: Verify data integrity
            DECLARE @TableCount INT, @IndexCount INT, @RowCount BIGINT;
            
            SELECT @TableCount = COUNT(*) FROM @StagingDB.INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE';
            SELECT @IndexCount = COUNT(*) FROM @StagingDB.sys.indexes WHERE index_id > 0;
            
            SET @TestResult = @TestResult + CHAR(13) + 'Tables restored: ' + CAST(@TableCount AS VARCHAR);
            SET @TestResult = @TestResult + CHAR(13) + 'Indexes restored: ' + CAST(@IndexCount AS VARCHAR);
            
            -- Step 4: Test application connectivity
            -- This would be replaced with actual application testing logic
            DECLARE @ConnectionTest BIT = 1; -- Assume success
            
            IF @ConnectionTest = 1
                SET @TestResult = @TestResult + CHAR(13) + 'Application connectivity test: SUCCESS';
            ELSE
                SET @TestResult = @TestResult + CHAR(13) + 'Application connectivity test: FAILED';
            
            -- Clean up staging database
            ALTER DATABASE @StagingDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
            DROP DATABASE @StagingDB;
        END
        ELSE
        BEGIN
            SET @TestResult = @TestResult + CHAR(13) + 'Database restore: FAILED';
            SET @ErrorCount = @ErrorCount + 1;
        END
        
        SET @EndTime = GETDATE();
        SET @ActualRecoveryTime = DATEDIFF(SECOND, @StartTime, @EndTime);
        
        -- Calculate expected recovery time (backup size / restore speed estimate)
        -- This is a simplified calculation - in practice you'd use historical data
        DECLARE @BackupSizeMB DECIMAL(18,2);
        SELECT @BackupSizeMB = CAST(mf.size * 8.0 / 1024 AS DECIMAL(18,2))
        FROM sys.master_files mf
        JOIN sys.databases d ON mf.database_id = d.database_id
        WHERE d.name = @TestDatabase AND mf.type = 0;
        
        SET @ExpectedRecoveryTime = @BackupSizeMB * 2; -- Assume 2 seconds per MB restore time
        
        IF @ErrorCount = 0
            SET @TestResult = @TestResult + CHAR(13) + 'DR Test completed: SUCCESS';
        ELSE
            SET @TestResult = @TestResult + CHAR(13) + 'DR Test completed: FAILED with ' + CAST(@ErrorCount AS VARCHAR) + ' errors';
            
    END TRY
    BEGIN CATCH
        SET @EndTime = GETDATE();
        SET @ActualRecoveryTime = DATEDIFF(SECOND, @StartTime, @EndTime);
        SET @TestResult = @TestResult + CHAR(13) + 'DR Test failed: ' + ERROR_MESSAGE();
        SET @ErrorCount = @ErrorCount + 1;
    END CATCH
    
    -- Log test results
    INSERT INTO DR_Test_History (DatabaseName, TestDate, StartTime, EndTime, ExpectedRecoveryTime, ActualRecoveryTime, Result, ErrorCount)
    VALUES (@TestDatabase, CAST(GETDATE() AS DATE), @StartTime, @EndTime, @ExpectedRecoveryTime, @ActualRecoveryTime, @TestResult, @ErrorCount);
    
    PRINT @TestResult;
END;
GO

-- Create table to track DR test history
CREATE TABLE DR_Test_History (
    TestID INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(128),
    TestDate DATE,
    StartTime DATETIME,
    EndTime DATETIME,
    ExpectedRecoveryTime INT,
    ActualRecoveryTime INT,
    Result NVARCHAR(MAX),
    ErrorCount INT
);
```

**Step 2: Comprehensive DR Testing Script**
```powershell
# Comprehensive Disaster Recovery Testing Script
# Save as: Test-DisasterRecovery.ps1

param(
    [Parameter(Mandatory=$true)]
    [string]$PrimaryServer,
    
    [Parameter(Mandatory=$false)]
    [string]$DrTestServer = 'localhost',
    
    [Parameter(Mandatory=$true)]
    [string]$TestBackupPath,
    
    [Parameter(Mandatory=$false)]
    [string]$ReportPath = 'C:\Reports\DRTest',
    
    [Parameter(Mandatory=$false)]
    [switch]$FullTest,
    
    [Parameter(Mandatory=$false)]
    [string[]]$TestDatabases
)

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
    Add-Content -Path (Join-Path $ReportPath "DRTest_$(Get-Date -Format 'yyyyMMdd').log") -Value $logMessage
}

function Start-DRTest {
    param(
        [string]$SourceServer,
        [string]$TestServer,
        [string]$Database,
        [string]$BackupPath
    )
    
    $testStartTime = Get-Date
    Write-Log "Starting DR test for $Database on $TestServer"
    
    # Step 1: Create fresh backup from primary server
    $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    $primaryBackupFile = Join-Path $BackupPath "$Database`_PrimaryBackup`_$timestamp.bak"
    
    try {
        # Create backup on primary server
        $backupQuery = @"
BACKUP DATABASE [$Database] 
TO DISK = '$primaryBackupFile'
WITH 
    FORMAT,
    COMPRESSION,
    CHECKSUM,
    STATS = 10;
"@
        
        Invoke-Sqlcmd -ServerInstance $SourceServer -Database master -Query $backupQuery -ErrorAction Stop
        Write-Log "Created primary backup: $primaryBackupFile"
        
        # Step 2: Transfer backup file to test server (simulate offsite storage)
        # In real scenario, this would use actual file transfer methods
        $testBackupFile = Join-Path $BackupPath "$Database`_TestRestore`_$timestamp.bak"
        Copy-Item $primaryBackupFile $testBackupFile -Force
        
        # Step 3: Restore backup on test server
        $restoreStartTime = Get-Date
        $stagingDb = "$Database`_DR_Test_$(Get-Date -Format 'yyyyMMdd_HHmmss')"
        
        $restoreQuery = @"
RESTORE DATABASE [$stagingDb] 
FROM DISK = '$testBackupFile'
WITH 
    REPLACE,
    MOVE '$($Database)_Data' TO 'C:\SQLData\$stagingDb.mdf',
    MOVE '$($Database)_Log' TO 'C:\SQLLogs\$stagingDb.ldf',
    STATS = 10,
    NORECOVERY;
"@
        
        Invoke-Sqlcmd -ServerInstance $TestServer -Database master -Query $restoreQuery -ErrorAction Stop
        $restoreEndTime = Get-Date
        $restoreDuration = ($restoreEndTime - $restoreStartTime).TotalSeconds
        
        Write-Log "Restore completed in $restoreDuration seconds"
        
        # Step 4: Verify database integrity
        $integrityQuery = "DBCC CHECKDB('$stagingDb') WITH NO_INFOMSGS"
        $integrityResult = Invoke-Sqlcmd -ServerInstance $TestServer -Database master -Query $integrityQuery -ErrorAction Stop
        Write-Log "Database integrity check: PASSED"
        
        # Step 5: Test query execution
        $testQuery = "SELECT COUNT(*) as RecordCount FROM $stagingDb.INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE'"
        $tableCount = Invoke-Sqlcmd -ServerInstance $TestServer -Database $stagingDb -Query $testQuery -ErrorAction Stop
        Write-Log "Tables found: $($tableCount.RecordCount)"
        
        # Step 6: Performance testing
        $perfTestQuery = @"
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
SELECT TOP 100 * FROM $stagingDb.dbo.INFORMATION_SCHEMA.TABLES;
SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
"@
        
        $perfStartTime = Get-Date
        Invoke-Sqlcmd -ServerInstance $TestServer -Database $stagingDb -Query $perfTestQuery -ErrorAction Stop
        $perfEndTime = Get-Date
        $perfDuration = ($perfEndTime - $perfStartTime).TotalSeconds
        
        Write-Log "Performance test completed in $perfDuration seconds"
        
        # Step 7: Cleanup
        $cleanupQuery = "ALTER DATABASE [$stagingDb] SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE [$stagingDb];"
        Invoke-Sqlcmd -ServerInstance $TestServer -Database master -Query $cleanupQuery -ErrorAction Stop
        
        # Cleanup backup files
        Remove-Item $primaryBackupFile -Force
        Remove-Item $testBackupFile -Force
        
        $testEndTime = Get-Date
        $totalDuration = ($testEndTime - $testStartTime).TotalSeconds
        
        return @{
            Success = $true
            Database = $Database
            RestoreTime = $restoreDuration
            TotalTime = $totalDuration
            TablesCount = $tableCount.RecordCount
            PerformanceTime = $perfDuration
        }
        
    }
    catch {
        $testEndTime = Get-Date
        $totalDuration = ($testEndTime - $testStartTime).TotalSeconds
        
        Write-Log "DR test failed for $Database`: $($_.Exception.Message)" 'ERROR'
        
        return @{
            Success = $false
            Database = $Database
            Error = $_.Exception.Message
            Duration = $totalDuration
        }
    }
}

# Main DR testing process
try {
    # Create report directory
    if (-not (Test-Path $ReportPath)) {
        New-Item -ItemType Directory -Path $ReportPath -Force | Out-Null
    }
    
    Write-Log "Starting comprehensive DR testing"
    Write-Log "Primary Server: $PrimaryServer"
    Write-Log "Test Server: $DrTestServer"
    Write-Log "Backup Path: $TestBackupPath"
    
    # Get databases to test
    if ($TestDatabases) {
        $databases = $TestDatabases
    }
    else {
        $databases = Invoke-Sqlcmd -ServerInstance $PrimaryServer -Database master -Query @"
        SELECT name FROM sys.databases
        WHERE state = 0 -- Online
        AND database_id > 4 -- Exclude system databases
        AND name NOT LIKE '%Temp%'
        ORDER BY name
"@ | Select-Object -ExpandProperty name
    }
    
    $testResults = @()
    $successfulTests = 0
    $failedTests = 0
    
    foreach ($db in $databases) {
        $result = Start-DRTest -SourceServer $PrimaryServer -TestServer $DrTestServer -Database $db -BackupPath $TestBackupPath
        
        $testResults += $result
        
        if ($result.Success) {
            $successfulTests++
        }
        else {
            $failedTests++
        }
    }
    
    # Generate comprehensive report
    $reportFile = Join-Path $ReportPath "DR_Test_Report_$(Get-Date -Format 'yyyyMMdd_HHmmss').html"
    
    $htmlReport = @"
<!DOCTYPE html>
<html>
<head>
    <title>Disaster Recovery Test Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background-color: #f0f0f0; padding: 10px; border-bottom: 2px solid #ccc; }
        .summary { margin: 20px 0; padding: 15px; border: 1px solid #ddd; }
        .success { color: green; font-weight: bold; }
        .failed { color: red; font-weight: bold; }
        table { border-collapse: collapse; width: 100%; margin: 20px 0; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Disaster Recovery Test Report</h1>
        <p>Test Date: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')</p>
        <p>Primary Server: $PrimaryServer</p>
        <p>Test Server: $DrTestServer</p>
    </div>
    
    <div class="summary">
        <h2>Executive Summary</h2>
        <p>Total Databases Tested: $($testResults.Count)</p>
        <p class="success">Successful Restores: $successfulTests</p>
        <p class="failed">Failed Restores: $failedTests</p>
        <p>Success Rate: $([math]::Round(($successfulTests / $testResults.Count) * 100, 2))%</p>
    </div>
    
    <table>
        <tr>
            <th>Database</th>
            <th>Status</th>
            <th>Restore Time (sec)</th>
            <th>Total Test Time (sec)</th>
            <th>Tables Count</th>
            <th>Performance Time (sec)</th>
            <th>Error</th>
        </tr>
"@
    
    foreach ($result in $testResults) {
        $statusClass = if ($result.Success) { 'success' } else { 'failed' }
        $status = if ($result.Success) { 'SUCCESS' } else { 'FAILED' }
        
        $htmlReport += @"
        <tr>
            <td>$($result.Database)</td>
            <td class="$statusClass">$status</td>
            <td>$([math]::Round($result.RestoreTime, 2))</td>
            <td>$([math]::Round($result.TotalTime, 2))</td>
            <td>$($result.TablesCount)</td>
            <td>$([math]::Round($result.PerformanceTime, 2))</td>
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
    
    Write-Log "DR testing completed. Report saved to: $reportFile"
    
    # Send summary email (if configured)
    # This would be implemented based on your email system
    
}
catch {
    Write-Log "DR testing process failed: $($_.Exception.Message)" 'ERROR'
}
```

## Friday - Backup Monitoring and Troubleshooting

### Backup Monitoring Dashboard

#### Real-Time Backup Monitoring Query
```sql
-- Query to monitor current backup operations
SELECT 
    session_id,
    command,
    percent_complete,
    estimated_completion_time / 1000 / 60 as estimated_minutes_remaining,
    estimated_completion_time / 1000 / 60 / 60 as estimated_hours_remaining,
    start_time,
    status,
    database_id,
    DB_NAME(database_id) as database_name,
    blocking_session_id,
    wait_type,
    wait_time,
    cpu_time,
    reads,
    writes,
    logical_reads
FROM sys.dm_exec_requests
WHERE command = 'BACKUP DATABASE' 
   OR command = 'BACKUP LOG'
ORDER BY start_time DESC;

-- Query to check backup history and performance
SELECT 
    bs.database_name,
    bs.type as backup_type,
    bs.backup_start_date,
    bs.backup_finish_date,
    DATEDIFF(SECOND, bs.backup_start_date, bs.backup_finish_date) as duration_seconds,
    bs.backup_size / 1024 / 1024 as size_mb,
    CASE 
        WHEN bs.backup_size > 0 THEN CAST(bs.backup_size / 1024.0 / 1024 / DATEDIFF(SECOND, bs.backup_start_date, bs.backup_finish_date) AS DECIMAL(10,2))
        ELSE 0 
    END as throughput_mbps,
    bs.is_compressed,
    bs.is_encrypted,
    bmf.physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.backup_start_date >= DATEADD(DAY, -7, GETDATE())
ORDER BY bs.backup_start_date DESC;
```

#### Backup Performance Analysis
```sql
-- Analyze backup performance trends
WITH BackupPerformance AS (
    SELECT 
        bs.database_name,
        bs.backup_start_date,
        bs.type,
        bs.backup_size / 1024.0 / 1024 as size_mb,
        DATEDIFF(SECOND, bs.backup_start_date, bs.backup_finish_date) as duration_seconds,
        bs.backup_size / 1024.0 / 1024 / NULLIF(DATEDIFF(SECOND, bs.backup_start_date, bs.backup_finish_date), 0) as throughput_mbps,
        ROW_NUMBER() OVER (PARTITION BY bs.database_name, bs.type ORDER BY bs.backup_start_date DESC) as rn
    FROM msdb.dbo.backupset bs
    WHERE bs.backup_start_date >= DATEADD(DAY, -30, GETDATE())
)
SELECT 
    database_name,
    type,
    COUNT(*) as backup_count,
    AVG(size_mb) as avg_size_mb,
    MIN(size_mb) as min_size_mb,
    MAX(size_mb) as max_size_mb,
    AVG(duration_seconds) as avg_duration_seconds,
    MIN(duration_seconds) as min_duration_seconds,
    MAX(duration_seconds) as max_duration_seconds,
    AVG(throughput_mbps) as avg_throughput_mbps,
    MIN(throughput_mbps) as min_throughput_mbps,
    MAX(throughput_mbps) as max_throughput_mbps,
    -- Identify performance degradation
    CASE 
        WHEN AVG(throughput_mbps) < LAG(AVG(throughput_mbps)) OVER (PARTITION BY database_name, type ORDER BY backup_start_date) * 0.8
        THEN 'PERFORMANCE_DEGRADATION'
        ELSE 'NORMAL'
    END as performance_status
FROM BackupPerformance
GROUP BY database_name, type
ORDER BY database_name, type;
```

### Backup Troubleshooting Procedures

#### Common Backup Failure Scenarios and Solutions

**Scenario 1: Disk Space Issues**
```sql
-- Query to identify disk space issues
SELECT 
    vs.volume_mount_point,
    vs.file_system_type,
    CAST(vs.total_bytes / 1024.0 / 1024 / 1024 AS DECIMAL(18,2)) as total_gb,
    CAST(vs.available_bytes / 1024.0 / 1024 / 1024 AS DECIMAL(18,2)) as available_gb,
    CAST((vs.total_bytes - vs.available_bytes) / 1024.0 / 1024 / 1024 AS DECIMAL(18,2)) as used_gb,
    CAST((vs.available_bytes * 100.0 / vs.total_bytes) AS DECIMAL(5,2)) as free_space_percent
FROM sys.master_files mf
CROSS APPLY sys.dm_os_volume_stats(mf.database_id, mf.file_id) vs
ORDER BY vs.volume_mount_point;

-- Check backup file sizes and growth patterns
SELECT 
    bs.database_name,
    bs.type,
    bs.backup_start_date,
    bs.backup_size / 1024.0 / 1024 as backup_size_mb,
    DATEDIFF(SECOND, bs.backup_start_date, bs.backup_finish_date) as backup_duration_seconds,
    bmf.physical_device_name,
    -- Identify abnormal backup size changes
    LAG(bs.backup_size / 1024.0 / 1024) OVER (PARTITION BY bs.database_name, bs.type ORDER BY bs.backup_start_date) as prev_backup_size_mb,
    CASE 
        WHEN bs.backup_size > LAG(bs.backup_size) OVER (PARTITION BY bs.database_name, bs.type ORDER BY bs.backup_start_date) * 1.5
        THEN 'SIZE_SPIKE'
        ELSE 'NORMAL'
    END as size_status
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.backup_start_date >= DATEADD(DAY, -7, GETDATE())
ORDER BY bs.backup_start_date DESC;
```

**Scenario 2: Permission and Access Issues**
```sql
-- Check backup path permissions using xp_cmdshell (requires elevated permissions)
-- Enable xp_cmdshell if needed (use with caution in production)
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;

-- Test backup path accessibility
DECLARE @backupPath NVARCHAR(500) = 'C:\SQLBackups\';
DECLARE @cmd NVARCHAR(1000);

-- List contents of backup directory
SET @cmd = 'dir "' + @backupPath + '" /b';
EXEC xp_cmdshell @cmd;

-- Check directory permissions
SET @cmd = 'cacls "' + @backupPath + '"';
EXEC xp_cmdshell @cmd;
```

**Scenario 3: Backup Corruption Issues**
```sql
-- Identify corrupted backups
SELECT 
    bs.database_name,
    bs.backup_start_date,
    bs.type,
    bmf.physical_device_name,
    bs.is_compressed,
    bs.is_encrypted,
    -- Check for backup verification status
    CASE 
        WHEN r.restore_type IS NOT NULL THEN 'VERIFIED'
        ELSE 'NOT_VERIFIED'
    END as verification_status
FROM msdb.dbo.backupset bs
LEFT JOIN msdb.dbo.restorehistory r ON bs.backup_set_id = r.backup_set_id
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.backup_start_date >= DATEADD(DAY, -30, GETDATE())
ORDER BY bs.backup_start_date DESC;

-- Verify backup integrity (run manually for suspected corrupted backups)
-- RESTORE VERIFYONLY FROM DISK = 'C:\Backups\YourBackupFile.bak';
```

#### Automated Backup Health Monitoring Script
```powershell
# Automated Backup Health Monitoring
# Save as: Monitor-BackupHealth.ps1

param(
    [Parameter(Mandatory=$true)]
    [string]$SqlServer,
    
    [Parameter(Mandatory=$false)]
    [int]$AlertThresholdDays = 1,
    
    [Parameter(Mandatory=$false)]
    [string]$AlertEmail = 'dba@company.com',
    
    [Parameter(Mandatory=$false)]
    [string]$SmtpServer = 'smtp.company.com'
)

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
}

function Get-BackupStatus {
    param([string]$Server)
    
    $query = @"
    SELECT 
        d.name as DatabaseName,
        d.database_id,
        d.state_desc,
        d.recovery_model_desc,
        -- Last full backup
        MAX(CASE WHEN bs.type = 'D' THEN bs.backup_finish_date ELSE NULL END) as LastFullBackup,
        -- Last differential backup
        MAX(CASE WHEN bs.type = 'I' THEN bs.backup_finish_date ELSE NULL END) as LastDiffBackup,
        -- Last log backup
        MAX(CASE WHEN bs.type = 'L' THEN bs.backup_finish_date ELSE NULL END) as LastLogBackup,
        -- Backup file location
        MAX(CASE WHEN bs.type = 'D' THEN bmf.physical_device_name ELSE NULL END) as LastFullBackupPath,
        -- Count backups in last 7 days
        SUM(CASE WHEN bs.backup_start_date >= DATEADD(DAY, -7, GETDATE()) THEN 1 ELSE 0 END) as BackupsLast7Days
    FROM sys.databases d
    LEFT JOIN msdb.dbo.backupset bs ON d.name = bs.database_name
    LEFT JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
    WHERE d.state = 0 -- Online
    AND d.database_id > 4 -- Exclude system databases
    GROUP BY d.name, d.database_id, d.state_desc, d.recovery_model_desc
    ORDER BY d.name;
"@
    
    return Invoke-Sqlcmd -ServerInstance $Server -Database master -Query $query
}

function Test-BackupFileIntegrity {
    param([string]$BackupFile)
    
    try {
        $verifyQuery = "RESTORE VERIFYONLY FROM DISK = '$BackupFile'"
        Invoke-Sqlcmd -ServerInstance $env:COMPUTERNAME -Database master -Query $verifyQuery -ErrorAction Stop
        return @{ Success = $true; Message = 'Backup file is valid' }
    }
    catch {
        return @{ Success = $false; Message = "Backup file integrity check failed: $($_.Exception.Message)" }
    }
}

function Send-AlertEmail {
    param(
        [string]$Subject,
        [string]$Body,
        [string]$To,
        [string]$SmtpServer
    )
    
    try {
        $smtp = New-Object Net.Mail.SmtpClient($SmtpServer)
        $msg = New-Object Net.Mail.MailMessage
        $msg.From = "SQLBackupMonitor@company.com"
        $msg.To.Add($To)
        $msg.Subject = $Subject
        $msg.Body = $Body
        $msg.IsBodyHtml = $true
        $smtp.Send($msg)
        Write-Log "Alert email sent successfully"
    }
    catch {
        Write-Log "Failed to send alert email: $($_.Exception.Message)" 'ERROR'
    }
}

# Main monitoring process
try {
    Write-Log "Starting backup health monitoring for $SqlServer"
    
    $backupStatus = Get-BackupStatus -Server $SqlServer
    
    $issues = @()
    $alerts = @()
    
    foreach ($db in $backupStatus) {
        Write-Log "Checking backup status for $($db.DatabaseName)"
        
        # Check if database is online
        if ($db.state_desc -ne 'ONLINE') {
            $issues += "$($db.DatabaseName) is not online (State: $($db.state_desc))"
            continue
        }
        
        # Check backup frequency based on recovery model
        $daysSinceFullBackup = if ($db.LastFullBackup) { (Get-Date) - $db.LastFullBackup } else { [TimeSpan]::MaxValue }
        $daysSinceLogBackup = if ($db.LastLogBackup) { (Get-Date) - $db.LastLogBackup } else { [TimeSpan]::MaxValue }
        
        switch ($db.recovery_model_desc) {
            'FULL' {
                # Full recovery model: check for regular full, diff, and log backups
                if ($daysSinceFullBackup.TotalDays -gt 7) {
                    $issues += "$($db.DatabaseName): Last full backup was $($daysSinceFullBackup.TotalDays.ToString('F1')) days ago"
                    $alerts += "$($db.DatabaseName) - No full backup for $($daysSinceFullBackup.TotalDays.ToString('F1')) days"
                }
                
                if ($daysSinceLogBackup.TotalHours -gt 2) {
                    $issues += "$($db.DatabaseName): Last log backup was $($daysSinceLogBackup.TotalHours.ToString('F1')) hours ago"
                    $alerts += "$($db.DatabaseName) - No log backup for $($daysSinceLogBackup.TotalHours.ToString('F1')) hours"
                }
            }
            'SIMPLE' {
                # Simple recovery model: only full and differential backups
                if ($daysSinceFullBackup.TotalDays -gt 7) {
                    $issues += "$($db.DatabaseName): Last full backup was $($daysSinceFullBackup.TotalDays.ToString('F1')) days ago"
                    $alerts += "$($db.DatabaseName) - No full backup for $($daysSinceFullBackup.TotalDays.ToString('F1')) days"
                }
            }
        }
        
        # Check if any backups exist in last 7 days
        if ($db.BackupsLast7Days -eq 0) {
            $issues += "$($db.DatabaseName): No backups in the last 7 days"
            $alerts += "$($db.DatabaseName) - No backups performed in last 7 days"
        }
        
        # Test backup file integrity if backup file path exists
        if ($db.LastFullBackupPath -and (Test-Path $db.LastFullBackupPath)) {
            $integrity = Test-BackupFileIntegrity -BackupFile $db.LastFullBackupPath
            if (-not $integrity.Success) {
                $issues += "$($db.DatabaseName): Backup file integrity issue - $($db.LastFullBackupPath)"
                $alerts += "$($db.DatabaseName) - Backup file corruption detected"
            }
        }
    }
    
    # Generate monitoring report
    $report = @"
    <h2>SQL Server Backup Health Report</h2>
    <p>Generated: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')</p>
    <p>Server: $SqlServer</p>
    
    <h3>Summary</h3>
    <p>Total Databases: $($backupStatus.Count)</p>
    <p>Issues Found: $($issues.Count)</p>
    
    <h3>Database Backup Status</h3>
    <table border='1' style='border-collapse: collapse;'>
        <tr>
            <th>Database</th>
            <th>Recovery Model</th>
            <th>Last Full Backup</th>
            <th>Last Log Backup</th>
            <th>Backups (7 days)</th>
            <th>Status</th>
        </tr>
"@
    
    foreach ($db in $backupStatus) {
        $status = if ($issues | Where-Object { $_ -like "$($db.DatabaseName)*" }) { 'ISSUES' } else { 'OK' }
        $statusColor = if ($status -eq 'ISSUES') { 'red' } else { 'green' }
        
        $report += @"
        <tr>
            <td>$($db.DatabaseName)</td>
            <td>$($db.recovery_model_desc)</td>
            <td>$($db.LastFullBackup)</td>
            <td>$($db.LastLogBackup)</td>
            <td>$($db.BackupsLast7Days)</td>
            <td style='color: $statusColor;'>$status</td>
        </tr>
"@
    }
    
    if ($issues.Count -gt 0) {
        $report += @"
        </table>
        
        <h3>Issues Found</h3>
        <ul>
"@
        
        foreach ($issue in $issues) {
            $report += "<li>$issue</li>"
        }
        
        $report += @"
        </ul>
        
        <h3>Recommended Actions</h3>
        <ul>
        <li>Review and update backup schedules as needed</li>
        <li>Investigate and resolve backup failures</li>
        <li>Test backup restoration procedures</li>
        <li>Verify backup file integrity</li>
        <li>Review backup storage capacity</li>
        </ul>
"@
    }
    
    $report += @"
    </table>
"@
    
    # Save report to file
    $reportPath = "C:\Reports\BackupHealth\BackupHealth_$(Get-Date -Format 'yyyyMMdd_HHmmss').html"
    if (-not (Test-Path (Split-Path $reportPath))) {
        New-Item -ItemType Directory -Path (Split-Path $reportPath) -Force | Out-Null
    }
    $report | Out-File -FilePath $reportPath -Encoding UTF8
    
    Write-Log "Backup health report saved to: $reportPath"
    
    # Send alert email if issues found
    if ($alerts.Count -gt 0) {
        $emailSubject = "SQL Backup Issues Detected - $SqlServer"
        $emailBody = $report
        Send-AlertEmail -Subject $emailSubject -Body $emailBody -To $AlertEmail -SmtpServer $SmtpServer
    }
    
    Write-Log "Backup health monitoring completed"
}
catch {
    Write-Log "Backup health monitoring failed: $($_.Exception.Message)" 'ERROR'
}
```

### Hands-On Lab: Complete Backup Strategy Implementation

**Scenario**: Implement a comprehensive backup strategy for a multi-database environment with automated monitoring and DR testing.

**Tasks**:
1. Create backup strategy documentation including RTO/RPO requirements
2. Implement automated backup jobs for all databases
3. Configure offsite backup to Azure Blob Storage
4. Set up backup monitoring and alerting
5. Create DR testing procedures
6. Perform backup restoration testing
7. Document troubleshooting procedures

**Deliverables**:
- Backup strategy document with RTO/RPO analysis
- SQL Server Agent backup job configurations
- PowerShell automation scripts
- Monitoring dashboard queries
- DR testing reports
- Troubleshooting guides

## Summary

This week covered comprehensive backup and disaster recovery strategies for SQL Server environments. You learned to:

- Design and implement automated backup strategies for different database types and recovery models
- Configure and manage offsite backup solutions using Azure Blob Storage and other cloud providers
- Develop comprehensive disaster recovery plans with proper testing procedures
- Implement backup monitoring and troubleshooting systems
- Create PowerShell scripts for advanced backup automation
- Perform regular DR testing and validation

Mastering these skills will ensure you can protect critical database assets and maintain business continuity in the face of disasters and system failures.