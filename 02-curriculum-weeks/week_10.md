# Week 10: High Availability Solutions

## Overview

This week focuses on implementing and managing SQL Server high availability solutions to ensure continuous database operations and minimize downtime. You'll learn about Always On Availability Groups, clustering, and replication technologies that are essential for mission-critical database environments.

## Learning Objectives

By the end of this week, you will be able to:
- Design and implement Always On Availability Groups for maximum availability
- Configure and manage SQL Server failover clustering instances
- Set up and maintain database replication for read-scale scenarios
- Plan and execute failover procedures with minimal downtime
- Monitor high availability health and performance metrics
- Troubleshoot common HA configuration issues and failures
- Implement disaster recovery strategies using HA technologies

## Monday - Always On Availability Groups Fundamentals

### Always On Availability Groups Architecture

Always On Availability Groups provide high availability and disaster recovery by maintaining synchronized copies of databases across multiple SQL Server instances.

**Architecture Components:**
```sql
-- Analyze current Always On configuration
SELECT 
    ag.name AS AvailabilityGroupName,
    ag.group_id,
    ag.resource_id,
    ag.resource_version,
    ag.resource_version_guid,
    ag.failure_condition_level,
    ag.health_check_timeout,
    ag-automated_backup_preference,
    ag-automated_backup_preference_desc,
    -- Database count in the group
    (SELECT COUNT(*) 
     FROM sys.availability_databases_cluster adc 
     WHERE adc.group_id = ag.group_id) AS DatabaseCount,
    -- Replica count
    (SELECT COUNT(*) 
     FROM sys.availability_replicas ar 
     WHERE ar.group_id = ag.group_id) AS ReplicaCount,
    -- Listener information
    (SELECT COUNT(*) 
     FROM sys.availability_group_listeners agl 
     WHERE agl.group_id = ag.group_id) AS ListenerCount,
    -- Primary replica information
    (SELECT ar.replica_server_name 
     FROM sys.availability_replicas ar 
     WHERE ar.group_id = ag.group_id AND ar.role = 1) AS PrimaryReplicaServer,
    (SELECT ar.endpoint_url 
     FROM sys.availability_replicas ar 
     WHERE ar.group_id = ag.group_id AND ar.role = 1) AS PrimaryReplicaEndpoint
FROM sys.availability_groups ag;

-- Detailed replica information
SELECT 
    ar.replica_server_name AS ServerName,
    ar.endpoint_url AS EndpointURL,
    ar.availability_mode_desc AS AvailabilityMode,
    ar.failover_mode_desc AS FailoverMode,
    ar.session_timeout AS SessionTimeout,
    ar.primary_role_allow_connections_desc AS PrimaryConnections,
    ar.secondary_role_allow_connections_desc AS SecondaryConnections,
    ar.backup_priority AS BackupPriority,
    ar.endpoint_url,
    ar.read_only_routing_url,
    -- Connection and session information
    ars.connected_state_desc AS ConnectionState,
    ars.operational_state_desc AS OperationalState,
    ars.recovery_health_desc AS RecoveryHealth,
    ars.synchronization_health_desc AS SynchronizationHealth,
    ars.suspend_reason_desc AS SuspendReason,
    -- Performance metrics
    ars.log_send_queue_size AS LogSendQueueSizeKB,
    ars.log_send_rate AS LogSendRateKBps,
    ars.redo_queue_size AS RedoQueueSizeKB,
    ars.redo_rate AS RedoRateKBps,
    ars.filestream_send_rate AS FileStreamRateKBps
FROM sys.availability_replicas ar
LEFT JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
ORDER BY ar.role DESC, ar.replica_server_name;
```

#### Database Synchronization Status Analysis
```sql
-- Analyze database synchronization status
SELECT 
    adc.database_name AS DatabaseName,
    ar.replica_server_name AS ReplicaServerName,
    ars.role_desc AS ReplicaRole,
    ars.connected_state_desc AS ConnectionState,
    ars.operational_state_desc AS OperationalState,
    ars.synchronization_health_desc AS SynchronizationHealth,
    ars.synchronization_state_desc AS SynchronizationState,
    ars.database_state_desc AS DatabaseState,
    -- Performance metrics
    ars.log_send_queue_size AS LogSendQueueKB,
    ars.log_send_rate AS LogSendRateKBps,
    ars.redo_queue_size AS RedoQueueKB,
    ars.redo_rate AS RedoRateKBps,
    ars.filestream_send_rate AS FileStreamRateKBps,
    -- Database size and growth
    (SELECT SUM(mf.size * 8.0 / 1024 / 1024) 
     FROM sys.master_files mf 
     WHERE mf.database_id = database_id(adc.database_name) AND mf.type = 0) AS DataFileSizeGB,
    (SELECT SUM(mf.size * 8.0 / 1024 / 1024) 
     FROM sys.master_files mf 
     WHERE mf.database_id = database_id(adc.database_name) AND mf.type = 1) AS LogFileSizeGB,
    -- Last activity
    ars.last_commit_time AS LastCommitTime,
    ars.last_send_time AS LastSendTime,
    ars.last_received_time AS LastReceivedTime,
    ars.last_redone_time AS LastRedoneTime,
    -- Status indicators
    CASE 
        WHEN ars.synchronization_state_desc != 'SYNCHRONIZED' THEN 'OUT_OF_SYNC'
        WHEN ars.synchronization_health_desc != 'HEALTHY' THEN 'UNHEALTHY'
        WHEN ars.log_send_queue_size > 1024000 THEN 'LARGE_LAG'  -- > 1GB
        WHEN ars.redo_queue_size > 512000 THEN 'LARGE_REDO_LAG'  -- > 512MB
        ELSE 'HEALTHY'
    END AS HealthStatus
FROM sys.availability_databases_cluster adc
INNER JOIN sys.availability_replicas ar ON adc.group_id = ar.group_id
INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
ORDER BY adc.database_name, ar.replica_server_name;
```

### Listener Configuration and Management

**Always On Listener Setup and Management:**
```sql
-- Create Always On Availability Group Listener
-- This must be executed on the primary replica
USE master;
GO

-- Create availability group listener
ALTER AVAILABILITY GROUP [SalesAG]
ADD LISTENER N'SalesAG-Listener' (
    WITH IP
    ((N'192.168.1.100', N'255.255.255.0')),
    PORT=1433
);
GO

-- Monitor listener health and connectivity
SELECT 
    agl.listener_id,
    agl.group_id,
    agl.dns_name AS ListenerName,
    agl.port AS ListenerPort,
    agl.ip_configuration_string_from_cluster,
    agl.state_desc AS ListenerState,
    -- Get IP addresses
    ipi.ip_address,
    ipi.ip_subnet_mask,
    ipi.state_desc AS IPState,
    -- Get replica connection information
    ar.replica_server_name,
    ar.role_desc AS ReplicaRole,
    -- Connection testing results
    CASE 
        WHEN ar.role_desc = 'PRIMARY' THEN 'Read-Write Endpoint'
        WHEN ar.role_desc = 'SECONDARY' AND ar.secondary_role_allow_connections_desc = 'ALL' THEN 'Read-Write and Read-Only'
        WHEN ar.role_desc = 'SECONDARY' AND ar.secondary_role_allow_connections_desc = 'READ_ONLY' THEN 'Read-Only'
        ELSE 'No Connections Allowed'
    END AS ConnectionMode
FROM sys.availability_group_listeners agl
INNER JOIN sys.availability_replicas ar ON agl.group_id = ar.group_id
OUTER APPLY sys.availability_group_listener_ip_addresses ipi ON agl.listener_id = ipi.listener_id
ORDER BY agl.listener_id, ar.role DESC;

-- Test listener connectivity
-- This query helps verify that clients can connect through the listener
SELECT 
    'Listener Connectivity Test' AS TestType,
    @@SERVERNAME AS CurrentServer,
    agl.dns_name AS ListenerName,
    agl.port AS ListenerPort,
    CASE 
        WHEN ar.role = 1 THEN 'This is the primary replica'
        WHEN ar.secondary_role_allow_connections_desc = 'ALL' THEN 'Readable secondary replica'
        WHEN ar.secondary_role_allow_connections_desc = 'READ_ONLY' THEN 'Read-only secondary replica'
        ELSE 'Replica not accepting connections'
    END AS ConnectionStatus
FROM sys.availability_group_listeners agl
INNER JOIN sys.availability_replicas ar ON agl.group_id = ar.group_id
WHERE ar.replica_server_name = @@SERVERNAME;

-- Create monitoring procedure for Always On health
CREATE OR ALTER PROCEDURE sp_MonitorAlwaysOnHealth
    @AlertOnLargeLagKB BIGINT = 1048576, -- 1GB
    @AlertOnLargeRedoLagKB BIGINT = 524288, -- 512MB
    @EmailRecipients NVARCHAR(MAX) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @HealthReport NVARCHAR(MAX);
    DECLARE @AlertLevel NVARCHAR(20);
    DECLARE @IssueCount INT = 0;
    
    SET @HealthReport = 'Always On Availability Groups Health Report' + CHAR(13) + CHAR(10);
    SET @HealthReport = @HealthReport + 'Generated: ' + CONVERT(VARCHAR(30), GETDATE(), 120) + CHAR(13) + CHAR(10);
    SET @HealthReport = @HealthReport + 'Server: ' + @@SERVERNAME + CHAR(13) + CHAR(10) + CHAR(13) + CHAR(10);
    
    -- Check availability group status
    SELECT @HealthReport = @HealthReport + 
        '=== Availability Group Status ===' + CHAR(13) + CHAR(10),
        ag.name AS AGName,
        CASE ar.role
            WHEN 1 THEN 'PRIMARY'
            WHEN 2 THEN 'SECONDARY'
            ELSE 'UNKNOWN'
        END AS CurrentRole,
        ar.connected_state_desc AS ConnectionState,
        ar.operational_state_desc AS OperationalState,
        ars.synchronization_state_desc AS SyncState,
        ars.synchronization_health_desc AS Health,
        ars.log_send_queue_size AS LogSendQueueKB,
        ars.redo_queue_size AS RedoQueueKB,
        CASE 
            WHEN ars.log_send_queue_size > @AlertOnLargeLagKB THEN 'LARGE_LAG'
            WHEN ars.redo_queue_size > @AlertOnLargeRedoLagKB THEN 'LARGE_REDO_LAG'
            ELSE 'NORMAL'
        END AS LagStatus
    FROM sys.availability_groups ag
    INNER JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
    INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
    WHERE ar.replica_server_name = @@SERVERNAME;
    
    -- Check database synchronization status
    SELECT @HealthReport = @HealthReport + CHAR(13) + CHAR(10) + 
        '=== Database Synchronization Status ===' + CHAR(13) + CHAR(10),
        adc.database_name AS DatabaseName,
        ar.replica_server_name AS ReplicaServer,
        ars.synchronization_state_desc AS SyncState,
        ars.synchronization_health_desc AS Health,
        ars.log_send_queue_size AS LogQueueKB,
        ars.redo_queue_size AS RedoQueueKB,
        CASE 
            WHEN ars.synchronization_state_desc != 'SYNCHRONIZED' THEN 'SYNC_ISSUE'
            WHEN ars.log_send_queue_size > @AlertOnLargeLagKB THEN 'LARGE_LAG'
            WHEN ars.redo_queue_size > @AlertOnLargeRedoLagKB THEN 'LARGE_REDO_LAG'
            ELSE 'HEALTHY'
        END AS DatabaseStatus
    FROM sys.availability_databases_cluster adc
    INNER JOIN sys.availability_replicas ar ON adc.group_id = ar.group_id
    INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
    ORDER BY adc.database_name;
    
    -- Check for issues
    SELECT @IssueCount = COUNT(*)
    FROM sys.availability_databases_cluster adc
    INNER JOIN sys.availability_replicas ar ON adc.group_id = ar.group_id
    INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
    WHERE ar.replica_server_name = @@SERVERNAME
    AND (ars.synchronization_state_desc != 'SYNCHRONIZED' 
         OR ars.synchronization_health_desc != 'HEALTHY'
         OR ars.log_send_queue_size > @AlertOnLargeLagKB
         OR ars.redo_queue_size > @AlertOnLargeRedoLagKB);
    
    -- Determine alert level
    IF @IssueCount > 0
    BEGIN
        IF EXISTS (
            SELECT 1 FROM sys.availability_databases_cluster adc
            INNER JOIN sys.availability_replicas ar ON adc.group_id = ar.group_id
            INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
            WHERE ar.replica_server_name = @@SERVERNAME
            AND ars.synchronization_state_desc != 'SYNCHRONIZED'
        )
            SET @AlertLevel = 'CRITICAL';
        ELSE
            SET @AlertLevel = 'WARNING';
    END
    ELSE
        SET @AlertLevel = 'HEALTHY';
    
    -- Add summary
    SET @HealthReport = @HealthReport + CHAR(13) + CHAR(10) + 
        '=== Health Summary ===' + CHAR(13) + CHAR(10) +
        'Alert Level: ' + @AlertLevel + CHAR(13) + CHAR(10) +
        'Total Issues Found: ' + CAST(@IssueCount AS VARCHAR) + CHAR(13) + CHAR(10) +
        'Timestamp: ' + CONVERT(VARCHAR(30), GETDATE(), 120);
    
    -- Log results
    INSERT INTO AlwaysOnHealthLog (ServerName, AlertLevel, IssueCount, HealthReport, CheckTime)
    VALUES (@@SERVERNAME, @AlertLevel, @IssueCount, @HealthReport, GETDATE());
    
    -- Display results
    PRINT @HealthReport;
    
    -- Send alerts if configured
    IF @IssueCount > 0 AND @EmailRecipients IS NOT NULL
    BEGIN
        -- In real implementation, send email using sp_send_dbmail
        PRINT 'ALERT: ' + @AlertLevel + ' level Always On health issues detected';
        -- EXEC msdb.dbo.sp_send_dbmail @recipients = @EmailRecipients, @subject = 'Always On Health Alert', @body = @HealthReport;
    END
    
    RETURN @IssueCount;
END;
GO

-- Create table to log health checks
CREATE TABLE AlwaysOnHealthLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    ServerName NVARCHAR(128),
    AlertLevel NVARCHAR(20),
    IssueCount INT,
    HealthReport NVARCHAR(MAX),
    CheckTime DATETIME DEFAULT GETDATE()
);
```

## Tuesday - Always On Availability Groups Implementation

### Prerequisites and Configuration

