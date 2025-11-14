# Week 2: Installation and Configuration

## Learning Objectives

By the end of this week, students will be able to:

1. **Plan SQL Server Installations**: Develop comprehensive installation plans considering hardware requirements, licensing, and organizational needs
2. **Install SQL Server**: Perform successful SQL Server installations using various methods including Setup GUI and command-line installations
3. **Configure Security Settings**: Implement secure authentication modes, configure service accounts, and establish security best practices
4. **Optimize Performance Settings**: Configure memory, CPU, and I/O settings for optimal database performance
5. **Implement High Availability**: Configure clustering, Always On Availability Groups, and other high availability features
6. **Validate Installation**: Perform post-installation validation tests and documentation

## Theoretical Content

### 1. Pre-Installation Planning and Requirements

#### 1.1 Hardware Requirements and Planning

**CPU Requirements**:
- **Minimum**: 1.4 GHz processor (64-bit or 32-bit)
- **Recommended**: 2.0 GHz or faster
- **Enterprise Workloads**: 8+ cores for production environments
- **NUMA Considerations**: NUMA (Non-Uniform Memory Access) topology awareness for large-scale deployments

**Memory Requirements**:
- **Minimum**: 1GB RAM (Express Edition), 2GB RAM (other editions)
- **Recommended**: 4GB or more
- **Production Systems**: 
  - **OLTP Systems**: 25% of database size + 4GB for OS
  - **Data Warehouses**: 50% of database size + 8GB for OS
  - **Mixed Workloads**: Dynamic allocation based on workload patterns

**Storage Requirements**:
- **Disk Types**: SSD for data files, enterprise SATA for archives
- **I/O Requirements**: 
  - **OLTP**: 100-500 IOPS per core
  - **Reporting**: 50-100 IOPS per core
  - **Data Warehousing**: Variable based on query patterns
- **File Placement**: Separate physical disks for data files, log files, and tempdb

**Network Requirements**:
- **Minimum**: 1 Gbps network connectivity
- **Production**: 10 Gbps for high-availability scenarios
- **Remote Admin**: Bandwidth for management operations and backup/restore

#### 1.2 Software Prerequisites and Dependencies

**Operating System Compatibility**:
```powershell
# Check OS compatibility
Get-WmiObject -Class Win32_OperatingSystem | Select-Object Caption, Version, ServicePackMajorVersion

# Check available memory
Get-WmiObject -Class Win32_ComputerSystem | Select-Object TotalPhysicalMemory
```

**Required Components**:
- **.NET Framework 4.6.2 or later**
- **Windows PowerShell 5.0 or later**
- **Microsoft Visual C++ Redistributable**
- **Windows Installer 4.5 or later**

**Optional Components**:
- **SQL Server Management Studio (SSMS)**
- **Azure Data Studio**
- **SQL Server Data Tools (SSDT)**
- **Power BI Desktop** (for reporting integration)

#### 1.3 Installation Scenarios and Architecture Planning

**Standalone Installation**:
- Single SQL Server instance
- Best for: Small applications, development environments
- Cost: Lower licensing costs
- Availability: Single point of failure

**Failover Cluster Installation**:
- Multiple nodes in active-passive configuration
- Best for: Mission-critical applications requiring high availability
- Cost: Higher due to cluster licensing
- Availability: Automatic failover capabilities

**Always On Availability Groups**:
- Multiple replicas with active capabilities
- Best for: Read-scale and disaster recovery scenarios
- Cost: Enterprise edition licensing required
- Availability: Multiple replicas, read-only access to secondaries

**Multi-Instance Installations**:
- Multiple SQL Server instances on single server
- Best for: Development/testing environments
- Cost: Multiple instance licensing
- Availability: Isolated environments

#### 1.4 Service Account Planning

**Local System Account (NT AUTHORITY\SYSTEM)**:
- **Advantages**: Full administrative rights, no password management
- **Disadvantages**: High security risk, not recommended for production
- **Use Case**: Development environments only

**Local User Account**:
- **Advantages**: Better security isolation
- **Disadvantages**: Password management overhead
- **Use Case**: Small organizations with limited infrastructure

**Domain User Account (Recommended)**:
- **Advantages**: Best security practices, centralized management
- **Disadvantages**: Requires Active Directory infrastructure
- **Use Case**: Production environments, enterprise deployments

**Managed Service Account (MSA)**:
- **Advantages**: Password management by Active Directory, no password expiration
- **Disadvantages**: Limited to single machine, no group membership
- **Use Case**: Windows Server 2008 R2 and later

