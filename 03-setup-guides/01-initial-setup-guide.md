# SQL Server Initial Setup and Installation Guide

## Table of Contents
1. [Prerequisites and System Requirements](#prerequisites)
2. [SQL Server Installation Process](#installation)
3. [SQL Server Configuration](#configuration)
4. [SQL Server Management Studio (SSMS) Setup](#ssms-setup)
5. [Post-Installation Verification](#verification)
6. [Troubleshooting Common Issues](#troubleshooting)
7. [Performance Optimization Tips](#optimization)

## Prerequisites and System Requirements {#prerequisites}

### Hardware Requirements

**Minimum Hardware Requirements:**
- CPU: 1.4 GHz 64-bit processor (2.0 GHz recommended)
- RAM: 512 MB (2 GB recommended)
- Hard Drive: 2.2 GB available space
- Monitor: 1024 x 768 resolution

**Recommended Hardware for Production:**
- CPU: 2.0 GHz or faster with multiple cores (4-8 cores minimum)
- RAM: 8 GB minimum (16-32 GB recommended)
- Hard Drive: SSD with 100+ GB available space
- Monitor: 1280 x 1024 or higher resolution

### Software Requirements

**Operating System Support:**
- Windows Server 2016 or later
- Windows 10 or later (for development/learning)
- SQL Server 2019 or later (2022 recommended for new installations)

**Prerequisites Software:**
- .NET Framework 4.7.2 or later
- Visual C++ Redistributable for Visual Studio
- Windows PowerShell 5.0 or later

### Network Requirements

- Stable internet connection for downloading installation files
- Local network access if installing on multiple servers
- Firewall rules configured for SQL Server ports (default: 1433)

### Pre-Installation Checklist

**User Account Control (UAC):**
```
1. Right-click on Start button and select "Windows PowerShell (Admin)"
2. Execute: Set-ExecutionPolicy RemoteSigned
3. Confirm execution policy change when prompted
```

**Windows Features Required:**
```
1. Open "Turn Windows features on or off"
2. Enable:
   - .NET Framework 3.5 (includes .NET 2.0 and 3.0)
   - Windows PowerShell (if not already installed)
```

**Antivirus Exclusion:**
```
Configure your antivirus to exclude SQL Server directories:
- Program Files\Microsoft SQL Server\
- Data directory where databases will be stored
```

## SQL Server Installation Process {#installation}

### Step 1: Download Installation Media

**Option A: SQL Server Evaluation Edition (Free)**
```
1. Visit Microsoft Evaluation Center
2. Search for "SQL Server 2022"
3. Download SQL Server 2022 Developer Edition (Free)
4. Download SQL Server Management Studio (SSMS)
```

**Option B: Azure SQL Edge (Lightweight Option)**
```
For containerized or cloud-native development:
1. Visit Docker Hub
2. Pull mcr.microsoft.com/azure-sql-edge:latest
3. Follow Docker container setup instructions
```

### Step 2: Run SQL Server Setup

```
1. Locate the downloaded SQL Server installation file (e.g., SQLServer2022-DEV-ENU-x64.exe)
2. Right-click and select "Run as administrator"
3. Wait for setup preparation to complete
```

### Step 3: Installation Type Selection

**New Standalone Installation:**
```
1. Select "New standalone SQL Server installation or add features to an existing installation"
2. Click "Next" through the Product Key screen (Evaluation or Developer)
3. Accept license terms and click "Next"
```

### Step 4: Feature Selection

**For Complete Installation:**
```
Select the following features:
☑ SQL Server Database Engine Services
☑ Full-Text and Semantic Extractions for Search
☑ SQL Server Replication
☑ LocalDB (for development)
☑ Client Tools Connectivity
☑ Client Tools SDK
☑ LocalDB
☑ Documentation Components (optional)
```

**For Minimal Installation:**
```
Select only:
☑ SQL Server Database Engine Services
☑ Client Tools Connectivity
```

### Step 5: Instance Configuration

**Default Instance:**
```
Instance ID: MSSQLSERVER
Instance root directory: C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\
```

**Named Instance (Recommended for Multiple Installations):**
```
Instance ID: [Choose meaningful name, e.g., "SQL2022"]
Instance root directory: [Auto-generated based on instance name]
Instance name: [Same as Instance ID]
```

**Important Considerations:**
- Default instance uses port 1433 automatically
- Named instances use dynamic ports by default
- Record your instance name for connection strings

### Step 6: Server Configuration

**Service Accounts:**
```
SQL Server Database Engine: NT AUTHORITY\NETWORK SERVICE (recommended for basic setup)
SQL Server Agent: NT AUTHORITY\NETWORK SERVICE
```

**Collation Settings:**
```
Default collation: SQL_Latin1_General_CP1_CI_AS
Consider database-specific collation if needed for international characters
```

### Step 7: Database Engine Configuration

**Authentication Mode:**
```
Authentication Mode: Mixed Mode (SQL Server and Windows authentication)
Set SA password: [Create strong password and record securely]
```

**SQL Server Administrators:**
```
1. Click "Add" button
2. Add your Windows user account
3. Add any additional administrators
4. Use "Check Names" to validate accounts
```

**Data Directories:**
```
Data root directory: C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\
Data directory: C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\Data\
Log directory: C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\Log\
TempDB directory: C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\TempDB\
Backup directory: C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\Backup\
```

### Step 8: Complete Installation

```
1. Review all configuration settings
2. Click "Next" to begin installation
3. Wait for installation to complete (15-30 minutes)
4. Restart system when prompted
5. Verify installation success message
```

## SQL Server Configuration {#configuration}

### Step 1: Enable TCP/IP Protocol

**Using SQL Server Configuration Manager:**
```
1. Open SQL Server Configuration Manager
2. Navigate to: SQL Server Network Configuration > Protocols for [Instance Name]
3. Right-click "TCP/IP" and select "Enable"
4. Right-click "TCP/IP" and select "Properties"
5. Set "Listen All" to "Yes"
6. On "IP Addresses" tab, configure IP settings:
   - IP1 (127.0.0.1): Set TCP Port to 1433
   - IP2 (your local IP): Set TCP Port to 1433
7. Click "Apply" and "OK"
8. Restart SQL Server service
```

**PowerShell Method:**
```powershell
# Run as Administrator
Import-Module SQLPS

# Get SQL Server instance
$instance = "MSSQLSERVER"
$server = New-Object -TypeName Microsoft.SqlServer.Management.Smo.Server $instance

# Enable TCP/IP
$server.Settings.TcpEnabled = $true
$server.Settings.Alter()
```

### Step 2: Configure SQL Server Port

**Static Port Configuration:**
```
1. Open SQL Server Configuration Manager
2. Navigate to: SQL Server Network Configuration > Protocols for [Instance Name] > TCP/IP > IP Addresses > IPAll
3. Set "TCP Port" to 1433
4. Set "TCP Dynamic Ports" to blank (empty)
5. Click "Apply" and "OK"
6. Restart SQL Server service
```

### Step 3: Windows Firewall Configuration

**Create Firewall Rules:**
```
# Open Command Prompt as Administrator
netsh advfirewall firewall add rule name="SQL Server" dir=in action=allow protocol=TCP localport=1433
netsh advfirewall firewall add rule name="SQL Server Browser" dir=in action=allow protocol=UDP localport=1434
```

**Alternative: Using PowerShell**
```powershell
New-NetFirewallRule -DisplayName "SQL Server" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow
New-NetFirewallRule -DisplayName "SQL Server Browser" -Direction Inbound -Protocol UDP -LocalPort 1434 -Action Allow
```

### Step 4: Database Engine Configuration

**Configure Security Settings:**
```sql
-- Connect to SQL Server using SSMS
-- Execute the following to configure security:

-- Enable advanced options
sp_configure 'show advanced options', 1;
RECONFIGURE;

-- Enable xp_cmdshell (if needed for specific tasks)
sp_configure 'xp_cmdshell', 1;
RECONFIGURE;

-- Set maximum memory usage
sp_configure 'max server memory (MB)', 4096;  -- Adjust based on your system
RECONFIGURE;
```

**Configure Database Settings:**
```sql
-- Set default data and log locations
USE master;
GO

-- Create procedures for future use
CREATE OR ALTER PROCEDURE sp_setdefaultdatalocation
    @DataPath NVARCHAR(260)
AS
BEGIN
    DECLARE @SQL NVARCHAR(500);
    SET @SQL = 'EXEC xp_create_subdir N''' + @DataPath + '''';
    EXEC sp_executesql @SQL;
END;
GO
```

## SQL Server Management Studio (SSMS) Setup {#ssms-setup}

### Step 1: Download and Install SSMS

**Download Latest SSMS:**
```
1. Visit: https://learn.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms
2. Download SSMS version 19.0 or later
3. Run installer as Administrator
4. Follow installation wizard prompts
5. Restart computer when complete
```

### Step 2: Initial SSMS Configuration

**First Time Connection:**
```
1. Launch SSMS
2. Server name: localhost or [YourServerName]\[InstanceName]
3. Authentication: SQL Server Authentication
4. Login: sa
5. Password: [The password you set during installation]
6. Click "Connect"
```

**Set Up Windows Authentication:**
```
1. In SSMS Object Explorer, right-click server name
2. Select "Properties"
3. In "Security" section, select "SQL Server and Windows Authentication mode"
4. Click "OK" and restart SQL Server service
```

### Step 3: SSMS Optimization

**Configure Editor Settings:**
```
1. Go to Tools > Options
2. Navigate to: Query Results > SQL Server > Results to Grid
3. Set defaults:
   - Display numbers in scientific notation: Unchecked
   - Include column headers when copying or saving results: Checked
4. Navigate to: Query Results > SQL Server > Results to Text
5. Set defaults:
   - Maximum number of characters displayed in each column: 8192
   - Include column headers in the result set: Checked
```

**Configure Keyboard Shortcuts:**
```
1. Go to Tools > Options > Environment > Keyboard
2. Common shortcuts to configure:
   - Ctrl+Shift+H: Comment Selection
   - Ctrl+H: Uncomment Selection
   - Ctrl+K, Ctrl+C: Comment Selection
   - Ctrl+K, Ctrl+U: Uncomment Selection
3. Click "OK" to save
```

### Step 4: Create Administrative User

**Create Windows Authentication Login:**
```sql
-- Run in SSMS connected to your SQL Server
USE master;
GO

-- Create a Windows authentication login
CREATE LOGIN [DOMAIN\YourUserName] FROM WINDOWS;
GO

-- Add user to server roles
ALTER SERVER ROLE sysadmin ADD MEMBER [DOMAIN\YourUserName];
GO
```

## Post-Installation Verification {#verification}

### Step 1: Service Status Check

**Using Services.msc:**
```
1. Press Win+R, type "services.msc", press Enter
2. Verify the following services are running:
   - SQL Server (MSSQLSERVER) or SQL Server ([InstanceName])
   - SQL Server Agent (MSSQLSERVER) or SQL Server Agent ([InstanceName])
   - SQL Server Browser
```

**Using PowerShell:**
```powershell
# Check SQL Server services
Get-Service | Where-Object {$_.Name -like "*SQL*"} | Format-Table Name, Status, StartType
```

### Step 2: Database Engine Connection Test

**Using SSMS:**
```
1. Open SSMS
2. Try connecting with Windows Authentication
3. Verify Object Explorer shows databases, tables, etc.
4. Run test query: SELECT @@VERSION
```

**Using Command Line:**
```cmd
sqlcmd -S localhost -Q "SELECT @@VERSION"
```

### Step 3: Test Remote Connectivity

**Local Network Test:**
```cmd
# Test from another machine on the network
sqlcmd -S [ServerIP] -U sa -P [Password] -Q "SELECT @@VERSION"
```

**Port Connectivity Test:**
```cmd
# Test port 1433 is accessible
telnet [ServerIP] 1433
```

### Step 4: Performance Verification

```sql
-- Run these diagnostic queries to verify performance
-- Check SQL Server version and edition
SELECT @@VERSION;

-- Check current connections
SELECT COUNT(*) AS ActiveConnections 
FROM sys.dm_exec_sessions 
WHERE is_user_process = 1;

-- Check database files
USE master;
GO
SELECT 
    name,
    physical_name,
    size * 8 / 1024 AS SizeMB,
    state_desc
FROM sys.master_files;
```

## Troubleshooting Common Issues {#troubleshooting}

### Issue 1: Installation Fails with Prerequisites Error

**Symptoms:**
- Setup fails with ".NET Framework" error
- "Visual C++ Redistributable" error message
- "Windows PowerShell" not found

**Solutions:**
```
1. Install Prerequisites:
   - Download and install .NET Framework 4.7.2 or later
   - Install Visual C++ Redistributable for Visual Studio
   - Enable Windows PowerShell in Windows Features

2. Run Windows Update:
   - Install all pending Windows updates
   - Restart system and retry installation

3. Manual Installation:
   - Download individual prerequisite installers
   - Install in order: .NET Framework, Visual C++, PowerShell
   - Restart and retry SQL Server installation
```

### Issue 2: SQL Server Service Won't Start

**Symptoms:**
- Service status shows "Stopped" or "Failed to start"
- Error in Windows Event Viewer
- Cannot connect to SQL Server

**Solutions:**
```
1. Check SQL Server Error Log:
   Location: C:\Program Files\Microsoft SQL Server\MSSQL[Version].[Instance]\MSSQL\Log
   
2. Common Causes and Fixes:
   - Permission issues: Run service with correct account
   - Port conflicts: Check if another service uses port 1433
   - Corrupted master database: Rebuild master database
   
3. Rebuild Master Database:
   Setup.exe /QUIET /ACTION=REBUILDDATABASE /INSTANCENAME=[Instance] /SAPWD=[NewSAPassword]
```

### Issue 3: Cannot Connect Remotely

**Symptoms:**
- Timeout errors when connecting from network
- "Network-related error" messages
- Local connections work, remote connections fail

**Solutions:**
```
1. Verify Network Configuration:
   - Ensure TCP/IP protocol is enabled
   - Check firewall rules are correctly configured
   - Verify SQL Server Browser service is running
   
2. Test Network Connectivity:
   - Use telnet to test port 1433 connectivity
   - Check if port is listening: netstat -an | findstr 1433
   
3. SQL Server Configuration:
   - Enable remote connections in SSMS
   - Configure static port (1433) instead of dynamic
   - Restart services after configuration changes
```

### Issue 4: Authentication Failures

**Symptoms:**
- "Login failed" error messages
- SA account authentication issues
- Windows authentication problems

**Solutions:**
```
1. Check Authentication Mode:
   - Verify mixed mode is enabled
   - Restart SQL Server service after changing authentication mode
   
2. Reset SA Password:
   - Start SQL Server in single-user mode
   - Connect using SQLCMD
   - Execute: ALTER LOGIN sa WITH PASSWORD = '[NewPassword]';
   
3. Enable Windows Authentication:
   - Verify user is member of appropriate groups
   - Check group policy settings
   - Add user to SQL Server logins
```

### Issue 5: Performance Problems

**Symptoms:**
- Slow query execution
- High CPU or memory usage
- Timeouts during operations

**Solutions:**
```
1. Check Resource Usage:
   - Monitor CPU and memory usage
   - Review wait statistics: SELECT * FROM sys.dm_os_wait_stats;
   
2. Optimize Configuration:
   - Set max server memory appropriately
   - Enable SQL Server Agent for maintenance tasks
   - Configure tempdb with multiple data files
   
3. Database Maintenance:
   - Update statistics regularly
   - Rebuild indexes periodically
   - Monitor file growth settings
```

## Performance Optimization Tips {#optimization}

### Memory Configuration

**Set Appropriate Memory Limits:**
```sql
-- Check current memory settings
sp_configure 'max server memory (MB)';

-- Set max memory to 75% of available RAM (example for 8GB system)
sp_configure 'max server memory (MB)', 6144;
RECONFIGURE;
```

### TempDB Configuration

**Optimize TempDB:**
```sql
-- Create multiple TempDB data files (one per CPU core, up to 8)
ALTER DATABASE tempdb 
ADD FILE (
    NAME = tempdev2,
    FILENAME = 'C:\SQLData\tempdev2.ndf',
    SIZE = 100MB,
    FILEGROWTH = 10MB
);
```

### Index Maintenance

**Create Maintenance Plan:**
```sql
-- Create stored procedure for index maintenance
CREATE PROCEDURE sp_MaintainIndexes
AS
BEGIN
    DECLARE @DatabaseName NVARCHAR(128);
    DECLARE @SQL NVARCHAR(MAX);
    
    DECLARE db_cursor CURSOR FOR
    SELECT name FROM sys.databases 
    WHERE state = 0 AND database_id > 4; -- Skip system databases
    
    OPEN db_cursor;
    FETCH NEXT FROM db_cursor INTO @DatabaseName;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @SQL = N'USE ' + QUOTENAME(@DatabaseName) + N';
        EXEC sp_msforeachtable "DBCC DBREINDEX(''?'')";
        UPDATE STATISTICS ? WITH FULLSCAN;';
        
        EXEC sp_executesql @SQL;
        FETCH NEXT FROM db_cursor INTO @DatabaseName;
    END
    
    CLOSE db_cursor;
    DEALLOCATE db_cursor;
END;
```

### Regular Maintenance Tasks

**Automated Backup Script:**
```sql
-- Create maintenance job in SQL Server Agent
USE msdb;
GO

-- Create backup job
EXEC dbo.sp_add_job
    @job_name = N'Database Backup',
    @enabled = 1,
    @description = N'Automated database backup',
    @owner_login_name = N'sa';

-- Add backup job step
EXEC sp_add_jobstep
    @job_name = N'Database Backup',
    @step_name = N'Backup All User Databases',
    @command = N'EXEC sp_BackupAllDatabases',
    @retry_attempts = 3;
```

This comprehensive guide provides the foundation for setting up SQL Server in production and development environments. Follow each step carefully and refer to the troubleshooting section for common issues. Remember to document your specific configuration choices for future reference and maintenance.

For additional support, consult the Microsoft SQL Server documentation or reach out to database administrators in your organization.