**Windows Cluster Configuration:**
```powershell
# Always On Availability Groups prerequisites script
# Save as: Configure-AlwaysOnPrerequisites.ps1

param(
    [Parameter(Mandatory=$true)]
    [string]$ClusterName,
    
    [Parameter(Mandatory=$true)]
    [string[]]$ClusterNodes,
    
    [Parameter(Mandatory=$false)]
    [string]$Domain = $env:USERDNSDOMAIN,
    
    [Parameter(Mandatory=$false)]
    [string]$ServiceAccount = "DOMAIN\SQLService",
    
    [Parameter(Mandatory=$false)]
    [switch]$EnableFeature,
    [Parameter(Mandatory=$false)]
    [switch]$ConfigureCluster,
    [Parameter(Mandatory=$false)]
    [switch]$EnableAlwaysOn
)

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
    Add-Content -Path "C:\Logs\AlwaysOnSetup_$(Get-Date -Format 'yyyyMMdd').log" -Value $logMessage
}

function Test-Administrator {
    $currentUser = [Security.Principal.WindowsIdentity]::GetCurrent()
    $principal = New-Object Security.Principal.WindowsPrincipal($currentUser)
    return $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
}

function Enable-NetFrameworkFeature {
    try {
        Write-Log "Enabling .NET Framework 3.5 features"
        Enable-WindowsOptionalFeature -Online -FeatureName NetFx3 -All
        Write-Log ".NET Framework 3.5 enabled successfully"
        return $true
    }
    catch {
        Write-Log "Failed to enable .NET Framework 3.5: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

function Enable-FailoverClustering {
    try {
        Write-Log "Installing Failover Clustering feature"
        Install-WindowsFeature -Name Failover-Clustering -IncludeManagementTools
        Write-Log "Failover Clustering feature installed successfully"
        return $true
    }
    catch {
        Write-Log "Failed to install Failover Clustering: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

function Test-ClusterConnectivity {
    param([string[]]$Nodes)
    
    Write-Log "Testing connectivity between cluster nodes"
    
    foreach ($node in $Nodes) {
        try {
            $connection = Test-Connection -ComputerName $node -Count 2 -Quiet
            if ($connection) {
                Write-Log "Node $node is reachable"
            }
            else {
                Write-Log "Node $node is not reachable" 'ERROR'
                return $false
            }
        }
        catch {
            Write-Log "Failed to connect to node $node`: $($_.Exception.Message)" 'ERROR'
            return $false
        }
    }
    
    return $true
}