**Group Managed Service Account (gMSA)**:
- **Advantages**: Multiple server support, password management, no password expiration
- **Disadvantages**: Windows Server 2012 or later, PowerShell knowledge required
- **Use Case**: Recommended for production SQL Server installations

### 2. Installation Methods and Best Practices

#### 2.1 Interactive Installation using SQL Server Setup

**Step-by-Step GUI Installation Process**:

1. **Installation Center Launch**
   ```powershell
   # Launch SQL Server Setup from installation media
   Setup.exe
   ```

2. **Installation Type Selection**
   - **New Standalone SQL Server Installation**: Single instance installation
   - **Add Node to SQL Server Failover Cluster**: Cluster node addition
   - **Upgrade from SQL Server**: In-place upgrade installation

3. **Feature Selection**
   ```sql
   -- Core Features Selection
   * SQL Server Database Services
     - Database Engine Services
     - Full-Text and Semantic Extractions for Search
     - Data Quality Services
   
   * Analysis Services
     - Tabular Mode
     - Multidimensional Mode
   
   * Reporting Services
     - Native Mode
     - SharePoint Mode
   
   * Integration Services
     - Default Installation
     - Scale Out Configuration
   
   * Machine Learning Services
     - In-Database Machine Learning
     - Language Extensions
   
   * Shared Features
     - Connectivity Components
     - Documentation Components
     - Management Tools - Complete
     - SQL Client Connectivity SDK
   ```

4. **Instance Configuration**
   ```sql
   -- Named Instance vs Default Instance
   Default Instance:
   - Instance ID: MSSQLSERVER
   - Installation Directory: C:\Program Files\Microsoft SQL Server\
   
   Named Instance:
   - Instance ID: Custom (e.g., SQL2019, PRODDB)
   - Installation Directory: C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQL2019\
   ```

5. **Disk Space Requirements**
   ```sql
   -- Estimated disk space for different configurations
   SQL Server Database Services: 2,514 MB
   SQL Server Analysis Services: 489 MB
   SQL Server Reporting Services: 580 MB
   SQL Server Integration Services: 411 MB
   Machine Learning Services: 129 MB
   Machine Learning Server: 233 MB
   Documentation Components: 277 MB
   Management Tools - Complete: 701 MB
   Connectivity Components: 47 MB
   SQL Client Connectivity SDK: 22 MB
   ```

6. **Server Configuration**
   ```sql
   -- Service Account Configuration
   SQL Server Database Services:
   - Account Name: DOMAIN\SQLService
   - Password: [Complex password]
   - Start Type: Automatic
   
   SQL Server Agent:
   - Account Name: DOMAIN\SQLAgent
   - Password: [Complex password]
   - Start Type: Automatic
   
   SQL Server Analysis Services:
   - Account Name: NT Service\MSOLAP$SQLEXPRESS
   - Start Type: Automatic
   
   SQL Server Reporting Services:
   - Account Name: NT Service\ReportServer$SQLEXPRESS
   - Start Type: Automatic
   ```

7. **Database Engine Configuration**
   ```sql
   -- Authentication Mode
   Windows Authentication Mode (Recommended)
   - Mixed Mode (SQL Server and Windows Authentication)
   - Mixed Mode Password: [Strong password for sa account]
   
   -- Database Administrator Assignment
   Add Current User (Windows Authentication)
   Add Domain Admins group
   Add DBAs group
   Add specific DBA accounts
   
   -- Data Directory Configuration
   Data root directory: D:\MSSQL15.SQL2019\
   User database data directory: D:\MSSQL15.SQL2019\MSSQL\DATA\
   User database log directory: D:\MSSQL15.SQL2019\MSSQL\DATA\
   TempDB directory: D:\MSSQL15.SQL2019\MSSQL\DATA\
   Backup directory: E:\MSSQL15.SQL2019\BACKUP\
   ```

8. **Features Configuration**
   ```sql
   -- Analysis Services Configuration
   Server Mode: Tabular Mode
   Administrator Assignment:
   - Add Current User
   - Add Domain Admins group
   - Data directory: D:\MSSQL15.SQL2019\MSAS13.SQL2019\
   
   -- Reporting Services Configuration
   Installation Mode: Install only (configure later)
   - Native Mode (recommended for standalone)
   - SharePoint Mode (for SharePoint integration)
   ```

#### 2.2 Unattended Installation using Configuration Files

**Configuration File Creation**:
```ini
; ConfigurationFile.ini for SQL Server 2019 Standard Installation
[OPTIONS]

; Required parameters
ACTION="Install"
FEATURES=SQLEngine,SSMS,ADV_SSMS

; Optional parameters
INSTANCENAME="SQL2019"
INSTALLSQLDIR="D:\Program Files\Microsoft SQL Server"
INSTALLSQLDIR="D:\Program Files\Microsoft SQL Server"

; Service account configuration
SQLSVCACCOUNT="DOMAIN\SQLService"
SQLSVCPASSWORD="SecurePassword123!"
AGTSVCACCOUNT="DOMAIN\SQLAgent"
AGTSVCPASSWORD="SecurePassword123!"
SQLSVCINSTANTFILEINIT="True"

; Authentication mode
SECURITYMODE="SQL"
SAPWD="SuperSecurePassword456!"

; Directories
INSTALLSQLDIR="D:\Program Files\Microsoft SQL Server"
INSTALLSQLDIR="D:\Program Files\Microsoft SQL Server"
INSTANCEDIR="D:\Program Files\Microsoft SQL Server"
SQLCOLLATION="SQL_Latin1_General_CP1_CI_AS"

; Database Engine directories
SQLSVCINSTANTFILEINIT="True"
FILESTREAMLEVEL="0"
ENABLERANU="False"
SQLSVCINSTANTFILEINIT="True"

; Reporting Services configuration
RSINSTALLMODE="DefaultNativeMode"
RSSHPINSTALLATIONMODE="DefaultInstall"

; Analysis Services configuration
ASSVCACCOUNT="NT Service\MSOLAP$SQLEXPRESS"
ASSVCSTARTUPTYPE="Automatic"
ASCollation="SQL_Latin1_General_CP1_CI_AS"
ASDATADIR="D:\Program Files\Microsoft SQL Server\MSAS13.SQL2019\OLAP\"
ASLOGDIR="D:\Program Files\Microsoft SQL Server\MSAS13.SQL2019\OLAP\Log\"
ASBACKUPDIR="D:\Program Files\Microsoft SQL Server\MSAS13.SQL2019\OLAP\Backup\"
ASTEMPDIR="D:\Program Files\Microsoft SQL Server\MSAS13.SQL2019\OLAP\Temp\"
ASCONFIGDIR="D:\Program Files\Microsoft SQL Server\MSAS13.SQL2019\OLAP\Config\"

; Management Tools
ADDCURRENTUSERASSQLADMIN="True"
```

**Automated Installation**:
```powershell
# Command line installation using configuration file
.\Setup.exe /ConfigurationFile=ConfigurationFile.ini

# With additional parameters
.\Setup.exe /ConfigurationFile=ConfigurationFile.ini /SQLSVCINSTANTFILEINIT="True" /SkipAutoIntro
```

#### 2.3 Command Line Installation

**Complete Command Line Installation**:
```powershell
# Extract media to temporary location
expand SQLServer2019-*.exe /s:C:\SQLInstall

# Run setup with all parameters
C:\SQLInstall\setup.exe /ACTION="Install" /FEATURES="SQLEngine,SSMS" /INSTANCENAME="SQL2019" /INSTALLSQLDIR="D:\Program Files\Microsoft SQL Server" /SQLSVCACCOUNT="DOMAIN\SQLService" /SQLSVCPASSWORD="SecurePassword123!" /AGTSVCACCOUNT="DOMAIN\SQLAgent" /AGTSVCPASSWORD="SecurePassword123!" /SECURITYMODE="SQL" /SAPWD="SuperSecurePassword456!" /SQLCOLLATION="SQL_Latin1_General_CP1_CI_AS" /SQLSVCINSTANTFILEINIT="True" /IAcceptSQLServerLicenseTerms="True"
```

### 3. Post-Installation Configuration

#### 3.1 Security Configuration

**Authentication Mode Verification**:
```sql
-- Check current authentication mode
EXEC xp_loginconfig 'login mode'

-- Change authentication mode (requires restart)
-- In SSMS: Server Properties → Security → SQL Server and Windows Authentication mode
EXEC xp_sqlagent_notify @op_type = N'login_mode', @job_id = NULL, @raise_error = 1
```

**Service Account Security**:
```sql
-- Verify service account permissions
SELECT service_name, service_account
FROM sys.dm_server_services
WHERE service_type_desc = 'SQL Engine';

-- Check service account privileges
-- Use: secpol.msc → Local Policies → User Rights Assignment
```

**Network Security Configuration**:
```sql
-- Enable TCP/IP protocol
-- Use SQL Server Configuration Manager
-- TCP/IP properties → IP Addresses → IPAll → TCP Port = 1433

-- Configure SQL Server Browser service (for named instances)
-- Ensure service is running and firewall allows UDP port 1434

-- Firewall configuration
netsh advfirewall firewall add rule name="SQL Server (TCP-In)" dir=in action=allow protocol=TCP localport=1433
netsh advfirewall firewall add rule name="SQL Server Browser (UDP-In)" dir=in action=allow protocol=UDP localport=1434
```