function Configure-WindowsCluster {
    param(
        [string]$ClusterName,
        [string[]]$Nodes,
        [string]$Domain
    )
    
    try {
        Write-Log "Creating Windows Cluster $ClusterName"
        
        # Create cluster
        $clusterParams = @{
            Name = $ClusterName
            Node = $Nodes
            StaticAddress = (Get-ClusterNetwork).Address
            IgnoreNetwork = @()
            NoStorage = $true
        }
        
        New-Cluster @clusterParams | Out-Null
        
        Write-Log "Cluster $ClusterName created successfully"
        
        # Set cluster quorum configuration
        Write-Log "Configuring cluster quorum"
        Set-ClusterQuorum -Cluster $ClusterName -NodeAndFileShareMajority "\\$Domain\ClusterShare"
        
        # Configure cluster network
        Write-Log "Configuring cluster networks"
        $clusterNetworks = Get-ClusterNetwork -Cluster $ClusterName
        foreach ($network in $clusterNetworks) {
            if ($network.Address -like "192.168.1.*") {
                Write-Log "Configuring cluster network $($network.Name) for client traffic"
                (Get-ClusterNetwork -Name $network.Name).Role = "ClusterAndClient"
            }
            else {
                Write-Log "Configuring cluster network $($network.Name) for cluster traffic only"
                (Get-ClusterNetwork -Name $network.Name).Role = "ClusterOnly"
            }
        }
        
        return $true
    }
    catch {
        Write-Log "Failed to configure Windows Cluster: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

function Enable-SQLAlwaysOn {
    param([string]$ServiceAccount)
    
    Write-Log "Enabling SQL Server Always On Availability Groups"
    
    # This requires SQL Server to be installed first
    $sqlServices = Get-Service | Where-Object { $_.Name -like "*SQL*_" }
    
    foreach ($service in $sqlServices) {
        try {
            Write-Log "Enabling Always On for SQL Server instance: $($service.Name)"
            
            # Stop SQL Server service
            Stop-Service -Name $service.Name -Force
            Start-Sleep -Seconds 5
            
            # Enable Always On (requires running as service account or admin)
            $sqlCmd = "EXEC sp_configure 'show advanced options', 1; RECONFIGURE; EXEC sp_configure 'hadrenabled', 1; RECONFIGURE;"
            
            # This would be executed using sqlcmd or SQL PowerShell
            # For this example, we'll use the SQL Server module
            Invoke-Sqlcmd -ServerInstance "localhost" -Query $sqlCmd -ErrorAction Stop
            
            Write-Log "Always On enabled for $($service.Name)"
            
            # Start SQL Server service
            Start-Service -Name $service.Name
            Start-Sleep -Seconds 10
            
            # Verify Always On is enabled
            $alwaysOnStatus = Invoke-Sqlcmd -ServerInstance "localhost" -Query "SELECT SERVERPROPERTY('IsHadrEnabled') AS IsAlwaysOnEnabled"
            if ($alwaysOnStatus.IsAlwaysOnEnabled -eq 1) {
                Write-Log "Always On is successfully enabled for $($service.Name)"
            }
            else {
                Write-Log "Always On verification failed for $($service.Name)" 'WARNING'
            }
        }
        catch {
            Write-Log "Failed to enable Always On for $($service.Name): $($_.Exception.Message)" 'ERROR'
            return $false
        }
    }
    
    return $true
}

# Main execution
try {
    if (-not (Test-Administrator)) {
        Write-Log "This script must be run as Administrator" 'ERROR'
        exit 1
    }
    
    Write-Log "Starting Always On Availability Groups prerequisites configuration"
    
    # Test prerequisites
    if ($EnableFeature) {
        if (-not (Enable-NetFrameworkFeature)) {
            throw ".NET Framework 3.5 enablement failed"
        }
        
        if (-not (Enable-FailoverClustering)) {
            throw "Failover Clustering feature installation failed"
        }
    }
    
    if ($ConfigureCluster) {
        if (-not (Test-ClusterConnectivity -Nodes $ClusterNodes)) {
            throw "Cluster node connectivity test failed"
        }
        
        if (-not (Configure-WindowsCluster -ClusterName $ClusterName -Nodes $ClusterNodes -Domain $Domain)) {
            throw "Windows Cluster configuration failed"
        }
    }
    
    if ($EnableAlwaysOn) {
        if (-not (Enable-SQLAlwaysOn -ServiceAccount $ServiceAccount)) {
            throw "SQL Server Always On enablement failed"
        }
    }
    
    Write-Log "Always On prerequisites configuration completed successfully"
}
catch {
    Write-Log "Always On prerequisites configuration failed: $($_.Exception.Message)" 'ERROR'
    exit 1
}
```

### Creating Always On Availability Groups

**Step-by-step AG Creation:**
```sql
-- Step 1: Create endpoints on all replicas (run on each server)
USE master;
GO

-- Create endpoints on primary replica
IF NOT EXISTS (SELECT * FROM sys.endpoints WHERE name = 'AlwaysOn_Endpoint')
BEGIN
    CREATE ENDPOINT [AlwaysOn_Endpoint]
    STATE=STARTED
    AS TCP (LISTENER_PORT = 5022, LISTENER_IP = ALL)
    FOR DATA_MIRRORING (ROLE = ALL, AUTHENTICATION = WINDOWS NEGOTIATE,
        ENCRYPTION = REQUIRED ALGORITHM AES)
END;
GO

-- Grant permissions on endpoints
GRANT CONNECT ON ENDPOINT::[AlwaysOn_Endpoint] TO [DOMAIN\SQLService];
GO

-- Step 2: Create availability group on primary replica
USE master;
GO

-- Create availability group
CREATE AVAILABILITY GROUP [SalesAG]
WITH (
    AUTOMATED_BACKUP_PREFERENCE = PRIMARY,
    DB_FAILOVER = ON,
    DTC_SUPPORT = NONE
)
FOR DATABASE [SalesDB], [CustomerDB], [InventoryDB]
REPLICA ON 
    N'SQLNODE1' WITH (
        ENDPOINT_URL = N'TCP://SQLNODE1.company.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        BACKUP_PRIORITY = 30,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL),
        SESSION_TIMEOUT = 10
    ),
    N'SQLNODE2' WITH (
        ENDPOINT_URL = N'TCP://SQLNODE2.company.com:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        BACKUP_PRIORITY = 70,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY),
        SESSION_TIMEOUT = 10
    ),
    N'SQLNODE3' WITH (
        ENDPOINT_URL = N'TCP://SQLNODE3.company.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        BACKUP_PRIORITY = 20,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL),
        SESSION_TIMEOUT = 10
    );
GO

-- Step 3: Add listener
ALTER AVAILABILITY GROUP [SalesAG]
ADD LISTENER N'SalesAG-Listener' (
    WITH IP
    ((N'192.168.1.100', N'255.255.255.0')),
    PORT=1433
);
GO

-- Step 4: Join secondary replicas (run on each secondary replica)
-- On SQLNODE2 and SQLNODE3:
USE master;
GO

-- Join secondary replica to availability group
ALTER AVAILABILITY GROUP [SalesAG] JOIN;
GO

-- Join database to availability group
ALTER DATABASE [SalesDB] SET HADR AVAILABILITY GROUP = [SalesAG];
GO

-- Repeat for CustomerDB and InventoryDB

-- Step 5: Configure read-only routing
-- On primary replica
USE master;
GO

-- Configure read-only routing
ALTER AVAILABILITY GROUP [SalesAG]
MODIFY REPLICA ON 
    N'SQLNODE2' WITH (
        SECONDARY_ROLE (READ_ONLY_ROUTING_URL = N'TCP://SQLNODE2.company.com:1433')
    );

ALTER AVAILABILITY GROUP [SalesAG]
MODIFY REPLICA ON 
    N'SQLNODE3' WITH (
        SECONDARY_ROLE (READ_ONLY_ROUTING_URL = N'TCP://SQLNODE3.company.com:1433')
    );

-- Create read-only routing list
ALTER AVAILABILITY GROUP [SalesAG]
MODIFY REPLICA ON 
    N'SQLNODE1' WITH (
        PRIMARY_ROLE (READ_ONLY_ROUTING_LIST = (N'SQLNODE2', N'SQLNODE3'))
    );
GO

-- Step 6: Verify AG configuration
SELECT 
    ag.name AS AvailabilityGroupName,
    ag.group_id,
    ar.replica_server_name AS ReplicaName,
    ar.endpoint_url AS EndpointURL,
    ar.availability_mode_desc AS AvailabilityMode,
    ar.failover_mode_desc AS FailoverMode,
    ar.primary_role_allow_connections_desc AS PrimaryConnections,
    ar.secondary_role_allow_connections_desc AS SecondaryConnections,
    ars.role_desc AS CurrentRole,
    ars.connected_state_desc AS ConnectionState,
    ars.synchronization_state_desc AS SynchronizationState
FROM sys.availability_groups ag
INNER JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
ORDER BY ag.name, ar.role DESC;
```

### Backup Strategy for Always On

**Configuring Backup Preferences:**
```sql
-- Configure backup preferences for the availability group
ALTER AVAILABILITY GROUP [SalesAG]
SET (
    AUTOMATED_BACKUP_PREFERENCE = SECONDARY
);
GO

-- Configure backup priority for each replica
ALTER AVAILABILITY GROUP [SalesAG]
MODIFY REPLICA ON 
    N'SQLNODE2' WITH (BACKUP_PRIORITY = 70);

ALTER AVAILABILITY GROUP [SalesAG]
MODIFY REPLICA ON 
    N'SQLNODE3' WITH (BACKUP_PRIORITY = 50);
GO

-- Create backup jobs that respect AG preferences
-- SQL Server Agent Job: AG Backup Job
USE msdb;
GO

-- Create job category
EXEC sp_addcategory 
    @name = N'Always On Backup',
    @type = N'LOCAL';
GO

-- Create backup job
DECLARE @jobId BINARY(16);
EXEC sp_add_job
    @job_name = N'AG Automated Backup',
    @enabled = 1,
    @description = N'Automated backup job that respects AG preferences',
    @category_name = N'Always On Backup',
    @owner_login_name = N'sa',
    @job_id = @jobId OUTPUT;

-- Add job step
EXEC sp_add_jobstep
    @job_id = @jobId,
    @step_name = N'Full Backup',
    @step_id = 1,
    @subsystem = N'CmdExec',
    @command = N'sqlcmd -S $(ESCAPE_SQUOTE(SRVR)) -E -Q "EXEC sp_BackupAlwaysOnDatabases @BackupType = ''FULL'', @PreferredReplica = 1" -b',
    @on_success_action = 1,
    @retry_attempts = 3,
    @retry_interval = 5;

-- Schedule job
EXEC sp_add_schedule
    @schedule_name = N'Weekly Full Backup',
    @enabled = 1,
    @freq_type = 8, -- Weekly
    @freq_interval = 1, -- Sunday
    @freq_recurrence_factor = 1,
    @active_start_time = 20000; -- 2:00 AM

EXEC sp_attach_schedule
    @job_name = N'AG Automated Backup',
    @schedule_name = N'Weekly Full Backup';

-- Assign job to server
EXEC sp_add_jobserver
    @job_name = N'AG Automated Backup',
    @server_name = N'(LOCAL)';
GO

-- Create procedure to perform AG-aware backups
CREATE OR ALTER PROCEDURE sp_BackupAlwaysOnDatabases
    @BackupType VARCHAR(10) = 'FULL', -- FULL, DIFF, LOG
    @PreferredReplica BIT = 0, -- 1 = use preferred backup replica
    @BackupPath NVARCHAR(500) = 'C:\SQLBackups\',
    @Compress BIT = 1,
    @Verify BIT = 1
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentReplica NVARCHAR(128) = @@SERVERNAME;
    DECLARE @PreferredBackupReplica NVARCHAR(128);
    DECLARE @BackupAvailable BIT = 0;
    DECLARE @DatabaseName NVARCHAR(128);
    DECLARE @BackupSQL NVARCHAR(MAX);
    DECLARE @BackupFile NVARCHAR(500);
    DECLARE @DateString NVARCHAR(20);
    
    -- Get preferred backup replica if specified
    IF @PreferredReplica = 1
    BEGIN
        SELECT TOP 1 @PreferredBackupReplica = replica_server_name
        FROM sys.availability_replicas ar
        WHERE ar.availability_mode_desc = 'ASYNCHRONOUS_COMMIT'
        ORDER BY ar.backup_priority DESC;
        
        -- If no async replica found, use any secondary
        IF @PreferredBackupReplica IS NULL
        BEGIN
            SELECT TOP 1 @PreferredBackupReplica = replica_server_name
            FROM sys.availability_replicas ar
            WHERE ar.replica_server_name != @CurrentReplica
            ORDER BY ar.backup_priority DESC;
        END
        
        -- If still no secondary replica found, backup on primary
        IF @PreferredBackupReplica IS NULL
        BEGIN
            SELECT TOP 1 @PreferredBackupReplica = replica_server_name
            FROM sys.availability_replicas ar
            WHERE ar.role = 1; -- Primary replica
        END
    END
    ELSE
    BEGIN
        SET @PreferredBackupReplica = @CurrentReplica;
    END
    
    -- Check if this replica can perform backups
    IF @PreferredReplica = 1 AND @PreferredBackupReplica != @CurrentReplica
    BEGIN
        PRINT 'This replica (' + @CurrentReplica + ') is not the preferred backup replica (' + @PreferredBackupReplica + ')';
        PRINT 'Backup will be skipped on this replica.';
        RETURN;
    END
    
    PRINT 'Starting ' + @BackupType + ' backup on replica: ' + @PreferredBackupReplica;
    
    -- Cursor for databases in availability groups
    DECLARE db_cursor CURSOR FOR
    SELECT DISTINCT adc.database_name
    FROM sys.availability_databases_cluster adc
    INNER JOIN sys.availability_groups ag ON adc.group_id = ag.group_id
    INNER JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
    WHERE ar.replica_server_name = @PreferredBackupReplica
    AND ar.role IN (1, 2) -- Primary or secondary
    AND adc.database_name NOT IN ('master', 'msdb', 'model', 'tempdb')
    ORDER BY adc.database_name;
    
    OPEN db_cursor;
    FETCH NEXT FROM db_cursor INTO @DatabaseName;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @DateString = CONVERT(NVARCHAR(20), GETDATE(), 112) + '_' + 
                         REPLACE(CONVERT(NVARCHAR(20), GETDATE(), 108), ':', '');
        
        SET @BackupFile = @BackupPath + @DatabaseName + '_' + @BackupType + '_' + @DateString + 
                         CASE @BackupType
                             WHEN 'LOG' THEN '.trn'
                             WHEN 'DIFF' THEN '.diff'
                             ELSE '.bak'
                         END;
        
        -- Ensure backup directory exists
        EXEC xp_create_subdir @BackupFile;
        
        -- Build backup command
        SET @BackupSQL = 'BACKUP ' + 
            CASE @BackupType
                WHEN 'FULL' THEN 'DATABASE'
                WHEN 'DIFF' THEN 'DATABASE'
                WHEN 'LOG' THEN 'LOG'
            END + ' [' + @DatabaseName + '] ' +
            'TO DISK = ''' + @BackupFile + '''';
        
        IF @Compress = 1
            SET @BackupSQL = @BackupSQL + ' WITH COMPRESSION';
        
        IF @BackupType = 'DIFF'
            SET @BackupSQL = @BackupSQL + ', DIFFERENTIAL';
        
        IF @BackupType = 'LOG'
            SET @BackupSQL = @BackupSQL + ', NO_TRUNCATE';
        
        SET @BackupSQL = @BackupSQL + ', STATS = 10';
        
        BEGIN TRY
            PRINT 'Backing up ' + @DatabaseName + ' to ' + @BackupFile;
            EXEC sp_executesql @BackupSQL;
            
            -- Verify backup if requested
            IF @Verify = 1
            BEGIN
                DECLARE @VerifySQL NVARCHAR(500) = 'RESTORE VERIFYONLY FROM DISK = ''' + @BackupFile + '''';
                EXEC sp_executesql @VerifySQL;
            END
            
            PRINT 'Backup completed successfully for ' + @DatabaseName;
            SET @BackupAvailable = 1;
            
        END TRY
        BEGIN CATCH
            PRINT 'Backup failed for ' + @DatabaseName + ': ' + ERROR_MESSAGE();
        END CATCH
        
        FETCH NEXT FROM db_cursor INTO @DatabaseName;
    END
    
    CLOSE db_cursor;
    DEALLOCATE db_cursor;
    
    IF @BackupAvailable = 1
        PRINT 'AG backup process completed successfully';
    ELSE
        PRINT 'AG backup process completed with no successful backups';
END;
GO

-- Grant execute permission
GRANT EXECUTE ON sp_BackupAlwaysOnDatabases TO [YourDbaUser];
```

## Wednesday - Failover Clustering Implementation

### Windows Failover Clustering Setup

**Cluster Configuration and Management:**
```sql
-- Monitor cluster health and status
SELECT 
    fc.cluster_name,
    fc.state_desc AS ClusterState,
    fc.root_directory AS RootPath,
    fc quorum_type_desc AS QuorumType,
    fc.quorum_state_desc AS QuorumState,
    -- Node information
    fn.node_name AS NodeName,
    fn.state_desc AS NodeState,
    fn.node_weight AS NodeWeight,
    fn.description AS NodeDescription,
    -- Network information
    fnc.address AS NetworkAddress,
    fnc.address_prefix AS AddressPrefix,
    fnc.state_desc AS NetworkState,
    fnc.role_desc AS NetworkRole,
    -- Resource information
    fr.resource_type AS ResourceType,
    fr.resource_name AS ResourceName,
    fr.state_desc AS ResourceState,
    fr.status_information AS ResourceStatus
FROM sys.dm_os_cluster_nodes fn
CROSS JOIN sys.dm_os_cluster fc
LEFT JOIN sys.dm_os_cluster_networks fnc ON fn.node_name = fnc.node_name
LEFT JOIN sys.dm_cluster_resources fr ON fn.node_name = fr.owner_node_name
ORDER BY fn.node_name, fr.resource_type;

-- Monitor SQL Server cluster resource
SELECT 
    cr.name AS ResourceName,
    cr.resource_type AS ResourceType,
    cr.state_desc AS ResourceState,
    cr.status_information AS StatusInfo,
    cr.is_read_only AS IsReadOnly,
    -- SQL Server instance information
    CASE 
        WHEN cr.resource_type = 'SQL Server' THEN SERVERPROPERTY('InstanceName')
        WHEN cr.resource_type = 'SQL Server Database' THEN SERVERPROPERTY('DatabaseName')
        ELSE NULL
    END AS InstanceInfo,
    -- Dependencies
    (SELECT COUNT(*) 
     FROM sys.dm_cluster_resource_dependencies crd
     WHERE crd.resource_id = cr.resource_id AND crd.is_dependent = 1) AS DependentResources
FROM sys.dm_cluster_resources cr
WHERE cr.resource_type IN ('SQL Server', 'SQL Server Database', 'SQL Server Agent')
ORDER BY cr.resource_type, cr.name;
```

#### Cluster Quorum Configuration
```powershell
# Configure cluster quorum settings
# Save as: Configure-ClusterQuorum.ps1

param(
    [Parameter(Mandatory=$true)]
    [string]$ClusterName,
    
    [Parameter(Mandatory=$false)]
    [ValidateSet('NodeMajority', 'NodeAndDiskMajority', 'NodeAndFileShareMajority', 'NodeAndCloudMajority', 'Majority')]
    [string]$QuorumType = 'NodeAndFileShareMajority',
    
    [Parameter(Mandatory=$false)]
    [string]$FileSharePath = "\\DOMAIN\ClusterShare",
    
    [Parameter(Mandatory=$false)]
    [int]$CloudWitnessTimeout = 60
)

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
    Add-Content -Path "C:\Logs\ClusterQuorum_$(Get-Date -Format 'yyyyMMdd').log" -Value $logMessage
}

function Test-ClusterNode {
    param([string]$ClusterName)
    
    try {
        $cluster = Get-Cluster -Name $ClusterName
        if ($cluster) {
            Write-Log "Cluster $ClusterName found and accessible"
            return $true
        }
        else {
            Write-Log "Cluster $ClusterName not found" 'ERROR'
            return $false
        }
    }
    catch {
        Write-Log "Failed to connect to cluster $ClusterName`: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

function Set-ClusterQuorumConfiguration {
    param(
        [string]$ClusterName,
        [string]$QuorumType,
        [string]$FileSharePath,
        [int]$CloudWitnessTimeout
    )
    
    try {
        Write-Log "Configuring cluster quorum for $ClusterName"
        Write-Log "Quorum type: $QuorumType"
        
        switch ($QuorumType) {
            'NodeMajority' {
                Write-Log "Setting node majority quorum"
                Set-ClusterQuorum -Cluster $ClusterName -NodeMajority
            }
            'NodeAndDiskMajority' {
                Write-Log "Setting node and disk majority quorum"
                # This would require a disk witness to be configured first
                Set-ClusterQuorum -Cluster $ClusterName -NodeAndDiskMajority
            }
            'NodeAndFileShareMajority' {
                Write-Log "Setting node and file share majority quorum"
                if (-not (Test-Path $FileSharePath)) {
                    Write-Log "Creating file share witness path: $FileSharePath"
                    New-Item -ItemType Directory -Path $FileSharePath -Force | Out-Null
                    
                    # Create the share
                    New-SmbShare -Name "ClusterShare" -Path $FileSharePath -FullAccess "EVERYONE"
                }
                Set-ClusterQuorum -Cluster $ClusterName -NodeAndFileShareMajority $FileSharePath
            }
            'NodeAndCloudMajority' {
                Write-Log "Setting cloud witness quorum"
                Set-ClusterQuorum -Cluster $ClusterName -NodeAndCloudMajority -CloudWitnessAccount $env:USERDNSDOMAIN -CloudWitnessTimeout $CloudWitnessTimeout
            }
            'Majority' {
                Write-Log "Setting majority quorum"
                Set-ClusterQuorum -Cluster $ClusterName -Majority
            }
        }
        
        Write-Log "Cluster quorum configuration completed successfully"
        return $true
    }
    catch {
        Write-Log "Failed to configure cluster quorum: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

function Get-ClusterQuorumStatus {
    param([string]$ClusterName)
    
    try {
        Write-Log "Getting cluster quorum status"
        
        $cluster = Get-Cluster -Name $ClusterName
        $quorum = Get-ClusterQuorum -Cluster $ClusterName
        
        Write-Log "=== Cluster Quorum Status ==="
        Write-Log "Cluster: $ClusterName"
        Write-Log "Quorum Type: $($quorum.QuorumType)"
        Write-Log "Quorum Resource: $($quorum.QuorumResource)"
        Write-Log "Quorum Node: $($quorum.QuorumNode)"
        Write-Log "Expected Votes: $($quorum.ExpectedVotes)"
        Write-Log "Total Votes: $($quorum.TotalVotes)"
        Write-Log "Node Weight: $($quorum.NodeWeight)"
        
        # Get node information
        $nodes = Get-ClusterNode -Cluster $ClusterName
        Write-Log "=== Cluster Nodes ==="
        foreach ($node in $nodes) {
            Write-Log "Node: $($node.Name), State: $($node.State), Votes: $($node.Votes), Weight: $($node.NodeWeight)"
        }
        
        # Get current cluster health
        $clusterNetwork = Get-ClusterNetwork -Cluster $ClusterName
        Write-Log "=== Cluster Networks ==="
        foreach ($network in $clusterNetwork) {
            Write-Log "Network: $($network.Name), Address: $($network.Address), Role: $($network.Role)"
        }
        
        return $true
    }
    catch {
        Write-Log "Failed to get cluster quorum status: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

# Main execution
try {
    Write-Log "Starting cluster quorum configuration"
    
    # Test cluster connectivity
    if (-not (Test-ClusterNode -ClusterName $ClusterName)) {
        throw "Cluster $ClusterName is not accessible"
    }
    
    # Configure quorum
    if (-not (Set-ClusterQuorumConfiguration -ClusterName $ClusterName -QuorumType $QuorumType -FileSharePath $FileSharePath -CloudWitnessTimeout $CloudWitnessTimeout)) {
        throw "Failed to configure cluster quorum"
    }
    
    # Get status
    if (-not (Get-ClusterQuorumStatus -ClusterName $ClusterName)) {
        Write-Log "Failed to retrieve quorum status" 'WARNING'
    }
    
    Write-Log "Cluster quorum configuration completed successfully"
}
catch {
    Write-Log "Cluster quorum configuration failed: $($_.Exception.Message)" 'ERROR'
    exit 1
}
```

### SQL Server Failover Cluster Instance (FCI)

**FCI Installation and Configuration:**
```sql
-- Monitor failover cluster instance health
SELECT 
    fc.cluster_name,
    fc.state_desc AS ClusterState,
    -- SQL Server FCI information
    SERVERPROPERTY('InstanceName') AS InstanceName,
    SERVERPROPERTY('ProductVersion') AS SQLVersion,
    SERVERPROPERTY('Clustered') AS IsClustered,
    -- Node information
    (SELECT COUNT(*) FROM sys.dm_os_cluster_nodes WHERE state_desc = 'UP') AS ActiveNodes,
    (SELECT COUNT(*) FROM sys.dm_os_cluster_nodes WHERE state_desc = 'DOWN') AS DownNodes,
    -- Resource health
    (SELECT COUNT(*) FROM sys.dm_cluster_resources WHERE state_desc = 'Online') AS OnlineResources,
    (SELECT COUNT(*) FROM sys.dm_cluster_resources WHERE state_desc = 'Offline') AS OfflineResources,
    (SELECT COUNT(*) FROM sys.dm_cluster_resources WHERE state_desc = 'Failed') AS FailedResources,
    -- Last failover information
    (SELECT MAX(failover_time) FROM sys.dm_os_cluster_failover_clusters) AS LastFailoverTime,
    -- Quorum information
    fc.quorum_type_desc AS QuorumType,
    fc.quorum_state_desc AS QuorumState;

-- Monitor cluster resource dependencies
SELECT 
    cr.resource_type AS ResourceType,
    cr.resource_name AS ResourceName,
    cr.state_desc AS ResourceState,
    cr.owner_node_name AS OwnerNode,
    cr.status_information AS StatusInfo,
    -- Dependencies
    crd.dependent_resource_name AS DependentResource,
    crd.is_dependent AS IsDependentFlag,
    crd.dependency_expression AS DependencyExpression,
    -- Resource characteristics
    cr.characteristics AS Characteristics,
    cr.characteristics_mask AS CharacteristicsMask
FROM sys.dm_cluster_resources cr
LEFT JOIN sys.dm_cluster_resource_dependencies crd ON cr.resource_id = crd.resource_id
ORDER BY cr.resource_type, cr.resource_name;

-- Monitor SQL Server FCI availability
SELECT 
    SERVERPROPERTY('ServerName') AS CurrentServer,
    SERVERPROPERTY('InstanceName') AS InstanceName,
    SERVERPROPERTY('IsClustered') AS IsClustered,
    -- Active node information
    (SELECT TOP 1 node_name 
     FROM sys.dm_os_cluster_nodes 
     WHERE node_name = CAST(SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS NVARCHAR(128))) AS ActiveNode,
    -- All cluster nodes
    STRING_AGG(node_name, ', ') WITHIN GROUP (ORDER BY node_name) AS AllNodes,
    -- Connection string components for clients
    SERVERPROPERTY('MachineName') AS MachineName,
    SERVERPROPERTY('InstanceName') AS InstanceName,
    CASE 
        WHEN SERVERPROPERTY('InstanceName') IS NULL THEN SERVERPROPERTY('MachineName')
        ELSE SERVERPROPERTY('MachineName') + '\' + SERVERPROPERTY('InstanceName')
    END AS FullyQualifiedInstanceName
FROM sys.dm_os_cluster_nodes;
```

#### Cluster Resource Monitoring Procedure
```sql
-- Create comprehensive cluster monitoring procedure
CREATE OR ALTER PROCEDURE sp_MonitorClusterHealth
    @AlertOnFailedResources BIT = 1,
    @AlertOnDownNodes BIT = 1,
    @AlertOnQuorumLoss BIT = 1,
    @EmailRecipients NVARCHAR(MAX) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertLevel NVARCHAR(20);
    DECLARE @IssueCount INT = 0;
    DECLARE @HealthReport NVARCHAR(MAX);
    DECLARE @ClusterName NVARCHAR(128);
    DECLARE @CurrentNode NVARCHAR(128);
    
    -- Get cluster information
    SELECT @ClusterName = cluster_name, 
           @CurrentNode = CAST(SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS NVARCHAR(128))
    FROM sys.dm_os_cluster;
    
    SET @HealthReport = 'SQL Server Failover Cluster Health Report' + CHAR(13) + CHAR(10);
    SET @HealthReport = @HealthReport + 'Generated: ' + CONVERT(VARCHAR(30), GETDATE(), 120) + CHAR(13) + CHAR(10);
    SET @HealthReport = @HealthReport + 'Cluster: ' + @ClusterName + CHAR(13) + CHAR(10);
    SET @HealthReport = @HealthReport + 'Current Node: ' + @CurrentNode + CHAR(13) + CHAR(10) + CHAR(13) + CHAR(10);
    
    -- Check cluster state
    DECLARE @ClusterState NVARCHAR(60);
    SELECT @ClusterState = fc.state_desc 
    FROM sys.dm_os_cluster fc;
    
    SET @HealthReport = @HealthReport + '=== Cluster State ===' + CHAR(13) + CHAR(10);
    SET @HealthReport = @HealthReport + 'Cluster State: ' + @ClusterState + CHAR(13) + CHAR(10);
    
    IF @ClusterState != 'Up'
    BEGIN
        SET @AlertLevel = 'CRITICAL';
        SET @IssueCount = @IssueCount + 1;
        SET @HealthReport = @HealthReport + 'ISSUE: Cluster is not in UP state' + CHAR(13) + CHAR(10);
    END
    
    -- Check node states
    SET @HealthReport = @HealthReport + CHAR(13) + CHAR(10) + '=== Node Status ===' + CHAR(13) + CHAR(10);
    
    DECLARE @UpNodes INT, @DownNodes INT;
    SELECT @UpNodes = COUNT(*), @DownNodes = COUNT(*)
    FROM sys.dm_os_cluster_nodes
    WHERE state_desc = 'UP'
    OPTION (MAXDOP 1);
    
    SELECT @DownNodes = COUNT(*)
    FROM sys.dm_os_cluster_nodes
    WHERE state_desc = 'DOWN';
    
    SELECT node_name, state_desc, node_weight, description
    FROM sys.dm_os_cluster_nodes;
    
    IF @DownNodes > 0
    BEGIN
        IF @AlertOnDownNodes = 1
        BEGIN
            SET @AlertLevel = CASE WHEN @AlertLevel = 'CRITICAL' THEN 'CRITICAL' ELSE 'WARNING' END;
            SET @IssueCount = @IssueCount + @DownNodes;
            SET @HealthReport = @HealthReport + 'ISSUE: ' + CAST(@DownNodes AS VARCHAR) + ' node(s) are DOWN' + CHAR(13) + CHAR(10);
        END
        ELSE
        BEGIN
            SET @HealthReport = @HealthReport + 'WARNING: ' + CAST(@DownNodes AS VARCHAR) + ' node(s) are DOWN (alert suppressed)' + CHAR(13) + CHAR(10);
        END
    END
    ELSE
    BEGIN
        SET @HealthReport = @HealthReport + 'All nodes are UP (' + CAST(@UpNodes AS VARCHAR) + ' nodes)' + CHAR(13) + CHAR(10);
    END
    
    -- Check resource states
    SET @HealthReport = @HealthReport + CHAR(13) + CHAR(10) + '=== Resource Status ===' + CHAR(13) + CHAR(10);
    
    DECLARE @OnlineResources INT, @OfflineResources INT, @FailedResources INT;
    
    SELECT @OnlineResources = COUNT(*)
    FROM sys.dm_cluster_resources
    WHERE state_desc = 'Online';
    
    SELECT @OfflineResources = COUNT(*)
    FROM sys.dm_cluster_resources
    WHERE state_desc = 'Offline';
    
    SELECT @FailedResources = COUNT(*)
    FROM sys.dm_cluster_resources
    WHERE state_desc = 'Failed';
    
    SELECT resource_type, resource_name, state_desc, owner_node_name
    FROM sys.dm_cluster_resources
    WHERE state_desc != 'Online'
    ORDER BY resource_type, resource_name;
    
    IF @FailedResources > 0
    BEGIN
        IF @AlertOnFailedResources = 1
        BEGIN
            SET @AlertLevel = CASE WHEN @AlertLevel = 'CRITICAL' THEN 'CRITICAL' ELSE 'WARNING' END;
            SET @IssueCount = @IssueCount + @FailedResources;
            SET @HealthReport = @HealthReport + 'ISSUE: ' + CAST(@FailedResources AS VARCHAR) + ' resource(s) are FAILED' + CHAR(13) + CHAR(10);
        END
        ELSE
        BEGIN
            SET @HealthReport = @HealthReport + 'WARNING: ' + CAST(@FailedResources AS VARCHAR) + ' resource(s) are FAILED (alert suppressed)' + CHAR(13) + CHAR(10);
        END
    END
    
    IF @OfflineResources > 0
    BEGIN
        SET @HealthReport = @HealthReport + 'INFO: ' + CAST(@OfflineResources AS VARCHAR) + ' resource(s) are OFFLINE' + CHAR(13) + CHAR(10);
    END
    
    -- Check quorum status
    SET @HealthReport = @HealthReport + CHAR(13) + CHAR(10) + '=== Quorum Status ===' + CHAR(13) + CHAR(10);
    
    DECLARE @QuorumType NVARCHAR(60), @QuorumState NVARCHAR(60);
    
    SELECT @QuorumType = fc.quorum_type_desc, @QuorumState = fc.quorum_state_desc
    FROM sys.dm_os_cluster fc;
    
    SET @HealthReport = @HealthReport + 'Quorum Type: ' + @QuorumType + CHAR(13) + CHAR(10);
    SET @HealthReport = @HealthReport + 'Quorum State: ' + @QuorumState + CHAR(13) + CHAR(10);
    
    IF @QuorumState NOT IN ('Normal', 'Quorum')
    BEGIN
        IF @AlertOnQuorumLoss = 1
        BEGIN
            SET @AlertLevel = 'CRITICAL';
            SET @IssueCount = @IssueCount + 1;
            SET @HealthReport = @HealthReport + 'ISSUE: Quorum is in ' + @QuorumState + ' state' + CHAR(13) + CHAR(10);
        END
        ELSE
        BEGIN
            SET @HealthReport = @HealthReport + 'WARNING: Quorum is in ' + @QuorumState + ' state (alert suppressed)' + CHAR(13) + CHAR(10);
        END
    END
    
    -- Determine overall alert level
    IF @IssueCount = 0
        SET @AlertLevel = 'HEALTHY';
    ELSE IF @AlertLevel IS NULL
        SET @AlertLevel = 'WARNING';
    
    -- Add summary
    SET @HealthReport = @HealthReport + CHAR(13) + CHAR(10) + 
        '=== Health Summary ===' + CHAR(13) + CHAR(10) +
        'Alert Level: ' + @AlertLevel + CHAR(13) + CHAR(10) +
        'Total Issues: ' + CAST(@IssueCount AS VARCHAR) + CHAR(13) + CHAR(10) +
        'Timestamp: ' + CONVERT(VARCHAR(30), GETDATE(), 120);
    
    -- Log results
    INSERT INTO ClusterHealthLog (ClusterName, NodeName, AlertLevel, IssueCount, HealthReport, CheckTime)
    VALUES (@ClusterName, @CurrentNode, @AlertLevel, @IssueCount, @HealthReport, GETDATE());
    
    -- Display results
    PRINT @HealthReport;
    
    -- Send alerts if configured
    IF @IssueCount > 0 AND @EmailRecipients IS NOT NULL
    BEGIN
        PRINT 'ALERT: ' + @AlertLevel + ' level cluster health issues detected';
        -- EXEC msdb.dbo.sp_send_dbmail @recipients = @EmailRecipients, @subject = 'Cluster Health Alert - ' + @AlertLevel, @body = @HealthReport;
    END
    
    RETURN @IssueCount;
END;
GO

-- Create table to log cluster health checks
CREATE TABLE ClusterHealthLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    ClusterName NVARCHAR(128),
    NodeName NVARCHAR(128),
    AlertLevel NVARCHAR(20),
    IssueCount INT,
    HealthReport NVARCHAR(MAX),
    CheckTime DATETIME DEFAULT GETDATE()
);

-- Grant execute permission
GRANT EXECUTE ON sp_MonitorClusterHealth TO [YourDbaUser];
```

## Thursday - Database Replication Configuration

### Transactional Replication Setup

**Replication Architecture and Configuration:**
```sql
-- Configure distribution database
USE master;
GO

-- Enable distribution server
EXEC sp_adddistributor 
    @distributor = @@SERVERNAME,
    @password = 'DistributorPassword123!';
GO

-- Create distribution database
EXEC sp_adddistributiondb 
    @database = N'distribution',
    @data_folder = N'C:\SQLData\Distribution',
    @log_folder = N'C:\SQLLogs\Distribution',
    @min_distretention = 0,
    @max_distretention = 72,
    @history_retention = 48,
    @check_validate = 0;
GO

-- Configure publication database
EXEC sp_replicationdboption 
    @dbname = N'SalesDB',
    @optname = N'publish',
    @value = N'true';
GO

-- Create publication
EXEC sp_addpublication 
    @publication = N'SalesDB_Publication',
    @description = N'Transactional publication of SalesDB',
    @sync_method = N'native',
    @allow_push = N'true',
    @allow_pull = N'true',
    @allow_anonymous = N'true',
    @immediate_sync = N'true',
    @enabled_for_internet = N'false',
    @dynamic_filters = N'false',
    @retention = 14;
GO

-- Add articles to publication
EXEC sp_addarticle 
    @publication = N'SalesDB_Publication',
    @article = N'Orders',
    @source_table = N'Orders',
    @source_owner = N'dbo',
    @destination_table = N'Orders',
    @destination_owner = N'dbo',
    @type = N'logbased',
    @active = N'true',
    @source_object = N'Orders',
    @schema_option = 0x0000000000000000,
    @creation_script = NULL,
    @pre_creation_cmd = N'drop',
    @vertical_partition = N'false';
GO

EXEC sp_addarticle 
    @publication = N'SalesDB_Publication',
    @article = N'OrderDetails',
    @source_table = N'OrderDetails',
    @source_owner = N'dbo',
    @destination_table = N'OrderDetails',
    @destination_owner = N'dbo',
    @type = N'logbased',
    @active = N'true',
    @source_object = N'OrderDetails',
    @schema_option = 0x0000000000000000;
GO

-- Create snapshot agent
EXEC sp_addpublication_snapshot 
    @publication = N'SalesDB_Publication',
    @frequency_type = 1,
    @frequency_interval = 1,
    @frequency_relative_interval = 1,
    @frequency_recurrence_factor = 1,
    @active_start_time = 0,
    @active_end_time = 0,
    @active_start_date = 0,
    @active_end_date = 0,
    @job_security_type = 1;
GO

-- Add push subscriptions
EXEC sp_addpushsubscription_agent 
    @publication = N'SalesDB_Publication',
    @subscriber = N'SUBSCRIBER_SERVER',
    @subscriber_db = N'SalesDB_Replicated',
    @subscriber_security_mode = 0,
    @subscriber_login = N'repl_subscriber',
    @subscriber_password = 'SubscriberPassword123!',
    @job_login = N'DOMAIN\repl_snapshot',
    @job_password = 'SnapshotAgentPassword123!',
    @subscriber_type = 0,
    @agent_type = 1,
    @frequency_type = 64,
    @frequency_interval = 0,
    @frequency_relative_interval = 0,
    @frequency_recurrence_factor = 0,
    @frequency_subday = 0,
    @frequency_subday_interval = 0,
    @active_start_time = 0,
    @active_end_time = 0,
    @active_start_date = 0,
    @active_end_date = 0,
    @alt_snapshot_folder = NULL,
    @working_directory = NULL,
    @use_ftp = 0,
    @ftp_port = 21,
    @ftp_login = N'anonymous',
    @ftp_password = NULL,
    @template_security_mode = 1,
    @conflict_policy = N'pub wins',
    @conflicttable = NULL,
    @auto_update_statistics = N'true',
    @delete_tracking = N'true',
    @allow_subscription_copy = 1;
GO
```

#### Replication Monitoring and Health Check
```sql
-- Monitor replication topology
SELECT 
    p.publication_id,
    p.publisher,
    p.publisher_db,
    p.publication,
    p.publication_type,
    CASE p.publication_type
        WHEN 0 THEN 'Transactional'
        WHEN 1 THEN 'Snapshot'
        WHEN 2 THEN 'Merge'
    END AS PublicationType,
    p.sync_method,
    p.allow_push,
    p.allow_pull,
    p.allow_anonymous,
    p.immediate_sync,
    p.retention,
    p.allow_tran连载 = N'true',
    p.tran连载_always_loopback = N'false',
    p.allow_dts = N'false',
    p.is_preview_allowed,
    p.allow_subscription_copy,
    -- Article count
    (SELECT COUNT(*) FROM dbo.sysarticles a WHERE a.pubid = p.pubid) AS ArticleCount,
    -- Subscriber count
    (SELECT COUNT(*) FROM dbo.syssubscriptions s WHERE s.pubid = p.pubid) AS SubscriberCount
FROM dbo.syspublications p;

-- Monitor replication agents
SELECT 
    j.job_id,
    j.name AS AgentName,
    j.enabled AS IsEnabled,
    j.date_created AS DateCreated,
    j.date_modified AS DateModified,
    -- Agent type
    CASE j.name
        WHEN '%SalesDB_Publication%' AND j.name LIKE '%snapshot%' THEN 'Snapshot Agent'
        WHEN '%SalesDB_Publication%' AND j.name LIKE '%logreader%' THEN 'Log Reader Agent'
        WHEN '%SalesDB_Publication%' AND j.name LIKE '%distribution%' THEN 'Distribution Agent'
        WHEN '%SalesDB_Publication%' AND j.name LIKE '%merge%' THEN 'Merge Agent'
        ELSE 'Unknown Agent'
    END AS AgentType,
    -- Last run information
    ja.run_status AS LastRunStatus,
    ja.run_date AS LastRunDate,
    ja.run_time AS LastRunTime,
    ja.run_duration AS LastRunDuration,
    -- Performance metrics
    CAST(ja.progress AS INT) AS Progress,
    CAST(ja.delivery_rate AS DECIMAL(10,2)) AS DeliveryRate,
    CAST(ja.delivery_rate_unit AS VARCHAR(20)) AS DeliveryRateUnit,
    CAST(ja.delivery_queue AS INT) AS DeliveryQueue
FROM msdb.dbo.sysjobs j
LEFT JOIN msdb.dbo.sysjobactivity ja ON j.job_id = ja.job_id AND ja.session_id = (SELECT MAX(session_id) FROM msdb.dbo.sysjobactivity)
WHERE j.name LIKE '%SalesDB_Publication%' OR j.name LIKE '%replication%'
ORDER BY j.name;

-- Monitor replication latency
SELECT 
    h.publisher,
    h.publisher_db,
    h.publication,
    h.subscriber,
    h.subscriber_db,
    h.agent_type,
    -- Latency measurements
    h.publisher_commit,
    h.distributor_commit,
    h.subscriber_commit,
    -- Calculate latencies in minutes
    DATEDIFF(MINUTE, h.publisher_commit, h.distributor_commit) AS PublisherToDistributorLatencyMinutes,
    DATEDIFF(MINUTE, h.distributor_commit, h.subscriber_commit) AS DistributorToSubscriberLatencyMinutes,
    DATEDIFF(MINUTE, h.publisher_commit, h.subscriber_commit) AS TotalLatencyMinutes,
    -- Command count
    h.command_count,
    h.command_id,
    -- Error information
    h.error_id,
    CASE h.error_id
        WHEN 0 THEN 'Success'
        ELSE 'Error - Check error details'
    END AS ErrorStatus
FROM distribution.dbo.msrepl_originator_lsns nolock
ORDER BY h.publisher_commit DESC
OPTION (MAXDOP 1);

-- Create replication health monitoring procedure
CREATE OR ALTER PROCEDURE sp_MonitorReplicationHealth
    @PublicationName NVARCHAR(128) = NULL,
    @AlertOnLatencyMinutes INT = 30,
    @AlertOnFailedAgents BIT = 1,
    @EmailRecipients NVARCHAR(MAX) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertLevel NVARCHAR(20);
    DECLARE @IssueCount INT = 0;
    DECLARE @HealthReport NVARCHAR(MAX);
    
    SET @HealthReport = 'Replication Health Report' + CHAR(13) + CHAR(10);
    SET @HealthReport = @HealthReport + 'Generated: ' + CONVERT(VARCHAR(30), GETDATE(), 120) + CHAR(13) + CHAR(10);
    SET @HealthReport = @HealthReport + 'Server: ' + @@SERVERNAME + CHAR(13) + CHAR(10) + CHAR(13) + CHAR(10);
    
    -- Check publication status
    SET @HealthReport = @HealthReport + '=== Publication Status ===' + CHAR(13) + CHAR(10);
    
    SELECT 
        p.publication,
        p.publisher_db,
        p.sync_method,
        p.allow_push,
        p.allow_pull,
        p.immediate_sync,
        -- Article information
        (SELECT COUNT(*) FROM dbo.sysarticles a WHERE a.pubid = p.pubid) AS ArticleCount,
        -- Subscription information
        (SELECT COUNT(*) FROM dbo.syssubscriptions s WHERE s.pubid = p.pubid) AS SubscriptionCount,
        -- Replication status
        (SELECT COUNT(*) FROM dbo.syssubscriptions s WHERE s.pubid = p.pubid AND s.status = 2) AS ActiveSubscriptions,
        (SELECT COUNT(*) FROM dbo.syssubscriptions s WHERE s.pubid = p.pubid AND s.status != 2) AS InactiveSubscriptions
    FROM dbo.syspublications p
    WHERE (@PublicationName IS NULL OR p.publication = @PublicationName)
    ORDER BY p.publication;
    
    -- Check agent status
    SET @HealthReport = @HealthReport + CHAR(13) + CHAR(10) + '=== Agent Status ===' + CHAR(13) + CHAR(10);
    
    SELECT 
        j.name AS AgentName,
        CASE j.name
            WHEN '%' + p.publication + '%' AND j.name LIKE '%snapshot%' THEN 'Snapshot Agent'
            WHEN '%' + p.publication + '%' AND j.name LIKE '%logreader%' THEN 'Log Reader Agent'
            WHEN '%' + p.publication + '%' AND j.name LIKE '%distribution%' THEN 'Distribution Agent'
            ELSE 'Unknown Agent'
        END AS AgentType,
        j.enabled AS IsEnabled,
        -- Last run status
        ja.run_status AS LastRunStatus,
        ja.run_date AS LastRunDate,
        ja.run_time AS LastRunTime,
        DATEDIFF(MINUTE, 
                 CONVERT(DATETIME, CAST(ja.run_date AS VARCHAR(8)), 112) + CAST(ja.run_time AS VARCHAR(6)),
                 GETDATE()) AS MinutesSinceLastRun,
        -- Error checking
        CASE ja.run_status
            WHEN 1 THEN 'Succeeded'
            WHEN 2 THEN 'Retrying'
            WHEN 3 THEN 'Canceled'
            WHEN 4 THEN 'In Progress'
            ELSE 'Unknown'
        END AS StatusDescription
    FROM msdb.dbo.sysjobs j
    LEFT JOIN msdb.dbo.sysjobactivity ja ON j.job_id = ja.job_id AND ja.session_id = (SELECT MAX(session_id) FROM msdb.dbo.sysjobactivity)
    CROSS JOIN dbo.syspublications p
    WHERE (j.name LIKE '%' + p.publication + '%' OR j.name LIKE '%replication%')
    AND (@PublicationName IS NULL OR p.publication = @PublicationName)
    ORDER BY j.name;
    
    -- Check replication latency
    SET @HealthReport = @HealthReport + CHAR(13) + CHAR(10) + '=== Replication Latency ===' + CHAR(13) + CHAR(10);
    
    DECLARE @HighLatencyCount INT;
    
    SELECT @HighLatencyCount = COUNT(*)
    FROM distribution.dbo.msrepl_originator_lsns nolock
    WHERE (@PublicationName IS NULL OR publication = @PublicationName)
    AND DATEDIFF(MINUTE, publisher_commit, subscriber_commit) > @AlertOnLatencyMinutes;
    
    SELECT TOP 20
        publisher,
        publisher_db,
        publication,
        subscriber,
        subscriber_db,
        publisher_commit,
        distributor_commit,
        subscriber_commit,
        -- Latency calculations
        DATEDIFF(MINUTE, publisher_commit, distributor_commit) AS PubToDistLatencyMin,
        DATEDIFF(MINUTE, distributor_commit, subscriber_commit) AS DistToSubLatencyMin,
        DATEDIFF(MINUTE, publisher_commit, subscriber_commit) AS TotalLatencyMin,
        -- Status
        CASE 
            WHEN DATEDIFF(MINUTE, publisher_commit, subscriber_commit) > @AlertOnLatencyMinutes THEN 'HIGH_LATENCY'
            ELSE 'NORMAL'
        END AS LatencyStatus
    FROM distribution.dbo.msrepl_originator_lsns nolock
    WHERE (@PublicationName IS NULL OR publication = @PublicationName)
    ORDER BY publisher_commit DESC;
    
    -- Count failed agents
    SELECT @IssueCount = @IssueCount + COUNT(*)
    FROM msdb.dbo.sysjobs j
    LEFT JOIN msdb.dbo.sysjobactivity ja ON j.job_id = ja.job_id AND ja.session_id = (SELECT MAX(session_id) FROM msdb.dbo.sysjobactivity)
    WHERE (j.name LIKE '%replication%' OR j.name LIKE '%' + ISNULL(@PublicationName, '') + '%')
    AND ja.run_status NOT IN (1, 4) -- Not succeeded or in progress
    AND (@AlertOnFailedAgents = 1);
    
    -- Count high latency
    SELECT @IssueCount = @IssueCount + @HighLatencyCount;
    
    -- Determine alert level
    IF @IssueCount = 0
        SET @AlertLevel = 'HEALTHY';
    ELSE IF EXISTS (
        SELECT 1 FROM msdb.dbo.sysjobs j
        LEFT JOIN msdb.dbo.sysjobactivity ja ON j.job_id = ja.job_id AND ja.session_id = (SELECT MAX(session_id) FROM msdb.dbo.sysjobactivity)
        WHERE (j.name LIKE '%replication%' OR j.name LIKE '%' + ISNULL(@PublicationName, '') + '%')
        AND ja.run_status = 3 -- Failed
    )
        SET @AlertLevel = 'CRITICAL';
    ELSE
        SET @AlertLevel = 'WARNING';
    
    -- Add summary
    SET @HealthReport = @HealthReport + CHAR(13) + CHAR(10) + 
        '=== Health Summary ===' + CHAR(13) + CHAR(10) +
        'Alert Level: ' + @AlertLevel + CHAR(13) + CHAR(10) +
        'Total Issues: ' + CAST(@IssueCount AS VARCHAR) + CHAR(13) + CHAR(10) +
        'High Latency Subscriptions: ' + CAST(@HighLatencyCount AS VARCHAR) + CHAR(13) + CHAR(10) +
        'Timestamp: ' + CONVERT(VARCHAR(30), GETDATE(), 120);
    
    -- Log results
    INSERT INTO ReplicationHealthLog (ServerName, PublicationName, AlertLevel, IssueCount, HealthReport, CheckTime)
    VALUES (@@SERVERNAME, @PublicationName, @AlertLevel, @IssueCount, @HealthReport, GETDATE());
    
    -- Display results
    PRINT @HealthReport;
    
    -- Send alerts if configured
    IF @IssueCount > 0 AND @EmailRecipients IS NOT NULL
    BEGIN
        PRINT 'ALERT: ' + @AlertLevel + ' level replication issues detected';
        -- EXEC msdb.dbo.sp_send_dbmail @recipients = @EmailRecipients, @subject = 'Replication Health Alert', @body = @HealthReport;
    END
    
    RETURN @IssueCount;
END;
GO

-- Create table to log replication health checks
CREATE TABLE ReplicationHealthLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    ServerName NVARCHAR(128),
    PublicationName NVARCHAR(128),
    AlertLevel NVARCHAR(20),
    IssueCount INT,
    HealthReport NVARCHAR(MAX),
    CheckTime DATETIME DEFAULT GETDATE()
);

-- Grant execute permission
GRANT EXECUTE ON sp_MonitorReplicationHealth TO [YourDbaUser];
```

### PowerShell Replication Management

```powershell
# Comprehensive Replication Management Script
# Save as: Manage-Replication.ps1

param(
    [Parameter(Mandatory=$true)]
    [string]$PublisherServer,
    
    [Parameter(Mandatory=$true)]
    [string]$DistributorServer,
    
    [Parameter(Mandatory=$true)]
    [string[]]$SubscriberServers,
    
    [Parameter(Mandatory=$true)]
    [string]$PublicationName,
    
    [Parameter(Mandatory=$false)]
    [string]$DatabaseName = 'master',
    
    [Parameter(Mandatory=$false)]
    [switch]$InitializeReplicas,
    [Parameter(Mandatory=$false)]
    [switch]$MonitorHealth,
    [Parameter(Mandatory=$false)]
    [switch]$StopReplicas
)

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
    Add-Content -Path "C:\Logs\ReplicationManagement_$(Get-Date -Format 'yyyyMMdd').log" -Value $logMessage
}

function Initialize-ReplicationTopology {
    param(
        [string]$Publisher,
        [string]$Distributor,
        [string[]]$Subscribers,
        [string]$Publication
    )
    
    Write-Log "Initializing replication topology"
    
    try {
        # Configure distributor (run on distributor server)
        $distributorScript = @"
        -- Enable distribution
        EXEC sp_adddistributor 
            @distributor = '$Publisher',
            @password = 'DistributorPassword123!';
        
        -- Create distribution database
        EXEC sp_adddistributiondb 
            @database = N'distribution',
            @data_folder = N'C:\SQLData\Distribution',
            @log_folder = N'C:\SQLLogs\Distribution',
            @min_distretention = 0,
            @max_distretention = 72,
            @history_retention = 48;
"@
        
        Write-Log "Configuring distributor on $Distributor"
        Invoke-Sqlcmd -ServerInstance $Distributor -Database master -Query $distributorScript
        
        # Configure publisher (run on publisher server)
        $publisherScript = @"
        -- Enable publication database
        EXEC sp_replicationdboption 
            @dbname = N'$DatabaseName',
            @optname = N'publish',
            @value = N'true';
        
        -- Create publication
        EXEC sp_addpublication 
            @publication = N'$Publication',
            @description = N'Transactional publication of $DatabaseName',
            @sync_method = N'native',
            @allow_push = N'true',
            @allow_pull = N'true',
            @allow_anonymous = N'true',
            @immediate_sync = N'true',
            @retention = 14;
"@
        
        Write-Log "Creating publication $Publication on $Publisher"
        Invoke-Sqlcmd -ServerInstance $Publisher -Database master -Query $publisherScript
        
        # Add subscribers
        foreach ($subscriber in $Subscribers) {
            Write-Log "Adding subscriber: $subscriber"
            
            $subscriberScript = @"
            -- Create subscription
            EXEC sp_addsubscription 
                @publication = N'$Publication',
                @subscriber = N'$subscriber',
                @subscriber_db = N'${DatabaseName}_Replicated',
                @subscription_type = N'Push';
            
            -- Create push subscription agent
            EXEC sp_addpushsubscription_agent 
                @publication = N'$Publication',
                @subscriber = N'$subscriber',
                @subscriber_db = N'${DatabaseName}_Replicated',
                @subscriber_security_mode = 0,
                @subscriber_login = N'repl_subscriber',
                @subscriber_password = 'SubscriberPassword123!',
                @frequency_type = 64,
                @active_start_date = 0,
                @active_end_date = 0;
"@
            
            Invoke-Sqlcmd -ServerInstance $Publisher -Database master -Query $subscriberScript
        }
        
        Write-Log "Replication topology initialized successfully"
        return $true
    }
    catch {
        Write-Log "Failed to initialize replication topology: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

function Start-ReplicationAgents {
    param([string]$Publication)
    
    Write-Log "Starting replication agents for publication: $Publication"
    
    try {
        # Start snapshot agent
        $snapshotJob = "repl_snapshot_" + $Publication
        Write-Log "Starting snapshot agent: $snapshotJob"
        Start-SqlAgentJob -ServerInstance $Publisher -JobName $snapshotJob
        
        # Start log reader agent
        $logReaderJob = "repl_logreader_" + $Publication
        Write-Log "Starting log reader agent: $logReaderJob"
        Start-SqlAgentJob -ServerInstance $Publisher -JobName $logReaderJob
        
        # Start distribution agents for each subscriber
        foreach ($subscriber in $SubscriberServers) {
            $distributionJob = "repl_distribution_" + $Publication + "_" + $subscriber
            Write-Log "Starting distribution agent: $distributionJob"
            Start-SqlAgentJob -ServerInstance $Distributor -JobName $distributionJob
        }
        
        Write-Log "All replication agents started successfully"
        return $true
    }
    catch {
        Write-Log "Failed to start replication agents: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

function Stop-ReplicationAgents {
    param([string]$Publication)
    
    Write-Log "Stopping replication agents for publication: $Publication"
    
    try {
        # Stop distribution agents first
        foreach ($subscriber in $SubscriberServers) {
            $distributionJob = "repl_distribution_" + $Publication + "_" + $subscriber
            Write-Log "Stopping distribution agent: $distributionJob"
            Stop-SqlAgentJob -ServerInstance $Distributor -JobName $distributionJob
        }
        
        # Stop log reader agent
        $logReaderJob = "repl_logreader_" + $Publication
        Write-Log "Stopping log reader agent: $logReaderJob"
        Stop-SqlAgentJob -ServerInstance $Publisher -JobName $logReaderJob
        
        # Stop snapshot agent
        $snapshotJob = "repl_snapshot_" + $Publication
        Write-Log "Stopping snapshot agent: $snapshotJob"
        Stop-SqlAgentJob -ServerInstance $Publisher -JobName $snapshotJob
        
        Write-Log "All replication agents stopped successfully"
        return $true
    }
    catch {
        Write-Log "Failed to stop replication agents: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

function Monitor-ReplicationHealth {
    param([string]$Publication)
    
    Write-Log "Monitoring replication health for publication: $Publication"
    
    try {
        # Get publication status
        $publicationQuery = @"
        SELECT 
            publication,
            publisher_db,
            sync_method,
            allow_push,
            allow_pull,
            immediate_sync,
            -- Article information
            (SELECT COUNT(*) FROM dbo.sysarticles a WHERE a.pubid = p.pubid) AS ArticleCount,
            -- Subscription information
            (SELECT COUNT(*) FROM dbo.syssubscriptions s WHERE s.pubid = p.pubid) AS SubscriptionCount
        FROM dbo.syspublications p
        WHERE p.publication = '$Publication'
"@
        
        $publicationStatus = Invoke-Sqlcmd -ServerInstance $PublisherServer -Database master -Query $publicationQuery
        
        Write-Log "=== Publication Status ==="
        foreach ($row in $publicationStatus) {
            Write-Log "Publication: $($row.publication)"
            Write-Log "Database: $($row.publisher_db)"
            Write-Log "Articles: $($row.ArticleCount)"
            Write-Log "Subscriptions: $($row.SubscriptionCount)"
            Write-Log "Sync Method: $($row.sync_method)"
        }
        
        # Check agent status
        $agentQuery = @"
        SELECT 
            j.name AS AgentName,
            j.enabled AS IsEnabled,
            ja.run_status AS LastRunStatus,
            ja.run_date AS LastRunDate,
            ja.run_time AS LastRunTime,
            DATEDIFF(MINUTE, 
                     CONVERT(DATETIME, CAST(ja.run_date AS VARCHAR(8)), 112) + CAST(ja.run_time AS VARCHAR(6)),
                     GETDATE()) AS MinutesSinceLastRun
        FROM msdb.dbo.sysjobs j
        LEFT JOIN msdb.dbo.sysjobactivity ja ON j.job_id = ja.job_id 
            AND ja.session_id = (SELECT MAX(session_id) FROM msdb.dbo.sysjobactivity)
        WHERE j.name LIKE '%$Publication%'
        ORDER BY j.name
"@
        
        $agentStatus = Invoke-Sqlcmd -ServerInstance $PublisherServer -Database msdb -Query $agentQuery
        
        Write-Log "=== Agent Status ==="
        foreach ($row in $agentStatus) {
            $status = switch ($row.LastRunStatus) {
                1 { "Succeeded" }
                2 { "Retrying" }
                3 { "Canceled" }
                4 { "In Progress" }
                default { "Unknown" }
            }
            
            Write-Log "Agent: $($row.AgentName)"
            Write-Log "Enabled: $($row.IsEnabled)"
            Write-Log "Last Status: $status"
            Write-Log "Minutes Since Last Run: $($row.MinutesSinceLastRun)"
            Write-Log "---"
        }
        
        # Check replication latency
        $latencyQuery = @"
        SELECT TOP 10
            publisher,
            publisher_db,
            publication,
            subscriber,
            subscriber_db,
            publisher_commit,
            distributor_commit,
            subscriber_commit,
            DATEDIFF(MINUTE, publisher_commit, subscriber_commit) AS TotalLatencyMin,
            CASE 
                WHEN DATEDIFF(MINUTE, publisher_commit, subscriber_commit) > 30 THEN 'HIGH_LATENCY'
                ELSE 'NORMAL'
            END AS LatencyStatus
        FROM distribution.dbo.msrepl_originator_lsns nolock
        WHERE publication = '$Publication'
        ORDER BY publisher_commit DESC
"@
        
        $latencyData = Invoke-Sqlcmd -ServerInstance $DistributorServer -Database distribution -Query $latencyQuery
        
        Write-Log "=== Replication Latency ==="
        foreach ($row in $latencyData) {
            Write-Log "Publisher: $($row.publisher)"
            Write-Log "Subscriber: $($row.subscriber)"
            Write-Log "Total Latency: $($row.TotalLatencyMin) minutes"
            Write-Log "Status: $($row.LatencyStatus)"
            Write-Log "---"
        }
        
        return $true
    }
    catch {
        Write-Log "Failed to monitor replication health: $($_.Exception.Message)" 'ERROR'
        return $false
    }
}

# Main execution
try {
    Write-Log "Starting replication management for publication: $PublicationName"
    
    if ($InitializeReplicas) {
        if (-not (Initialize-ReplicationTopology -Publisher $PublisherServer -Distributor $DistributorServer -Subscribers $SubscriberServers -Publication $PublicationName)) {
            throw "Failed to initialize replication topology"
        }
    }
    
    if ($MonitorHealth) {
        if (-not (Monitor-ReplicationHealth -Publication $PublicationName)) {
            Write-Log "Replication health monitoring failed" 'WARNING'
        }
    }
    
    if ($StopReplicas) {
        if (-not (Stop-ReplicationAgents -Publication $PublicationName)) {
            throw "Failed to stop replication agents"
        }
    }
    else {
        if (-not (Start-ReplicationAgents -Publication $PublicationName)) {
            throw "Failed to start replication agents"
        }
    }
    
    Write-Log "Replication management completed successfully"
}
catch {
    Write-Log "Replication management failed: $($_.Exception.Message)" 'ERROR'
    exit 1
}
```

## Friday - High Availability Monitoring and Troubleshooting

### Comprehensive HA Monitoring Dashboard

**Always On Dashboard Query:**
```sql
-- Create comprehensive Always On monitoring dashboard
CREATE OR ALTER VIEW vw_AlwaysOnDashboard AS
SELECT 
    -- Availability Group Information
    ag.name AS AvailabilityGroupName,
    ag.group_id,
    ag.failure_condition_level,
    ag.health_check_timeout,
    -- Primary Replica Information
    PrimaryReplica.replica_server_name AS PrimaryServer,
    PrimaryReplica.endpoint_url AS PrimaryEndpoint,
    PrimaryReplica.availability_mode_desc AS PrimaryAvailabilityMode,
    PrimaryReplica.failover_mode_desc AS PrimaryFailoverMode,
    -- Current Replica Information (for this server)
    CurrentReplica.role_desc AS CurrentRole,
    CurrentReplica.connected_state_desc AS CurrentConnectionState,
    CurrentReplica.operational_state_desc AS CurrentOperationalState,
    CurrentReplica.synchronization_health_desc AS CurrentSyncHealth,
    -- Database Information
    adc.database_name AS DatabaseName,
    DatabaseReplica.synchronization_state_desc AS DatabaseSyncState,
    DatabaseReplica.synchronization_health_desc AS DatabaseSyncHealth,
    DatabaseReplica.database_state_desc AS DatabaseState,
    -- Performance Metrics
    DatabaseReplica.log_send_queue_size AS LogSendQueueKB,
    DatabaseReplica.log_send_rate AS LogSendRateKBps,
    DatabaseReplica.redo_queue_size AS RedoQueueKB,
    DatabaseReplica.redo_rate AS RedoRateKBps,
    DatabaseReplica.filestream_send_rate AS FileStreamRateKBps,
    -- Health Indicators
    CASE 
        WHEN CurrentReplica.role = 1 THEN 'PRIMARY_REPLICA'
        WHEN CurrentReplica.role = 2 THEN 'SECONDARY_REPLICA'
        ELSE 'UNKNOWN_ROLE'
    END AS ReplicaRoleCategory,
    CASE 
        WHEN DatabaseReplica.synchronization_state_desc != 'SYNCHRONIZED' THEN 'SYNC_ISSUE'
        WHEN DatabaseReplica.synchronization_health_desc != 'HEALTHY' THEN 'HEALTH_ISSUE'
        WHEN DatabaseReplica.log_send_queue_size > 1048576 THEN 'LARGE_LAG'  -- > 1GB
        WHEN DatabaseReplica.redo_queue_size > 524288 THEN 'LARGE_REDO_LAG'  -- > 512MB
        ELSE 'HEALTHY'
    END AS DatabaseHealthStatus,
    -- Last Activity
    DatabaseReplica.last_commit_time AS LastCommitTime,
    DatabaseReplica.last_send_time AS LastSendTime,
    DatabaseReplica.last_received_time AS LastReceivedTime,
    DatabaseReplica.last_redone_time AS LastRedoneTime
FROM sys.availability_groups ag
INNER JOIN sys.availability_replicas PrimaryReplica ON ag.group_id = PrimaryReplica.group_id AND PrimaryReplica.role = 1
INNER JOIN sys.availability_replicas CurrentReplica ON ag.group_id = CurrentReplica.group_id AND CurrentReplica.replica_server_name = @@SERVERNAME
INNER JOIN sys.availability_databases_cluster adc ON ag.group_id = adc.group_id
INNER JOIN sys.dm_hadr_availability_replica_states DatabaseReplica ON adc.group_id = DatabaseReplica.group_id 
    AND DatabaseReplica.replica_id = CurrentReplica.replica_id
WHERE ag.name IS NOT NULL;

-- Create AG health summary view
CREATE OR ALTER VIEW vw_AGHealthSummary AS
SELECT 
    ag.name AS AvailabilityGroupName,
    -- Primary replica info
    PrimaryReplica.replica_server_name AS PrimaryServer,
    -- Total replicas
    (SELECT COUNT(*) FROM sys.availability_replicas ar WHERE ar.group_id = ag.group_id) AS TotalReplicas,
    (SELECT COUNT(*) FROM sys.availability_replicas ar WHERE ar.group_id = ag.group_id AND ar.role = 1) AS PrimaryReplicas,
    (SELECT COUNT(*) FROM sys.availability_replicas ar WHERE ar.group_id = ag.group_id AND ar.role = 2) AS SecondaryReplicas,
    -- Healthy replicas
    (SELECT COUNT(*) FROM sys.availability_replicas ar
     INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
     WHERE ar.group_id = ag.group_id AND ars.synchronization_health_desc = 'HEALTHY') AS HealthyReplicas,
    -- Database sync status
    (SELECT COUNT(*) FROM sys.availability_databases_cluster adc
     INNER JOIN sys.dm_hadr_availability_replica_states ars ON adc.group_id = ars.group_id
     WHERE adc.group_id = ag.group_id AND ars.synchronization_state_desc = 'SYNCHRONIZED') AS SynchronizedDatabases,
    (SELECT COUNT(*) FROM sys.availability_databases_cluster adc
     INNER JOIN sys.dm_hadr_availability_replica_states ars ON adc.group_id = ars.group_id
     WHERE adc.group_id = ag.group_id AND ars.synchronization_state_desc != 'SYNCHRONIZED') AS NonSynchronizedDatabases,
    -- Performance metrics
    (SELECT AVG(log_send_queue_size) FROM sys.availability_databases_cluster adc
     INNER JOIN sys.dm_hadr_availability_replica_states ars ON adc.group_id = ars.group_id
     WHERE adc.group_id = ag.group_id) AS AvgLogSendQueueKB,
    (SELECT AVG(redo_queue_size) FROM sys.availability_databases_cluster adc
     INNER JOIN sys.dm_hadr_availability_replica_states ars ON adc.group_id = ars.group_id
     WHERE adc.group_id = ag.group_id) AS AvgRedoQueueKB,
    -- Overall health
    CASE 
        WHEN EXISTS (
            SELECT 1 FROM sys.availability_replicas ar
            INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
            WHERE ar.group_id = ag.group_id AND ars.synchronization_health_desc != 'HEALTHY'
        ) THEN 'DEGRADED'
        WHEN EXISTS (
            SELECT 1 FROM sys.availability_databases_cluster adc
            INNER JOIN sys.dm_hadr_availability_replica_states ars ON adc.group_id = ars.group_id
            WHERE adc.group_id = ag.group_id AND ars.synchronization_state_desc != 'SYNCHRONIZED'
        ) THEN 'SYNC_ISSUE'
        WHEN (SELECT AVG(log_send_queue_size) FROM sys.availability_databases_cluster adc
              INNER JOIN sys.dm_hadr_availability_replica_states ars ON adc.group_id = ars.group_id
              WHERE adc.group_id = ag.group_id) > 1048576 THEN 'HIGH_LATENCY'
        ELSE 'HEALTHY'
    END AS OverallHealthStatus
FROM sys.availability_groups ag
INNER JOIN sys.availability_replicas PrimaryReplica ON ag.group_id = PrimaryReplica.group_id AND PrimaryReplica.role = 1;
```

### Troubleshooting Common HA Issues

**Failover Issues Diagnosis:**
```sql
-- Diagnose failover-related issues
SELECT 
    'Failover Issues Analysis' AS AnalysisType,
    -- Check AG state
    ag.name AS AvailabilityGroupName,
    ar.replica_server_name AS ReplicaServer,
    ars.role_desc AS CurrentRole,
    ars.connected_state_desc AS ConnectionState,
    ars.operational_state_desc AS OperationalState,
    ars.synchronization_health_desc AS SyncHealth,
    ars.synchronization_state_desc AS SyncState,
    -- Automatic failover configuration
    ar.failover_mode_desc AS FailoverMode,
    CASE ar.failover_mode_desc
        WHEN 'AUTOMATIC' THEN 'Automatic failover enabled'
        WHEN 'MANUAL' THEN 'Manual failover only'
        WHEN 'EXTERNAL' THEN 'External failover (WSFC)'
        ELSE 'Unknown'
    END AS FailoverConfiguration,
    -- Last failover information
    ars.last_connect_error_description AS LastConnectError,
    ars.last_connect_error_timestamp AS LastConnectErrorTime,
    -- Health check issues
    CASE 
        WHEN ars.operational_state_desc = 'FAILED' THEN 'OPERATIONAL_FAILURE'
        WHEN ars.connected_state_desc = 'DISCONNECTED' THEN 'CONNECTION_ISSUE'
        WHEN ars.synchronization_state_desc NOT IN ('SYNCHRONIZED', 'SYNCHRONIZING') THEN 'SYNCHRONIZATION_ISSUE'
        WHEN ars.synchronization_health_desc != 'HEALTHY' THEN 'HEALTH_CHECK_FAILURE'
        ELSE 'HEALTHY'
    END AS FailoverReadiness,
    -- Recommendations
    CASE 
        WHEN ars.operational_state_desc = 'FAILED' THEN 'Resolve operational failure before attempting failover'
        WHEN ars.connected_state_desc = 'DISCONNECTED' THEN 'Check network connectivity and endpoints'
        WHEN ars.synchronization_state_desc NOT IN ('SYNCHRONIZED', 'SYNCHRONIZING') THEN 'Wait for synchronization before failover'
        WHEN ar.failover_mode_desc = 'MANUAL' THEN 'Manual failover required'
        ELSE 'Ready for automatic failover'
    END AS RecommendedAction
FROM sys.availability_groups ag
INNER JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
ORDER BY ag.name, ar.role DESC;

-- Check for blocking issues preventing failover
SELECT 
    'Blocking Analysis' AS AnalysisType,
    session_id,
    blocking_session_id,
    wait_type,
    wait_resource,
    wait_time,
    DATEDIFF(SECOND, start_time, GETDATE()) AS DurationSeconds,
    -- Query information
    SUBSTRING(st.text, 1, 200) AS QueryText,
    -- AG-specific blocking
    CASE 
        WHEN wait_type LIKE 'HADR%' THEN 'Always On related wait'
        WHEN wait_type IN ('LCK_M_S', 'LCK_M_U', 'LCK_M_X') THEN 'Locking blocking'
        WHEN wait_type IN ('PAGEIOLATCH%') THEN 'I/O latency blocking'
        ELSE 'General blocking'
    END AS BlockingCategory
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE blocking_session_id > 0
OR session_id IN (
    -- Check for AG-related blocking
    SELECT DISTINCT session_id 
    FROM sys.dm_tran_locks 
    WHERE resource_description LIKE '%AvailabilityGroup%'
    OR resource_description LIKE '%Replica%'
)
ORDER BY wait_time DESC;

-- Create troubleshooting procedure for AG issues
CREATE OR ALTER PROCEDURE sp_TroubleshootAGIssues
    @AvailabilityGroupName NVARCHAR(128) = NULL,
    @DetailedAnalysis BIT = 1,
    @GenerateRecommendations BIT = 1
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Report NVARCHAR(MAX);
    DECLARE @IssuesFound BIT = 0;
    DECLARE @CriticalIssues INT = 0;
    DECLARE @WarningIssues INT = 0;
    
    SET @Report = 'Always On Availability Groups Troubleshooting Report' + CHAR(13) + CHAR(10);
    SET @Report = @Report + 'Generated: ' + CONVERT(VARCHAR(30), GETDATE(), 120) + CHAR(13) + CHAR(10);
    SET @Report = @Report + 'Server: ' + @@SERVERNAME + CHAR(13) + CHAR(10);
    
    IF @AvailabilityGroupName IS NOT NULL
        SET @Report = @Report + 'Availability Group: ' + @AvailabilityGroupName + CHAR(13) + CHAR(10);
    
    SET @Report = @Report + CHAR(13) + CHAR(10);
    
    -- 1. Check AG state and configuration
    SELECT @Report = @Report + 
        '=== Availability Group Configuration ===' + CHAR(13) + CHAR(10),
        ag.name AS AGName,
        ar.replica_server_name AS ReplicaName,
        ar.endpoint_url AS EndpointURL,
        ar.availability_mode_desc AS AvailabilityMode,
        ar.failover_mode_desc AS FailoverMode,
        ar.secondary_role_allow_connections_desc AS SecondaryConnections,
        -- Issues identification
        CASE 
            WHEN ar.endpoint_url IS NULL THEN 'MISSING_ENDPOINT'
            WHEN ar.availability_mode_desc IS NULL THEN 'MISSING_CONFIGURATION'
            ELSE 'CONFIGURED'
        END AS ConfigurationStatus
    FROM sys.availability_groups ag
    INNER JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
    WHERE (@AvailabilityGroupName IS NULL OR ag.name = @AvailabilityGroupName)
    ORDER BY ag.name, ar.role DESC;
    
    -- 2. Check synchronization status
    SELECT @Report = @Report + CHAR(13) + CHAR(10) +
        '=== Synchronization Status ===' + CHAR(13) + CHAR(10),
        adc.database_name AS DatabaseName,
        ar.replica_server_name AS ReplicaName,
        ars.synchronization_state_desc AS SyncState,
        ars.synchronization_health_desc AS SyncHealth,
        ars.database_state_desc AS DatabaseState,
        ars.log_send_queue_size AS LogQueueKB,
        ars.redo_queue_size AS RedoQueueKB,
        -- Issues identification
        CASE 
            WHEN ars.database_state_desc != 'ONLINE' THEN 'DATABASE_OFFLINE'
            WHEN ars.synchronization_state_desc != 'SYNCHRONIZED' THEN 'NOT_SYNCHRONIZED'
            WHEN ars.synchronization_health_desc != 'HEALTHY' THEN 'UNHEALTHY'
            WHEN ars.log_send_queue_size > 1048576 THEN 'HIGH_LOG_LAG'  -- > 1GB
            WHEN ars.redo_queue_size > 524288 THEN 'HIGH_REDO_LAG'     -- > 512MB
            ELSE 'HEALTHY'
        END AS DatabaseHealthStatus,
        -- Issue counting
        CASE 
            WHEN ars.database_state_desc != 'ONLINE' THEN 2  -- Critical
            WHEN ars.synchronization_state_desc != 'SYNCHRONIZED' THEN 1 -- Warning
            WHEN ars.synchronization_health_desc != 'HEALTHY' THEN 1      -- Warning
            ELSE 0
        END AS IssueSeverity
    FROM sys.availability_databases_cluster adc
    INNER JOIN sys.availability_replicas ar ON adc.group_id = ar.group_id
    INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
    WHERE (@AvailabilityGroupName IS NULL OR ag.name = @AvailabilityGroupName)
    ORDER BY ar.role DESC, adc.database_name;
    
    -- 3. Check endpoint connectivity
    SELECT @Report = @Report + CHAR(13) + CHAR(10) +
        '=== Endpoint Connectivity ===' + CHAR(13) + CHAR(10),
        ar.replica_server_name AS ReplicaName,
        ar.endpoint_url AS EndpointURL,
        ars.connected_state_desc AS ConnectionState,
        ars.operational_state_desc AS OperationalState,
        -- Connectivity issues
        CASE 
            WHEN ars.connected_state_desc = 'DISCONNECTED' THEN 'CONNECTION_FAILED'
            WHEN ars.operational_state_desc = 'FAILED' THEN 'OPERATION_FAILED'
            WHEN ars.connected_state_desc = 'CONNECTED' AND ars.operational_state_desc = 'ONLINE' THEN 'CONNECTED_OK'
            ELSE 'UNKNOWN_STATE'
        END AS ConnectivityStatus
    FROM sys.availability_replicas ar
    INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
    WHERE (@AvailabilityGroupName IS NULL OR ag.name = @AvailabilityGroupName)
    ORDER BY ar.role DESC;
    
    -- 4. Generate recommendations if requested
    IF @GenerateRecommendations = 1
    BEGIN
        SET @Report = @Report + CHAR(13) + CHAR(10) + 
            '=== Troubleshooting Recommendations ===' + CHAR(13) + CHAR(10);
        
        -- Endpoint issues
        IF EXISTS (
            SELECT 1 FROM sys.availability_replicas ar
            INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
            WHERE (@AvailabilityGroupName IS NULL OR ag.name = @AvailabilityGroupName)
            AND ars.connected_state_desc = 'DISCONNECTED'
        )
        BEGIN
            SET @Report = @Report + 'ENDPOINT CONNECTIVITY ISSUES:' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '1. Verify firewall rules allow traffic on port 5022' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '2. Check that endpoints are started on all replicas' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '3. Verify service accounts have access to endpoints' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '4. Check network connectivity between replica servers' + CHAR(13) + CHAR(10);
            SET @Report = @Report + CHAR(13) + CHAR(10);
            SET @WarningIssues = @WarningIssues + 1;
        END
        
        -- Synchronization issues
        IF EXISTS (
            SELECT 1 FROM sys.availability_databases_cluster adc
            INNER JOIN sys.dm_hadr_availability_replica_states ars ON adc.group_id = ars.group_id
            WHERE (@AvailabilityGroupName IS NULL OR ag.name = @AvailabilityGroupName)
            AND ars.synchronization_state_desc != 'SYNCHRONIZED'
        )
        BEGIN
            SET @Report = @Report + 'SYNCHRONIZATION ISSUES:' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '1. Check for long-running transactions on primary replica' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '2. Monitor log backup frequency and restore on secondary' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '3. Verify adequate network bandwidth between replicas' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '4. Check for blocking issues on secondary replica' + CHAR(13) + CHAR(10);
            SET @Report = @Report + CHAR(13) + CHAR(10);
            SET @WarningIssues = @WarningIssues + 1;
        END
        
        -- Database offline issues
        IF EXISTS (
            SELECT 1 FROM sys.availability_databases_cluster adc
            INNER JOIN sys.dm_hadr_availability_replica_states ars ON adc.group_id = ars.group_id
            WHERE (@AvailabilityGroupName IS NULL OR ag.name = @AvailabilityGroupName)
            AND ars.database_state_desc != 'ONLINE'
        )
        BEGIN
            SET @Report = @Report + 'DATABASE AVAILABILITY ISSUES:' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '1. Check database status on primary replica' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '2. Verify database is properly joined to availability group' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '3. Check for restore operations or recovery pending status' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '4. Review error logs for database-level issues' + CHAR(13) + CHAR(10);
            SET @Report = @Report + CHAR(13) + CHAR(10);
            SET @CriticalIssues = @CriticalIssues + 1;
        END
        
        -- Performance issues
        IF EXISTS (
            SELECT 1 FROM sys.availability_databases_cluster adc
            INNER JOIN sys.dm_hadr_availability_replica_states ars ON adc.group_id = ars.group_id
            WHERE (@AvailabilityGroupName IS NULL OR ag.name = @AvailabilityGroupName)
            AND (ars.log_send_queue_size > 1048576 OR ars.redo_queue_size > 524288)
        )
        BEGIN
            SET @Report = @Report + 'PERFORMANCE ISSUES:' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '1. High log send queue - optimize transaction log size' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '2. High redo queue - check secondary replica performance' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '3. Consider adjusting availability mode for better performance' + CHAR(13) + CHAR(10);
            SET @Report = @Report + '4. Monitor network latency between replicas' + CHAR(13) + CHAR(10);
            SET @Report = @Report + CHAR(13) + CHAR(10);
            SET @WarningIssues = @WarningIssues + 1;
        END
    END
    
    -- Count total issues
    SELECT @CriticalIssues = @CriticalIssues + COUNT(*)
    FROM sys.availability_databases_cluster adc
    INNER JOIN sys.dm_hadr_availability_replica_states ars ON adc.group_id = ars.group_id
    WHERE (@AvailabilityGroupName IS NULL OR ag.name = @AvailabilityGroupName)
    AND ars.database_state_desc != 'ONLINE';
    
    SELECT @WarningIssues = @WarningIssues + COUNT(*)
    FROM sys.availability_databases_cluster adc
    INNER JOIN sys.dm_hadr_availability_replica_states ars ON adc.group_id = ars.group_id
    WHERE (@AvailabilityGroupName IS NULL OR ag.name = @AvailabilityGroupName)
    AND (ars.synchronization_state_desc != 'SYNCHRONIZED' 
         OR ars.synchronization_health_desc != 'HEALTHY'
         OR ars.connected_state_desc = 'DISCONNECTED');
    
    -- Set issues found flag
    SET @IssuesFound = CASE WHEN @CriticalIssues > 0 OR @WarningIssues > 0 THEN 1 ELSE 0 END;
    
    -- Add summary
    SET @Report = @Report + '=== Summary ===' + CHAR(13) + CHAR(10) +
        'Critical Issues: ' + CAST(@CriticalIssues AS VARCHAR) + CHAR(13) + CHAR(10) +
        'Warning Issues: ' + CAST(@WarningIssues AS VARCHAR) + CHAR(13) + CHAR(10) +
        'Total Issues: ' + CAST(@CriticalIssues + @WarningIssues AS VARCHAR) + CHAR(13) + CHAR(10) +
        'Timestamp: ' + CONVERT(VARCHAR(30), GETDATE(), 120);
    
    -- Log results
    INSERT INTO AGTroubleshootingLog (ServerName, AGName, IssuesFound, CriticalIssues, WarningIssues, AnalysisReport, AnalysisTime)
    VALUES (@@SERVERNAME, @AvailabilityGroupName, @IssuesFound, @CriticalIssues, @WarningIssues, @Report, GETDATE());
    
    -- Display results
    PRINT @Report;
    
    RETURN @CriticalIssues + @WarningIssues;
END;
GO

-- Create table to log troubleshooting sessions
CREATE TABLE AGTroubleshootingLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    ServerName NVARCHAR(128),
    AGName NVARCHAR(128),
    IssuesFound BIT,
    CriticalIssues INT,
    WarningIssues INT,
    AnalysisReport NVARCHAR(MAX),
    AnalysisTime DATETIME DEFAULT GETDATE()
);

-- Grant execute permission
GRANT EXECUTE ON sp_TroubleshootAGIssues TO [YourDbaUser];
```

### PowerShell HA Monitoring Dashboard

```powershell
# Comprehensive High Availability Monitoring Dashboard
# Save as: Monitor-HighAvailability.ps1

param(
    [Parameter(Mandatory=$true)]
    [string]$ServerInstance,
    
    [Parameter(Mandatory=$false)]
    [string[]]$AvailabilityGroups,
    
    [Parameter(Mandatory=$false)]
    [string]$DashboardPath = 'C:\Reports\HighAvailability',
    
    [Parameter(Mandatory=$false)]
    [switch]$GenerateHTMLDashboard,
    
    [Parameter(Mandatory=$false)]
    [switch]$SendEmailAlerts,
    
    [Parameter(Mandatory=$false)]
    [string]$AlertRecipients = 'dba@company.com',
    
    [Parameter(Mandatory=$false)]
    [decimal]$LatencyAlertThresholdMB = 1024
)

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logMessage = "[$timestamp] [$Level] $Message"
    Write-Host $logMessage
    
    $logPath = "$DashboardPath\HighAvailability_$(Get-Date -Format 'yyyyMMdd').log"
    if (-not (Test-Path (Split-Path $logPath))) {
        New-Item -ItemType Directory -Path (Split-Path $logPath) -Force | Out-Null
    }
    Add-Content -Path $logPath -Value $logMessage
}

function Get-AlwaysOnStatus {
    param([string]$Server, [string]$AGName = $null)
    
    $query = @"
    SELECT 
        ag.name AS AvailabilityGroupName,
        ar.replica_server_name AS ReplicaServerName,
        ar.endpoint_url AS EndpointURL,
        ar.availability_mode_desc AS AvailabilityMode,
        ar.failover_mode_desc AS FailoverMode,
        ar.role_desc AS CurrentRole,
        ars.connected_state_desc AS ConnectionState,
        ars.operational_state_desc AS OperationalState,
        ars.synchronization_health_desc AS SynchronizationHealth,
        ars.synchronization_state_desc AS SynchronizationState,
        ars.database_state_desc AS DatabaseState,
        ars.log_send_queue_size AS LogSendQueueKB,
        ars.redo_queue_size AS RedoQueueKB,
        ars.log_send_rate AS LogSendRateKBps,
        ars.redo_rate AS RedoRateKBps,
        ars.last_commit_time AS LastCommitTime,
        ars.last_send_time AS LastSendTime
    FROM sys.availability_groups ag
    INNER JOIN sys.availability_replicas ar ON ag.group_id = ag.group_id
    INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
"@
    
    if ($AGName) {
        $query += " WHERE ag.name = '$AGName'"
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
        Write-Log "Failed to get Always On status: $($_.Exception.Message)" 'ERROR'
        return $null
    }
}

function Get-ClusterStatus {
    param([string]$Server)
    
    $query = @"
    SELECT 
        fc.cluster_name,
        fc.state_desc AS ClusterState,
        fn.node_name AS NodeName,
        fn.state_desc AS NodeState,
        fn.node_weight AS NodeWeight,
        cr.resource_type AS ResourceType,
        cr.resource_name AS ResourceName,
        cr.state_desc AS ResourceState,
        cr.owner_node_name AS OwnerNode,
        fc.quorum_type_desc AS QuorumType,
        fc.quorum_state_desc AS QuorumState
    FROM sys.dm_os_cluster fc
    CROSS JOIN sys.dm_os_cluster_nodes fn
    LEFT JOIN sys.dm_cluster_resources cr ON fn.node_name = cr.owner_node_name
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
        Write-Log "Failed to get cluster status: $($_.Exception.Message)" 'ERROR'
        return $null
    }
}

function Test-HAAvailability {
    param([object]$AlwaysOnData, [object]$ClusterData, [decimal]$LatencyThresholdMB)
    
    $healthStatus = @{
        OverallHealth = 'UNKNOWN'
        Issues = @()
        Alerts = @()
        Metrics = @{}
    }
    
    # Check Always On health
    if ($AlwaysOnData) {
        $primaryReplicas = $AlwaysOnData | Where-Object { $_.CurrentRole -eq 'PRIMARY' }
        $secondaryReplicas = $AlwaysOnData | Where-Object { $_.CurrentRole -eq 'SECONDARY' }
        
        # Check for disconnected replicas
        $disconnectedReplicas = $AlwaysOnData | Where-Object { $_.ConnectionState -ne 'CONNECTED' }
        if ($disconnectedReplicas.Count -gt 0) {
            $healthStatus.Alerts += @{
                Level = 'CRITICAL'
                Message = "$($disconnectedReplicas.Count) replica(s) are disconnected"
                Component = 'Always On Connectivity'
            }
        }
        
        # Check for synchronization issues
        $nonSynchronizedReplicas = $AlwaysOnData | Where-Object { $_.SynchronizationState -ne 'SYNCHRONIZED' }
        if ($nonSynchronizedReplicas.Count -gt 0) {
            $healthStatus.Alerts += @{
                Level = 'WARNING'
                Message = "$($nonSynchronizedReplicas.Count) database(s) are not synchronized"
                Component = 'Always On Synchronization'
            }
        }
        
        # Check for high latency
        $highLatencyReplicas = $AlwaysOnData | Where-Object { $_.LogSendQueueKB -gt ($LatencyThresholdMB * 1024) }
        if ($highLatencyReplicas.Count -gt 0) {
            $healthStatus.Alerts += @{
                Level = 'WARNING'
                Message = "$($highLatencyReplicas.Count) replica(s) have high latency"
                Component = 'Always On Performance'
            }
        }
        
        # Calculate metrics
        $healthStatus.Metrics.AvailabilityGroups = ($AlwaysOnData.AvailabilityGroupName | Sort-Object | Get-Unique).Count
        $healthStatus.Metrics.TotalReplicas = $AlwaysOnData.Count
        $healthStatus.Metrics.PrimaryReplicas = $primaryReplicas.Count
        $healthStatus.Metrics.SecondaryReplicas = $secondaryReplicas.Count
        $healthStatus.Metrics.SynchronizedReplicas = ($AlwaysOnData | Where-Object { $_.SynchronizationState -eq 'SYNCHRONIZED' }).Count
        $healthStatus.Metrics.ConnectedReplicas = ($AlwaysOnData | Where-Object { $_.ConnectionState -eq 'CONNECTED' }).Count
    }
    
    # Check cluster health
    if ($ClusterData) {
        $downNodes = $ClusterData | Where-Object { $_.NodeState -ne 'UP' }
        if ($downNodes.Count -gt 0) {
            $healthStatus.Alerts += @{
                Level = 'CRITICAL'
                Message = "$($downNodes.Count) cluster node(s) are down"
                Component = 'Windows Cluster'
            }
        }
        
        $failedResources = $ClusterData | Where-Object { $_.ResourceState -eq 'FAILED' }
        if ($failedResources.Count -gt 0) {
            $healthStatus.Alerts += @{
                Level = 'CRITICAL'
                Message = "$($failedResources.Count) cluster resource(s) have failed"
                Component = 'Cluster Resources'
            }
        }
        
        $healthStatus.Metrics.ClusterNodes = ($ClusterData.NodeName | Sort-Object | Get-Unique).Count
        $healthStatus.Metrics.ClusterResources = $ClusterData.Count
    }
    
    # Determine overall health
    if ($healthStatus.Alerts | Where-Object { $_.Level -eq 'CRITICAL' }) {
        $healthStatus.OverallHealth = 'CRITICAL'
    }
    elseif ($healthStatus.Alerts | Where-Object { $_.Level -eq 'WARNING' }) {
        $healthStatus.OverallHealth = 'WARNING'
    }
    else {
        $healthStatus.OverallHealth = 'HEALTHY'
    }
    
    return $healthStatus
}

function New-HADashboard {
    param(
        [object]$AlwaysOnData,
        [object]$ClusterData,
        [object]$HealthStatus,
        [string]$DashboardPath
    )
    
    $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    $dashboardFile = "$DashboardPath\HighAvailability_Dashboard_$timestamp.html"
    
    $htmlContent = @"
    <!DOCTYPE html>
    <html>
    <head>
        <title>High Availability Dashboard</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 20px; }
            .header { background-color: #f0f0f0; padding: 10px; border-bottom: 2px solid #ccc; }
            .critical { background-color: #ffebee; padding: 10px; margin: 10px 0; border-left: 4px solid #f44336; }
            .warning { background-color: #fff3e0; padding: 10px; margin: 10px 0; border-left: 4px solid #ff9800; }
            .success { background-color: #e8f5e8; padding: 10px; margin: 10px 0; border-left: 4px solid #4caf50; }
            .info { background-color: #e3f2fd; padding: 10px; margin: 10px 0; border-left: 4px solid #2196f3; }
            table { border-collapse: collapse; width: 100%; margin: 20px 0; }
            th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
            th { background-color: #f2f2f2; }
            .healthy { color: #4caf50; }
            .warning { color: #ff9800; }
            .critical { color: #f44336; }
            .metrics { display: flex; flex-wrap: wrap; gap: 20px; }
            .metric-box { 
                background-color: #f9f9f9; 
                padding: 15px; 
                border-radius: 5px; 
                border: 1px solid #ddd;
                flex: 1;
                min-width: 200px;
            }
        </style>
    </head>
    <body>
        <div class="header">
            <h1>High Availability Dashboard</h1>
            <p>Generated: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')</p>
            <p>Server: $ServerInstance</p>
            <p>Overall Status: <span class="$($HealthStatus.OverallHealth.ToLower())">$($HealthStatus.OverallHealth)</span></p>
        </div>
        
        <div class="metrics">
            <div class="metric-box">
                <h3>Availability Groups</h3>
                <p><strong>$($HealthStatus.Metrics.AvailabilityGroups)</strong> total</p>
            </div>
            <div class="metric-box">
                <h3>Replicas</h3>
                <p><strong>$($HealthStatus.Metrics.TotalReplicas)</strong> total ($($HealthStatus.Metrics.PrimaryReplicas) primary, $($HealthStatus.Metrics.SecondaryReplicas) secondary)</p>
            </div>
            <div class="metric-box">
                <h3>Synchronization</h3>
                <p><strong>$($HealthStatus.Metrics.SynchronizedReplicas)</strong> synchronized</p>
            </div>
            <div class="metric-box">
                <h3>Connectivity</h3>
                <p><strong>$($HealthStatus.Metrics.ConnectedReplicas)</strong> connected</p>
            </div>
        </div>
    "@
    
    # Add alerts section
    if ($HealthStatus.Alerts.Count -gt 0) {
        $htmlContent += @"
        <div class="$($HealthStatus.OverallHealth.ToLower())">
            <h2>Active Alerts ($($HealthStatus.Alerts.Count))</h2>
            <ul>
        "@
        
        foreach ($alert in $HealthStatus.Alerts) {
            $alertClass = switch ($alert.Level) {
                'CRITICAL' { 'critical' }
                'WARNING' { 'warning' }
                default { 'info' }
            }
            $htmlContent += "<li class='$alertClass'><strong>$($alert.Level):</strong> $($alert.Message) ($($alert.Component))</li>"
        }
        
        $htmlContent += @"
            </ul>
        </div>
        "@
    }
    
    # Add Always On details
    if ($AlwaysOnData -and $AlwaysOnData.Count -gt 0) {
        $htmlContent += @"
        <h2>Always On Availability Groups Details</h2>
        <table>
            <tr>
                <th>AG Name</th>
                <th>Replica Server</th>
                <th>Role</th>
                <th>Sync State</th>
                <th>Connection</th>
                <th>Health</th>
                <th>Log Queue (KB)</th>
                <th>Status</th>
            </tr>
        "@
        
        foreach ($row in $AlwaysOnData) {
            $syncClass = switch ($row.SynchronizationState) {
                'SYNCHRONIZED' { 'healthy' }
                'SYNCHRONIZING' { 'warning' }
                default { 'critical' }
            }
            
            $connClass = switch ($row.ConnectionState) {
                'CONNECTED' { 'healthy' }
                'DISCONNECTED' { 'critical' }
                default { 'warning' }
            }
            
            $status = if ($row.SynchronizationState -eq 'SYNCHRONIZED' -and $row.ConnectionState -eq 'CONNECTED') { 'HEALTHY' } else { 'ISSUES' }
            $statusClass = if ($status -eq 'HEALTHY') { 'healthy' } else { 'critical' }
            
            $htmlContent += @"
            <tr>
                <td>$($row.AvailabilityGroupName)</td>
                <td>$($row.ReplicaServerName)</td>
                <td>$($row.CurrentRole)</td>
                <td class="$syncClass">$($row.SynchronizationState)</td>
                <td class="$connClass">$($row.ConnectionState)</td>
                <td>$($row.SynchronizationHealth)</td>
                <td>$([math]::Round($row.LogSendQueueKB / 1024, 2))</td>
                <td class="$statusClass">$status</td>
            </tr>
        "@
        }
        
        $htmlContent += "</table>"
    }
    
    # Add cluster details
    if ($ClusterData -and $ClusterData.Count -gt 0) {
        $htmlContent += @"
        <h2>Windows Failover Cluster Details</h2>
        <table>
            <tr>
                <th>Cluster Name</th>
                <th>Node Name</th>
                <th>Node State</th>
                <th>Resource Type</th>
                <th>Resource Name</th>
                <th>Resource State</th>
                <th>Owner Node</th>
                <th>Quorum Status</th>
            </tr>
        "@
        
        $clusterNodes = $ClusterData.NodeName | Sort-Object | Get-Unique
        foreach ($node in $clusterNodes) {
            $nodeData = $ClusterData | Where-Object { $_.NodeName -eq $node }
            $nodeStatus = if (($nodeData | Where-Object { $_.NodeState -eq 'UP' }).Count -gt 0) { 'UP' } else { 'DOWN' }
            $nodeClass = if ($nodeStatus -eq 'UP') { 'healthy' } else { 'critical' }
            
            $quorumInfo = $nodeData | Where-Object { $_.QuorumType -ne $null } | Select-Object -First 1
            
            foreach ($resource in $nodeData) {
                $resourceClass = switch ($resource.ResourceState) {
                    'Online' { 'healthy' }
                    'Offline' { 'warning' }
                    'Failed' { 'critical' }
                    default { 'warning' }
                }
                
                $htmlContent += @"
                <tr>
                    <td>$($resource.ClusterName)</td>
                    <td class="$nodeClass">$($resource.NodeName) ($nodeStatus)</td>
                    <td class="$nodeClass">$($resource.NodeState)</td>
                    <td>$($resource.ResourceType)</td>
                    <td>$($resource.ResourceName)</td>
                    <td class="$resourceClass">$($resource.ResourceState)</td>
                    <td>$($resource.OwnerNode)</td>
                    <td>$($quorumInfo.QuorumType) - $($quorumInfo.QuorumState)</td>
                </tr>
            "@
            }
        }
        
        $htmlContent += "</table>"
    }
    
    $htmlContent += @"
    </body>
    </html>
    "@
    
    $htmlContent | Out-File -FilePath $dashboardFile -Encoding UTF8
    return $dashboardFile
}

function Send-HAAlert {
    param([object]$HealthStatus, [string]$Recipients, [string]$Server)
    
    $subject = "High Availability Alert - $($HealthStatus.OverallHealth)"
    $body = @"
    High Availability Alert for $Server
    
    Overall Status: $($HealthStatus.OverallHealth)
    Alert Count: $($HealthStatus.Alerts.Count)
    
    Active Alerts:
    "@
    
    foreach ($alert in $HealthStatus.Alerts) {
        $body += "`n- $($alert.Level): $($alert.Message) ($($alert.Component))"
    }
    
    $body += "`n`nGenerated: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')"
    
    # In real implementation, send email using Send-MailMessage
    Write-Log "Alert email would be sent to: $Recipients"
    Write-Log "Subject: $subject"
    
    return $subject, $body
}

# Main execution
try {
    Write-Log "Starting High Availability monitoring for $ServerInstance"
    
    # Get Always On status
    $alwaysOnData = Get-AlwaysOnStatus -Server $ServerInstance
    
    # Get cluster status
    $clusterData = Get-ClusterStatus -Server $ServerInstance
    
    # Test HA availability
    $healthStatus = Test-HAAvailability -AlwaysOnData $alwaysOnData -ClusterData $clusterData -LatencyThresholdMB $LatencyAlertThresholdMB
    
    # Generate dashboard
    if ($GenerateHTMLDashboard) {
        $dashboardFile = New-HADashboard -AlwaysOnData $alwaysOnData -ClusterData $clusterData -HealthStatus $healthStatus -DashboardPath $DashboardPath
        Write-Log "HA Dashboard generated: $dashboardFile"
    }
    
    # Send alerts if configured
    if ($SendEmailAlerts -and $healthStatus.OverallHealth -ne 'HEALTHY') {
        $email = Send-HAAlert -HealthStatus $healthStatus -Recipients $AlertRecipients -Server $ServerInstance
        # Send email implementation would go here
    }
    
    # Display summary
    Write-Log "=== High Availability Summary ==="
    Write-Log "Overall Health: $($healthStatus.OverallHealth)"
    Write-Log "Total Alerts: $($healthStatus.Alerts.Count)"
    
    if ($healthStatus.Metrics.AvailabilityGroups) {
        Write-Log "Availability Groups: $($healthStatus.Metrics.AvailabilityGroups)"
        Write-Log "Total Replicas: $($healthStatus.Metrics.TotalReplicas)"
        Write-Log "Synchronized Replicas: $($healthStatus.Metrics.SynchronizedReplicas)"
        Write-Log "Connected Replicas: $($healthStatus.Metrics.ConnectedReplicas)"
    }
    
    if ($healthStatus.Metrics.ClusterNodes) {
        Write-Log "Cluster Nodes: $($healthStatus.Metrics.ClusterNodes)"
        Write-Log "Cluster Resources: $($healthStatus.Metrics.ClusterResources)"
    }
    
    # Display alerts
    if ($healthStatus.Alerts.Count -gt 0) {
        Write-Log "=== Active Alerts ===" 'WARNING'
        foreach ($alert in $healthStatus.Alerts) {
            Write-Log "$($alert.Level): $($alert.Message) ($($alert.Component))" $alert.Level
        }
    }
    
    Write-Log "High Availability monitoring completed successfully"
}
catch {
    Write-Log "High Availability monitoring failed: $($_.Exception.Message)" 'ERROR'
    exit 1
}
```

### Hands-On Lab: Complete High Availability Implementation

**Scenario**: Implement a comprehensive high availability solution for a mission-critical e-commerce database system.

**Tasks**:
1. Design and implement Always On Availability Groups with multiple replicas
2. Configure Windows Failover Clustering for additional redundancy
3. Set up database replication for read-scale scenarios
4. Implement automated failover procedures and testing
5. Create comprehensive monitoring and alerting system
6. Develop disaster recovery procedures using HA technologies
7. Perform failover testing and documentation

**Deliverables**:
- Complete HA architecture documentation with diagrams
- Configured Always On Availability Groups with listener
- Windows Failover Cluster configuration
- Database replication setup for read-scale
- Automated monitoring dashboard and alerting system
- Comprehensive failover and disaster recovery procedures
- Testing results and performance metrics

## Summary

This week covered comprehensive SQL Server high availability solutions. You learned to:

- Design and implement Always On Availability Groups for maximum availability
- Configure and manage SQL Server failover clustering instances  
- Set up and maintain database replication for read-scale scenarios
- Plan and execute failover procedures with minimal downtime
- Monitor high availability health and performance metrics
- Troubleshoot common HA configuration issues and failures
- Implement disaster recovery strategies using HA technologies
- Automate HA monitoring and alerting using PowerShell

Mastering these high availability skills will enable you to design and maintain mission-critical database systems that ensure business continuity and minimal downtime in production environments.