#### 3.2 Performance Configuration

**Memory Configuration**:
```sql
-- Check current memory settings
EXEC sp_configure 'min server memory (MB)', 1024
EXEC sp_configure 'max server memory (MB)', 8192
RECONFIGURE;

-- Dynamic memory allocation (recommended for most systems)
EXEC sp_configure 'min server memory (MB)', 0
EXEC sp_configure 'max server memory (MB)', 2147483647
RECONFIGURE;
```

**CPU Configuration**:
```sql
-- Check affinity mask settings
EXEC sp_configure 'affinity mask', 0
EXEC sp_configure 'affinity I/O mask', 0
RECONFIGURE;

-- Check maximum degree of parallelism
EXEC sp_configure 'max degree of parallelism', 0
RECONFIGURE;
```

**I/O Configuration**:
```sql
-- Verify I/O subsystem performance
-- Use disk performance monitoring tools
-- Check tempdb configuration for optimal performance
```

#### 3.3 TempDB Configuration

**TempDB Optimization**:
```sql
-- Check current TempDB configuration
USE tempdb;
GO
SELECT 
    name,
    physical_name,
    size*8/1024 as SizeMB,
    max_size*8/1024 as MaxSizeMB,
    growth*8/1024 as GrowthMB,
    type_desc
FROM sys.database_files;
GO

-- Recommended configuration for TempDB
-- Multiple data files of equal size (typically 8 data files)
-- Auto-growth in MB (not percentage)
-- Separate physical disks if possible
-- Consider data compression
```

### 4. High Availability Configuration Options

#### 4.1 Always On Availability Groups Setup

**Prerequisites**:
- Windows Server Failover Clustering (WSFC)
- SQL Server Enterprise Edition
- Domain-joined servers
- Same SQL Server version and edition

**AG Configuration Process**:
```sql
-- Enable Always On Availability Groups
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'hadr enabled', 1;
RECONFIGURE;
-- Restart SQL Server service

-- Create endpoint for availability groups
CREATE ENDPOINT [AlwaysOn_Endpoint] 
    AS TCP (LISTENER_PORT = 5022)
    FOR DATA_MIRRORING (ROLE = ALL, AUTHENTICATION = WINDOWS NEGOTIATE);
GO
```

**Availability Group Creation**:
```sql
-- Create availability group
CREATE AVAILABILITY GROUP [AG_Production]
WITH (
    DB_FAILOVER = ON,
    DTC_SUPPORT = NONE,
    CLUSTER_TYPE = WSFC,
    AUTOMATED_BACKUP_PREFERENCE = SECONDARY,
    SECONDARY_ROLE(ALLOW_CONNECTIONS = ALL)
)
FOR DATABASE [ProductionDB]
REPLICA ON 
    N'SQLNode1' WITH (
        ENDPOINT_URL = N'TCP://SQLNode1.domain.com:5022',
        PRIMARY_ROLE (ALLOW_CONNECTIONS = ALL),
        SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL),
        BACKUP_PRIORITY = 50,
        FAILOVER_MODE = AUTOMATIC
    ),
    N'SQLNode2' WITH (
        ENDPOINT_URL = N'TCP://SQLNode2.domain.com:5022',
        PRIMARY_ROLE (ALLOW_CONNECTIONS = ALL),
        SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL),
        BACKUP_PRIORITY = 50,
        FAILOVER_MODE = AUTOMATIC
    );
GO

-- Add secondary replica
ALTER AVAILABILITY GROUP [AG_Production] JOIN;
```

#### 4.2 Failover Cluster Configuration

**Cluster Prerequisites**:
- Shared storage (SAN/NAS)
- Quorum configuration
- Network infrastructure
- Domain membership

**Cluster Creation**:
```powershell
# Install failover clustering feature
Install-WindowsFeature -Name Failover-Clustering -IncludeAllSubFeature

# Validate cluster configuration
Test-Cluster -Node SQLNode1, SQLNode2 -Include Storage, Network, SystemConfiguration

# Create cluster
New-Cluster -Name SQLCluster -Node SQLNode1, SQLNode2 -StaticAddress 192.168.1.100

# Configure quorum
Set-ClusterQuorum -Cluster SQLCluster -NodeAndDiskMajority SQLNode1
```

### 5. Installation Validation and Testing

#### 5.1 Post-Installation Validation

**System Verification**:
```sql
-- SQL Server version verification
SELECT @@VERSION;

-- Component verification
SELECT SERVERPROPERTY('ProductVersion') as Version,
       SERVERPROPERTY('Edition') as Edition,
       SERVERPROPERTY('ProductLevel') as ServicePack,
       SERVERPROPERTY('ProductUpdateReference') as Update,
       SERVERPROPERTY('IsClustered') as IsClustered;

-- Service status verification
SELECT 
    service_type_desc,
    service_name,
    service_account,
    process_id,
    status_desc,
    start_up_type_desc
FROM sys.dm_server_services;
```

**Feature Verification**:
```sql
-- Check installed features
SELECT feature_id, feature_name, installation_type, inst_id
FROM sys.dm_db_install_features;

-- Verify specific components
SELECT SERVERPROPERTY('IsFullTextInstalled') as FullTextInstalled,
       SERVERPROPERTY('IsIntegratedSecurityOnly') as IntegratedSecurity,
       SERVERPROPERTY('IsXTPSupported') as XTPInstalled,
       SERVERPROPERTY('IsAdvancedAnalyticsInstalled') as RInstalled;
```

**Performance Testing**:
```sql
-- Basic performance test
SELECT 
    CPU = @@CPU_BUSY,
    IO = @@IO_BUSY,
    Connections = @@CONNECTIONS,
    Transactions = @@TRANCOUNT;

-- Test query execution
DBCC CHECKDB WITH NO_INFOMSGS;

-- Check for configuration issues
SELECT 
    name,
    value_in_use,
    description
FROM sys.configurations
WHERE value_in_use != default_value;
```

#### 5.2 Security Validation

**Authentication Testing**:
```sql
-- Test Windows authentication
SELECT SYSTEM_USER as CurrentUser,
       USER as DatabaseUser,
       CURRENT_USER as CurrentUserContext;

-- Test SQL authentication (if enabled)
-- Use sa account or other SQL login to connect
-- Execute similar queries to verify access
```

**Service Account Testing**:
```sql
-- Test service account permissions
EXEC xp_cmdshell 'whoami';  -- Requires xp_cmdshell to be enabled

-- Check backup directory access
EXEC xp_cmdshell 'dir C:\SQLBackups\';
```

## Practical Exercises

### Exercise 1: Planning SQL Server Installation

**Objective**: Create a comprehensive installation plan for a production SQL Server environment

**Scenario**: You are implementing SQL Server for a medium-sized healthcare organization with the following requirements:

- 100 concurrent users
- 24/7 availability requirement
- HIPAA compliance requirements
- 1TB current database size, 20% annual growth
- Integration with existing Active Directory
- Need for reporting and analysis capabilities
- Budget: $25,000 for software licenses

**Tasks**:

1. **Requirements Analysis**
   - Calculate hardware requirements
   - Determine SQL Server edition needed
   - Identify additional software requirements
   - Plan for compliance requirements

2. **Architecture Design**
   - Single server vs. high availability configuration
   - Service account strategy
   - Network and security requirements
   - Storage layout design

3. **Installation Plan Creation**
   ```sql
   -- Template for installation plan
   -- Project Name: HealthcareDB_SQLServer
   -- Installation Date: [Current Date]
   -- Environment: Production
   
   -- Hardware Specifications:
   CPU: [Based on requirements]
   RAM: [Based on requirements]
   Storage: [Based on requirements]
   
   -- Software Requirements:
   OS: [Compatible OS version]
   SQL Server Edition: [Selected edition]
   Additional Components: [SSMS, SSAS, SSIS, etc.]
   
   -- Configuration Decisions:
   Authentication Mode: [Windows/Mixed]
   Collation: [Healthcare-specific collation]
   Memory Allocation: [Configured amount]
   Service Accounts: [Planned accounts]
   ```

**Deliverable**: Complete installation plan document with justification for all decisions

### Exercise 2: SQL Server Installation

**Objective**: Perform a complete SQL Server installation using different methods

**Requirements**:
- SQL Server 2019 Developer or Express Edition
- Windows Server 2016/2019 or Windows 10
- Administrative privileges
- At least 20GB free disk space

**Tasks**:

1. **Preparation Phase**
   ```powershell
   # Check system requirements
   Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, TotalPhysicalMemory, CsProcessors
   
   # Check available disk space
   Get-WmiObject -Class Win32_LogicalDisk | Where-Object {$_.DriveType -eq 3} | Select-Object DeviceID, Size, FreeSpace
   
   # Verify .NET Framework
   Get-ItemProperty "HKLM:SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full\" | Select-Object Release
   ```

2. **GUI Installation**
   - Download SQL Server 2019 Developer/Express Edition
   - Mount ISO or extract installation files
   - Run Setup.exe
   - Follow installation wizard with custom settings
   - Document each step and configuration decision

3. **Configuration File Installation**
   - Create configuration file based on GUI installation
   - Perform command-line installation using configuration file
   - Compare installation results

4. **Post-Installation Configuration**
   ```sql
   -- Execute post-installation validation
   -- Verify services are running
   -- Configure security settings
   -- Optimize performance settings
   -- Test connectivity from SSMS
   ```

**Deliverable**: Working SQL Server installation with documented configuration and validation results

### Exercise 3: Security Hardening

**Objective**: Implement security best practices for SQL Server installation

**Tasks**:

1. **Authentication Configuration**
   ```sql
   -- Review current authentication mode
   EXEC xp_loginconfig 'login mode'
   
   -- Configure for production environment
   -- Document change requirements and process
   ```

2. **Service Account Security**
   ```powershell
   # Verify service account permissions
   secedit /export /cfg C:\temp\secconfig.inf
   # Review generated file for relevant permissions
   ```

3. **Network Security**
   ```sql
   -- Disable unnecessary protocols
   -- Use SQL Server Configuration Manager to:
   -- * Disable Named Pipes
   -- * Configure TCP/IP settings
   -- * Set appropriate port numbers
   ```

4. **Surface Area Reduction**
   ```sql
   -- Disable unnecessary features
   EXEC sp_configure 'show advanced options', 1;
   RECONFIGURE;
   
   -- Disable xp_cmdshell (if not needed)
   EXEC sp_configure 'xp_cmdshell', 0;
   RECONFIGURE;
   
   -- Disable OLE Automation (if not needed)
   EXEC sp_configure 'Ole Automation Procedures', 0;
   RECONFIGURE;
   ```

**Deliverable**: Security hardening checklist and verification scripts

### Exercise 4: Performance Optimization

**Objective**: Configure SQL Server for optimal performance

**Tasks**:

1. **Memory Configuration**
   ```sql
   -- Calculate recommended memory settings
   -- For 16GB RAM system:
   EXEC sp_configure 'min server memory (MB)', 2048;
   EXEC sp_configure 'max server memory (MB)', 14336; -- Reserve 2GB for OS
   RECONFIGURE;
   ```

2. **TempDB Optimization**
   ```sql
   -- Configure TempDB for optimal performance
   USE master;
   GO
   
   ALTER DATABASE tempdb 
   MODIFY FILE (
       NAME = tempdev,
       SIZE = 1024MB,
       FILEGROWTH = 128MB,
       MAXSIZE = UNLIMITED
   );
   GO
   
   -- For multiple data files (recommended):
   -- * Equal sizing for all data files
   -- * Same growth increments
   -- * Separate physical disks if available
   ```

3. **Database Settings Optimization**
   ```sql
   -- Optimize database settings
   ALTER DATABASE [YourDatabase] SET AUTO_CREATE_STATISTICS ON;
   ALTER DATABASE [YourDatabase] SET AUTO_UPDATE_STATISTICS ON;
   ALTER DATABASE [YourDatabase] SET AUTO_UPDATE_STATISTICS_ASYNC ON;
   ALTER DATABASE [YourDatabase] SET COMPATIBILITY_LEVEL = 150; -- SQL Server 2019
   ```

4. **Performance Monitoring Setup**
   ```sql
   -- Create performance baseline queries
   -- Monitor resource usage
   -- Set up automated monitoring
   ```

**Deliverable**: Performance configuration script with monitoring setup

## Real-World Scenarios

### Scenario 1: Multi-Server SQL Server Deployment - Global Retail Company

**Background**: GlobalRetail Inc. is expanding to 15 countries and needs to deploy SQL Server infrastructure to support their e-commerce platform, inventory management, and reporting systems across multiple regions.

**Current Situation**:
- Existing SQL Server 2016 environment in North American headquarters
- 5 regional data centers being established
- Centralized inventory system supporting online and physical stores
- 500GB database size with 30% annual growth
- Real-time inventory synchronization required
- Compliance requirements: GDPR (Europe), CCPA (California), PCI DSS (payment processing)

**Deployment Requirements**:
- **Performance**: < 200ms response time for inventory queries
- **Availability**: 99.9% uptime, < 4-hour recovery time
- **Scalability**: Support 10x current transaction volume
- **Security**: End-to-end encryption, role-based access
- **Disaster Recovery**: Cross-region failover capability

**Technical Challenges**:
1. **Global Latency Management**
   - Database replication strategy for global distribution
   - Read/Write splitting for geographic distribution
   - Consistent application data across regions

2. **Compliance and Data Sovereignty**
   - GDPR compliance requiring data residency
   - Cross-border data transfer restrictions
   - Regional audit and logging requirements

3. **Infrastructure Standardization**
   - Consistent configuration across all regions
   - Automated deployment and configuration management
   - Centralized monitoring and management

**DBA Tasks**:

1. **Infrastructure Planning**
   ```powershell
   # Regional server specifications calculation
   # Based on expected load and data distribution
   # CPU: 16 cores minimum for each region
   # RAM: 64GB for high-traffic regions, 32GB for low-traffic
   # Storage: SSD for data files, separate storage for backups
   # Network: 10Gbps for inventory synchronization
   ```

2. **Standardized Installation Process**
   ```ini
   ; Standard configuration file for all regional installations
   [OPTIONS]
   ACTION="Install"
   FEATURES="SQLEngine,SSMS,IS,BC"
   INSTANCENAME="GLOBAL"
   SQLSVCACCOUNT="GLOBAL\SQLService"
   SECURITYMODE="SQL"
   SAPWD="RegionalSpecificPassword"
   SQLCOLLATION="SQL_Latin1_General_CP1_CI_AS"
   ```

3. **Always On Availability Groups for High Availability**
   ```sql
   -- Primary replica in each region
   -- Secondary replicas for disaster recovery
   -- Read-scale secondaries for reporting
   
   CREATE AVAILABILITY GROUP [GlobalInventory_AG]
   WITH (
       DB_FAILOVER = ON,
       DTC_SUPPORT = NONE,
       CLUSTER_TYPE = WSFC,
       AUTOMATED_BACKUP_PREFERENCE = SECONDARY
   )
   FOR DATABASE [InventoryDB];
   ```

**Learning Outcome**: Understanding global database deployment challenges and multi-server management strategies

### Scenario 2: Database Migration and Upgrade - Manufacturing Company

**Background**: ManufacturePro is migrating from SQL Server 2012 to SQL Server 2019 while implementing new ERP system. Current environment has 200GB database with legacy applications that need to continue running during migration.

**Current Environment**:
- SQL Server 2012 Standard Edition on Windows Server 2012 R2
- 2-node failover cluster
- Mixed workload: OLTP (60%), Reporting (30%), ETL (10%)
- Database: 200GB with 500+ tables, 1,200+ stored procedures
- Applications: Custom ERP system, legacy reporting tools, custom dashboards
- Maintenance window: 8 hours maximum per week

**Migration Requirements**:
- **Zero Data Loss**: Financial data accuracy is critical
- **Minimal Downtime**: Production operations cannot stop
- **Application Compatibility**: Existing applications must continue working
- **Performance Improvement**: 40% performance improvement required
- **Compliance**: SOX compliance requirements for financial data

**Migration Strategy**:
1. **Assessment and Planning Phase**
   ```sql
   -- Database compatibility assessment
   SELECT 
       OBJECT_NAME(object_id) as ObjectName,
       type_desc as ObjectType,
       create_date,
       modify_date
   FROM sys.objects
   WHERE type_desc IN ('PROCEDURE', 'FUNCTION', 'TRIGGER', 'VIEW')
   ORDER BY type_desc, modify_date DESC;
   
   -- Deprecated feature usage check
   SELECT 
       OBJECT_NAME(object_id) as DeprecatedObject,
       type_desc as ObjectType,
       COALESCE(OBJECT_DEFINITION(object_id), '') as Definition
   FROM sys.objects
   WHERE OBJECT_DEFINITION(object_id) LIKE '%sp_executesql%' -- Example deprecated feature
   OR OBJECT_DEFINITION(object_id) LIKE '%SET ROWCOUNT%';
   ```

2. **Staged Migration Approach**
   - **Phase 1**: Side-by-side installation of SQL Server 2019
   - **Phase 2**: Database migration to new instance
   - **Phase 3**: Application redirection
   - **Phase 4**: Old system decommissioning

3. **Upgrade Process Implementation**
   ```sql
   -- In-place upgrade process
   -- 1. Backup all databases
   -- 2. Run upgrade advisor
   -- 3. Perform upgrade during maintenance window
   -- 4. Verify application functionality
   -- 5. Update statistics and rebuild indexes
   
   -- Post-upgrade optimization
   UPDATE STATISTICS [ManufacturingDB] WITH FULLSCAN;
   ALTER INDEX ALL ON [ManufacturingDB].[dbo].[ProductionOrders] REBUILD WITH (ONLINE = ON);
   ```

**Learning Outcome**: Database migration planning, execution, and validation techniques

### Scenario 3: Cloud Migration - Financial Services Company

**Background**: FirstBank is migrating on-premises SQL Server to Azure SQL Database while maintaining hybrid connectivity for compliance and performance requirements.

**Current On-Premises Environment**:
- SQL Server 2019 Enterprise Edition
- 2-node Always On Availability Group
- Database size: 1TB (300GB in prod, 700GB in data warehouse)
- 24/7 trading platform with sub-second response time requirements
- Regulatory compliance: SOX, PCI DSS, local banking regulations
- Performance monitoring: Continuous monitoring with < 5% performance degradation tolerance

**Azure Migration Strategy**:
- **Phase 1**: Azure SQL Database for production workloads
- **Phase 2**: Azure Synapse for data warehouse
- **Phase 3**: Azure Analysis Services for reporting
- **Phase 4**: Hybrid connectivity for specialized requirements

**Technical Challenges**:
1. **Performance and Latency**
   - Network latency between on-premises and Azure
   - Query performance in cloud environment
   - Data synchronization between hybrid environments

2. **Compliance and Security**
   - Data residency requirements
   - Encryption requirements for data in transit and at rest
   - Audit and monitoring requirements

3. **Application Modification**
   - Connection string changes
   - Feature compatibility (linked servers, extended procedures)
   - Dependency management

**Migration Implementation**:

1. **Connection String Migration**
   ```sql
   -- On-premises connection string format
   -- Server=ONPREMISES-SQL01;Database=TradingDB;Integrated Security=true;
   
   -- Azure SQL Database connection string format
   -- Server=tcp:myserver.database.windows.net,1433;Database=TradingDB;Authentication=Active Directory Password;Encrypt=true;
   ```

2. **Hybrid Configuration**
   ```sql
   -- Azure SQL Managed Instance for hybrid scenarios
   -- Provides near-complete on-premises compatibility
   
   -- Configure hybrid connectivity
   CREATE ENDPOINT [Hadr_endpoint] AS TCP (LISTENER_PORT = 5022)
   FOR DATA_MIRRORING (ROLE = ALL, AUTHENTICATION = WINDOWS NEGOTIATE);
   ```

3. **Performance Optimization for Cloud**
   ```sql
   -- Azure SQL Database specific optimizations
   ALTER DATABASE [TradingDB] SET COMPATIBILITY_LEVEL = 150;
   
   -- Enable automatic tuning
   ALTER DATABASE [TradingDB] SET AUTOMATIC_TUNING (FORCE_LAST_GOOD_PLAN = ON);
   
   -- Configure intelligent query processing
   ALTER DATABASE SCOPED CONFIGURATION SET LEGACY_CARDINALITY_ESTIMATION = OFF;
   ALTER DATABASE SCOPED CONFIGURATION SET BATCH_MODE_MEMORY_GRANT_FEEDBACK = ON;
   ```

**Learning Outcome**: Cloud migration planning, hybrid connectivity, and Azure SQL Database administration

## Summary and Key Takeaways

This week covered comprehensive SQL Server installation and configuration, focusing on:

1. **Pre-Installation Planning**: Proper planning prevents installation failures and performance issues
2. **Installation Methods**: GUI, command-line, and automated installation techniques for different scenarios
3. **Security Configuration**: Authentication modes, service accounts, and security hardening
4. **Performance Optimization**: Memory, CPU, TempDB, and database settings for optimal performance
5. **High Availability**: Always On Availability Groups and failover clustering configuration
6. **Post-Installation Validation**: Comprehensive testing and verification procedures

Key considerations for production installations:
- **Plan for Growth**: Design configurations to accommodate future expansion
- **Security First**: Implement security best practices from the start
- **Monitor Continuously**: Establish monitoring and alerting from day one
- **Document Everything**: Maintain comprehensive documentation for operations and compliance

## Additional Resources

### Microsoft Documentation
- SQL Server Installation Guide: https://docs.microsoft.com/sql/database-engine/install-windows/install-sql-server
- SQL Server Setup Best Practices: https://docs.microsoft.com/sql/database-engine/install-windows/sql-server-setup-best-practices
- SQL Server Security Configuration: https://docs.microsoft.com/sql/sql-server/install/security-considerations-for-a-sql-server-installation

### Tools and Utilities
- Database Experimentation Assistant (DEA): For migration testing
- SQL Server Upgrade Advisor: For compatibility assessment
- SQL Server Assessment: For health checks and best practices

### Scripts and Automation
- SQL Server Installation Scripts: Available in SQL Server samples
- PowerShell SQL Server Modules: SqlServer and SQLServer
- Community scripts from SQL Server Central and Brent Ozar

---

**Next Week Preview**: Database Design Fundamentals - We'll explore normalization principles, indexing strategies, and data modeling techniques for optimal database performance and maintainability.