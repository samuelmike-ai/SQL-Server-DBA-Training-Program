# Week 4: Security and Permissions

## Learning Objectives

By the end of this week, students will be able to:

1. **Understand SQL Server Security Architecture**: Comprehend the layered security model of SQL Server including server, database, and object-level security
2. **Implement Authentication Methods**: Configure Windows Authentication, SQL Server Authentication, and Azure Active Directory authentication
3. **Design Authorization Strategies**: Create effective permission models using roles, users, and schemas
4. **Apply Security Best Practices**: Implement encryption, auditing, and compliance measures
5. **Manage Service Accounts**: Configure and secure service accounts for optimal functionality and security
6. **Monitor Security Events**: Set up auditing, alerting, and incident response procedures

## Theoretical Content

### 1. SQL Server Security Architecture

#### 1.1 Security Layers Overview

SQL Server employs a multi-layered security architecture that provides defense-in-depth protection:

**Network Layer Security**:
- **Protocols**: TCP/IP, Named Pipes, Shared Memory
- **Encryption**: SSL/TLS for network communication
- **Firewalls**: Port filtering and network segmentation

**Authentication Layer**:
- **Server Level**: Server logins and authentication methods
- **Database Level**: Database users mapped to server logins
- **Windows Integration**: Domain accounts and groups

**Authorization Layer**:
- **Server Level**: Server roles and permissions
- **Database Level**: Database roles and ownership chains
- **Object Level**: Specific permissions on tables, views, procedures

**Data Layer Security**:
- **Encryption at Rest**: TDE (Transparent Data Encryption)
- **Encryption in Transit**: SSL/TLS certificates
- **Column Encryption**: Always Encrypted feature
- **Backup Encryption**: Encrypted backup files

#### 1.2 Security Principals

**Server-Level Principals**:
- **Logins**: Identity used to connect to SQL Server
- **Server Roles**: Predefined groups with specific server-level permissions
- **Fixed Server Roles**: SQL Server built-in roles (sysadmin, securityadmin, etc.)

**Database-Level Principals**:
- **Users**: Database principals mapped to logins
- **Database Roles**: User-defined groups within a database
- **Fixed Database Roles**: SQL Server built-in database roles (db_owner, db_datareader, etc.)
- **Application Roles**: Password-protected roles that can be activated programmatically

**Schema-Level Security**:
- **Schemas**: Containers for database objects with associated ownership
- **Schema Permissions**: Permissions granted on schemas apply to all contained objects
- **Ownership Chains**: Permission inheritance through object relationships

#### 1.3 Permission Categories

**Server Permissions**:
```sql
-- View all server-level permissions
SELECT 
    permission_name,
    state_desc,
    class_desc,
    major_id,
    minor_id
FROM sys.server_permissions
ORDER BY permission_name;

-- Check fixed server role members
SELECT 
    r.name AS role_name,
    m.name AS member_name,
    m.type_desc AS member_type
FROM sys.server_role_members srm
INNER JOIN sys.server_principals r ON srm.member_principal_id = r.principal_id
INNER JOIN sys.server_principals m ON srm.member_principal_id = m.principal_id
ORDER BY r.name;
```

**Database Permissions**:
```sql
-- View database permissions
SELECT 
    p.permission_name,
    p.state_desc,
    s.name AS schema_name,
    o.name AS object_name,
    pr.name AS principal_name,
    pr.type_desc AS principal_type
FROM sys.database_permissions p
LEFT JOIN sys.schemas s ON p.major_id = s.schema_id
LEFT JOIN sys.objects o ON p.major_id = o.object_id
LEFT JOIN sys.database_principals pr ON p.grantee_principal_id = pr.principal_id
ORDER BY p.permission_name;
```

### 2. Authentication Methods and Configuration

#### 2.1 Windows Authentication

**Concept**: Uses Windows/Active Directory accounts for authentication
- **Advantages**: Single sign-on, centralized management, enhanced security
- **Process**: Windows validates credentials, SQL Server trusts the authentication
- **Best Practices**: Use domain groups for scalability

**Implementation**:
```sql
-- Create Windows login from domain group
CREATE LOGIN [DOMAIN\FinanceGroup] FROM WINDOWS;

-- Create Windows login from specific user
CREATE LOGIN [DOMAIN\jdoe] FROM WINDOWS;

-- Grant server role to Windows group
ALTER SERVER ROLE [sysadmin] ADD MEMBER [DOMAIN\DBAs];

-- Create database user from Windows login
USE FinanceDB;
CREATE USER [DOMAIN\FinanceGroup] FOR LOGIN [DOMAIN\FinanceGroup];

-- Grant database permissions to Windows group
ALTER ROLE [db_datareader] ADD MEMBER [DOMAIN\FinanceGroup];
```

**Group-Based Security Strategy**:
```sql
-- Create database roles for business functions
CREATE ROLE HR_ReadOnly;
CREATE ROLE HR_Update;
CREATE ROLE HR_Admin;

-- Grant specific permissions to roles
GRANT SELECT ON SCHEMA::hr TO HR_ReadOnly;
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::hr TO HR_Update;
GRANT ALL ON SCHEMA::hr TO HR_Admin;

-- Map domain groups to database roles
ALTER ROLE HR_ReadOnly ADD MEMBER [DOMAIN\HR_Team];
ALTER ROLE HR_Update ADD MEMBER [DOMAIN\HR_Managers];
ALTER ROLE HR_Admin ADD MEMBER [DOMAIN\HR_Admins];
```

#### 2.2 SQL Server Authentication

**Concept**: Uses SQL Server login credentials (username/password)
- **Advantages**: Works in workgroup environments, no domain dependency
- **Disadvantages**: Password management complexity, security challenges

**Implementation**:
```sql
-- Create SQL Server login with password policy
CREATE LOGIN SQLAdmin WITH PASSWORD = 'ComplexPassword123!',
    CHECK_POLICY = ON,
    CHECK_EXPIRATION = ON;

-- Create login without password expiration for service accounts
CREATE LOGIN ServiceAccount WITH PASSWORD = 'SecureServicePassword456!',
    CHECK_POLICY = ON,
    CHECK_EXPIRATION = OFF;

-- Map login to database user
USE MyDatabase;
CREATE USER ServiceApp FOR LOGIN ServiceAccount;

-- Grant specific permissions
GRANT SELECT, INSERT, UPDATE, DELETE TO ServiceApp;
```

**Password Policy Configuration**:
```sql
-- Check current password policy settings
EXEC xp_loginconfig 'password policy';

-- Configure password policy at server level
-- This is controlled by Windows password policy for SQL logins

-- Create login with password complexity requirements
CREATE LOGIN ApplicationUser WITH PASSWORD = 'SecurePassword789!@#',
    CHECK_POLICY = ON;  -- Requires password complexity
```

#### 2.3 Mixed Mode Authentication

**Configuration**: Enables both Windows and SQL Server authentication
```sql
-- Check authentication mode
EXEC xp_loginconfig 'login mode';

-- Change authentication mode (requires restart)
-- Use SQL Server Configuration Manager:
-- 1. SQL Server Properties → Security
-- 2. Select "SQL Server and Windows Authentication mode"
-- 3. Restart SQL Server service
```

#### 2.4 Azure Active Directory Authentication

**Modern Authentication**: Leverages Azure AD for identity management
```sql
-- Configure Azure AD authentication (requires Azure AD setup)
-- This requires Azure AD Connect and proper configuration

-- Create Azure AD user login
CREATE LOGIN [user@domain.com] FROM EXTERNAL PROVIDER;

-- Create Azure AD group login
CREATE LOGIN [AzureGroup] FROM EXTERNAL PROVIDER;

-- Map to database users
USE MyDatabase;
CREATE USER [user@domain.com] FOR LOGIN [user@domain.com];
CREATE USER [AzureGroup] FOR LOGIN [AzureGroup];
```

### 3. Authorization and Permission Management

#### 3.1 Role-Based Security Model

**Server Roles (Fixed)**:
```sql
-- Built-in server roles and their capabilities

-- sysadmin: Full control over SQL Server
ALTER SERVER ROLE [sysadmin] ADD MEMBER [DOMAIN\SQLAdmins];

-- securityadmin: Can manage logins and permissions
ALTER SERVER ROLE [securityadmin] ADD MEMBER [DOMAIN\SecurityTeam];

-- serveradmin: Can configure server settings
ALTER SERVER ROLE [serveradmin] ADD MEMBER [DOMAIN\OpsTeam];

-- dbcreator: Can create databases
ALTER SERVER ROLE [dbcreator] ADD MEMBER [DOMAIN\DevTeam];

-- diskadmin: Can manage disk files
ALTER SERVER ROLE [diskadmin] ADD MEMBER [DOMAIN\OpsTeam];

-- processadmin: Can manage SQL Server processes
ALTER SERVER ROLE [processadmin] ADD MEMBER [DOMAIN\OpsTeam];

-- bulkadmin: Can execute BULK INSERT statements
ALTER SERVER ROLE [bulkadmin] ADD MEMBER [DOMAIN\ETLTeam];

-- Setupadmin: Can manage linked servers and startup procedures
ALTER SERVER ROLE [setupadmin] ADD MEMBER [DOMAIN\IntegrationTeam];
```

**Database Roles (Fixed)**:
```sql
-- Built-in database roles

-- db_owner: Full control over database
ALTER ROLE [db_owner] ADD MEMBER [DOMAIN\DatabaseAdmins];

-- db_securityadmin: Can manage roles and permissions
ALTER ROLE [db_securityadmin] ADD MEMBER [DOMAIN\SecurityTeam];

-- db_accessadmin: Can manage database access
ALTER ROLE [db_accessadmin] ADD MEMBER [DOMAIN\SupportTeam];

-- db_backupoperator: Can backup database
ALTER ROLE [db_backupoperator] ADD MEMBER [DOMAIN\BackupTeam];

-- db_datareader: Can view all data
ALTER ROLE [db_datareader] ADD MEMBER [DOMAIN\Analysts];

-- db_datawriter: Can modify all data
ALTER ROLE [db_datawriter] ADD MEMBER [DOMAIN\Applications];

-- db_denydatareader: Cannot view data
ALTER ROLE [db_denydatareader] ADD MEMBER [UserDeniedRead];

-- db_denydatawriter: Cannot modify data
ALTER ROLE [db_denydatawriter] ADD MEMBER [UserDeniedWrite];
```

**User-Defined Database Roles**:
```sql
-- Create application-specific roles
CREATE ROLE ApplicationReaders;
CREATE ROLE ApplicationWriters;
CREATE ROLE ApplicationAdmins;
CREATE ROLE ReportUsers;
CREATE ROLE ETLProcessors;

-- Grant permissions to user-defined roles
-- Grant SELECT on specific tables to readers
GRANT SELECT ON dbo.Customers TO ApplicationReaders;
GRANT SELECT ON dbo.Products TO ApplicationReaders;
GRANT SELECT ON dbo.Orders TO ApplicationReaders;

-- Grant data modification permissions to writers
GRANT SELECT, INSERT, UPDATE, DELETE ON dbo.Orders TO ApplicationWriters;
GRANT SELECT, INSERT, UPDATE, DELETE ON dbo.OrderDetails TO ApplicationWriters;

-- Grant administrative permissions
GRANT ALL ON dbo.Orders TO ApplicationAdmins;
GRANT ALL ON dbo.OrderDetails TO ApplicationAdmins;

-- Grant report-specific permissions
GRANT SELECT ON dbo.vw_SalesSummary TO ReportUsers;
GRANT SELECT ON dbo.vw_CustomerAnalysis TO ReportUsers;

-- Grant ETL processing permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON dbo.StagingTables TO ETLProcessors;
GRANT EXECUTE ON dbo.sp_ProcessETL TO ETLProcessors;
```

#### 3.2 Permission Management

**Granting Permissions**:
```sql
-- Grant server-level permissions
GRANT SELECT ON sys.server_permissions TO ApplicationLogin;

-- Grant database-level permissions
USE SalesDatabase;
GRANT SELECT, INSERT, UPDATE ON dbo.Products TO ApplicationUser;
GRANT EXECUTE ON dbo.sp_ProcessOrder TO ApplicationUser;

-- Grant schema-level permissions
GRANT SELECT ON SCHEMA::Production TO ProductionUser;
GRANT ALL ON SCHEMA::Admin TO AdminUser;

-- Grant object-specific permissions
GRANT SELECT ON dbo.vw_TopCustomers TO ReportingUser;
GRANT EXECUTE ON dbo.usp_GetSalesReport TO ReportingUser;
```

**Revoking Permissions**:
```sql
-- Revoke specific permissions
REVOKE INSERT, UPDATE ON dbo.Sales FROM ApplicationUser;

-- Revoke schema permissions
REVOKE SELECT ON SCHEMA::Sales FROM ReportUser;

-- Revoke role membership
ALTER ROLE ApplicationWriters DROP MEMBER UserAccount;

-- Revoke server role membership
ALTER SERVER ROLE [dbcreator] DROP MEMBER [DOMAIN\DevTeam];
```

**Denying Permissions**:
```sql
-- Explicitly deny permissions (takes precedence over GRANT)
DENY SELECT ON dbo.EmployeeSalaries TO HRStaff;

-- Deny database role membership
DENY db_datareader TO GuestUser;

-- Deny server permissions
DENY VIEW ANY DATABASE TO RestrictedUser;
```

#### 3.3 Ownership Chains and Security Context

**Ownership Chain Concept**:
- **Same Owner**: If all objects in a query are owned by the same principal, permissions are checked only at the first object
- **Broken Chain**: When object ownership changes, full permission checking occurs

**Managing Ownership Chains**:
```sql
-- Set schema ownership to maintain ownership chains
CREATE SCHEMA Sales AUTHORIZATION dbo;

-- Create all objects within the same schema
CREATE TABLE Sales.Orders (...);
CREATE VIEW Sales.vw_OrderSummary AS
SELECT * FROM Sales.Orders;  -- Ownership chain maintained

-- Create stored procedures for security
CREATE PROCEDURE Sales.usp_GetOrderDetails
    @OrderID INT
AS
BEGIN
    SELECT OrderID, CustomerID, OrderDate, TotalAmount
    FROM Sales.Orders
    WHERE OrderID = @OrderID;
END;

-- Grant execute permission on procedure instead of table
GRANT EXECUTE ON Sales.usp_GetOrderDetails TO ReportingUser;
```

**Dynamic SQL Security**:
```sql
-- Dynamic SQL breaks ownership chains
CREATE PROCEDURE Sales.usp_GetOrderDynamic
    @OrderID INT
AS
BEGIN
    DECLARE @SQL NVARCHAR(MAX);
    SET @SQL = 'SELECT * FROM Sales.Orders WHERE OrderID = ' + CAST(@OrderID AS NVARCHAR(10));
    EXEC sp_executesql @SQL;  -- Breaks ownership chain
END;

-- Use EXECUTE AS to control security context
CREATE PROCEDURE Sales.usp_GetOrderAsUser
    @OrderID INT
WITH EXECUTE AS 'db_owner'
AS
BEGIN
    SELECT OrderID, CustomerID, OrderDate, TotalAmount
    FROM Sales.Orders
    WHERE OrderID = @OrderID;
END;
```

### 4. Advanced Security Features

#### 4.1 Transparent Data Encryption (TDE)

**Purpose**: Encrypt database files at rest
```sql
-- Create master key (only once per server)
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'MasterKeyPassword123!';

-- Create certificate for TDE
CREATE CERTIFICATE TDECertificate
WITH SUBJECT = 'TDE Certificate',
EXPIRY_DATE = '2030-12-31';

-- Create database encryption key
USE SalesDatabase;
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDECertificate;

-- Enable TDE
ALTER DATABASE SalesDatabase
SET ENCRYPTION ON;

-- Verify encryption status
SELECT 
    database_id,
    name,
    is_encrypted,
    encryption_state_desc,
    percent_complete
FROM sys.dm_database_encryption_keys;
```

**Backup Encryption**:
```sql
-- Enable backup encryption
BACKUP DATABASE SalesDatabase
TO DISK = 'C:\Backups\SalesDatabase_Encrypted.bak'
WITH 
    ENCRYPTION (
        ALGORITHM = AES_256,
        SERVER CERTIFICATE = BackupCert
    ),
    COMPRESSION;
```

#### 4.2 Always Encrypted

**Purpose**: Encrypt sensitive columns while allowing application access
```sql
-- Create column master key
CREATE COLUMN MASTER KEY [CMK_Auto1]
WITH (
    KEY_STORE_PROVIDER_NAME = 'MSSQL_CERTIFICATE_STORE',
    KEY_PATH = 'CurrentUser/My/ABCEFG1234567890ABCDEFG1234567890'
);

-- Create column encryption key
CREATE COLUMN ENCRYPTION KEY [CEK_Auto1]
WITH VALUES (
    COLUMN_MASTER_KEY = [CMK_Auto1],
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x...  -- Encrypted column encryption key value
);

-- Create table with encrypted columns
CREATE TABLE dbo.Customers (
    CustomerID INT IDENTITY(1,1) PRIMARY KEY,
    SSN CHAR(11) COLLATE Latin1_General_BIN2 ENCRYPTED 
        WITH (
            COLUMN_ENCRYPTION_KEY = [CEK_Auto1],
            ENCRYPTION_TYPE = DETERMINISTIC,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ),
    CreditCardNumber NVARCHAR(16) COLLATE Latin1_General_BIN2 ENCRYPTED
        WITH (
            COLUMN_ENCRYPTION_KEY = [CEK_Auto1],
            ENCRYPTION_TYPE = RANDOMIZED,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ),
    FirstName NVARCHAR(50),
    LastName NVARCHAR(50)
);
```

#### 4.3 Row-Level Security

**Purpose**: Restrict data access based on user identity
```sql
-- Create security policy function
CREATE FUNCTION dbo.fn_securitypredicate(@CustomerID INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 
    FROM dbo.CustomerAccess ca
    WHERE ca.UserName = USER_NAME()
    AND ca.CustomerID = @CustomerID
);

-- Enable row-level security on table
CREATE SECURITY POLICY CustomerSecurityPolicy
ADD FILTER PREDICATE dbo.fn_securitypredicate(CustomerID)
ON dbo.Customers
AFTER INSERT
WITH (STATE = ON);

-- Create customer access table
CREATE TABLE dbo.CustomerAccess (
    CustomerID INT NOT NULL,
    UserName NVARCHAR(128) NOT NULL,
    PRIMARY KEY (CustomerID, UserName)
);
```

#### 4.4 Dynamic Data Masking

**Purpose**: Hide sensitive data in query results
```sql
-- Create table with masked columns
CREATE TABLE dbo.Employee (
    EmployeeID INT IDENTITY(1,1) PRIMARY KEY,
    EmployeeName NVARCHAR(100),
    Email NVARCHAR(100) MASKED WITH (FUNCTION = 'email()'),
    PhoneNumber NVARCHAR(20) MASKED WITH (FUNCTION = 'partial(0,"XXX-XXX-",4)'),
    Salary DECIMAL(10,2) MASKED WITH (FUNCTION = 'random(1, 100000)'),
    SocialSecurityNumber CHAR(11) MASKED WITH (FUNCTION = 'default()')
);

-- Grant UNMASK permission to specific users
GRANT UNMASK TO HRManager;

-- Create masking policy
CREATE SECURITY POLICY MaskingPolicy
ADD FILTER PREDICATE dbo.fn_securitypredicate(EmployeeID)
ON dbo.Employee
FOR SELECT
WITH (STATE = ON);
```

### 5. Service Account Security

#### 5.1 Service Account Types and Best Practices

**Local System Account**:
- **Account**: NT AUTHORITY\SYSTEM
- **Privileges**: Highest local privileges
- **Use Case**: NOT recommended for production (security risk)
- **Limitations**: Account not managed by domain, no network access

**Local User Account**:
- **Account**: Local Windows user
- **Privileges**: Configurable local privileges
- **Use Case**: Small organizations, isolated servers
- **Management**: Local password management required

**Domain User Account**:
- **Account**: DOMAIN\ServiceAccount
- **Privileges**: Configurable domain permissions
- **Use Case**: Recommended for production
- **Benefits**: Centralized management, audit trail

**Managed Service Account (MSA)**:
- **Account**: DOMAIN\AccountName$
- **Managed by**: Active Directory automatically
- **Benefits**: No password expiration, no manual password management
- **Limitations**: Single server, no group membership

**Group Managed Service Account (gMSA)**:
- **Account**: DOMAIN\AccountName$
- **Managed by**: Active Directory automatically
- **Benefits**: Multi-server support, password management, group membership
- **Requirements**: Windows Server 2012 or later, PowerShell

#### 5.2 Service Account Configuration

**SQL Server Database Engine Service**:
```powershell
# Configure using SQL Server Configuration Manager
# - Right-click SQL Server (MSSQLSERVER) → Properties
# - Log on tab → Select appropriate account
# - Configure password and startup type

# Recommended: Domain User Account or gMSA
Account: DOMAIN\SQLService$
Privileges: Log on as a service, Adjust memory quotas for a process, Bypass traverse checking
```

**SQL Server Agent Service**:
```powershell
# Configure SQL Server Agent
Account: DOMAIN\SQLAgent$
Privileges: Log on as a service, Create global objects, Increase quotas
```

**Service Account Security Requirements**:
```sql
-- Check current service accounts
SELECT 
    service_type_desc,
    service_name,
    service_account,
    status_desc,
    start_up_type_desc
FROM sys.dm_server_services;

-- Verify service account permissions
-- Service account needs:
-- 1. Log on as a service right
-- 2. Access to data and log files
-- 3. Access to backup directories
-- 4. Network access for remote connections
```

#### 5.3 Privileges and Rights Assignment

**Required Windows Privileges**:
```powershell
# Log on as a service
# Bypass traverse checking (SeChangeNotifyPrivilege)
# Adjust memory quotas for a process (SeIncreaseQuotaPrivilege)
# Increase scheduling priority (SeIncreaseBasePriorityPrivilege)
# Replace a process-level token (SeAssignPrimaryTokenPrivilege)

# Grant privileges using Local Security Policy (secpol.msc)
# Or using PowerShell:
Add-LocalGroupMember -Group "Logon as a service" -Member "DOMAIN\SQLService"
```

**File System Permissions**:
```powershell
# Grant access to data directory
icacls "C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\DATA" 
/grant "DOMAIN\SQLService:(OI)(CI)F"

# Grant access to log directory  
icacls "C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\LOG" 
/grant "DOMAIN\SQLService:(OI)(CI)F"

# Grant access to backup directory
icacls "E:\SQLBackups" 
/grant "DOMAIN\SQLService:(OI)(CI)F"
```

### 6. Auditing and Compliance

#### 6.1 SQL Server Audit

**Creating Server Audit**:
```sql
-- Create server audit
CREATE SERVER AUDIT [SecurityAudit]
TO FILE (
    FILEPATH = 'C:\SQLAudit',
    MAXSIZE = 1 GB,
    MAX_ROLLOVER_FILES = 10,
    RESERVE_DISK_SPACE = OFF
)
WITH (
    QUEUE_DELAY = 1000,
    ON_FAILURE = CONTINUE,
    AUDIT_GUID = '12345678-1234-1234-1234-123456789012'
);

-- Create audit specification for server-level events
CREATE SERVER AUDIT SPECIFICATION [ServerAuditSpec]
FOR SERVER AUDIT [SecurityAudit]
ADD (SERVER_PRINCIPAL_CHANGE_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (SERVER_OBJECT_CHANGE_GROUP),
ADD (SERVER_OPERATION_GROUP),
ADD (SERVER_PERMISSION_CHANGE_GROUP)
WITH (STATE = ON);

-- Create database audit specification
CREATE DATABASE AUDIT SPECIFICATION [DatabaseAuditSpec]
FOR SERVER AUDIT [SecurityAudit]
ADD (INSERT ON dbo.Orders BY ApplicationUser),
ADD (UPDATE ON dbo.Orders BY ApplicationUser),
ADD (DELETE ON dbo.Orders BY ApplicationUser),
ADD (SELECT ON dbo.EmployeeSalaries BY PUBLIC)
WITH (STATE = ON);
```

**Extended Events for Auditing**:
```sql
-- Create extended event session for login monitoring
CREATE EVENT SESSION [LoginAudit] ON SERVER
ADD EVENT sqlserver.login(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.client_pid,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.username
    )
)
ADD TARGET package0.event_file(
    SET filename = N'C:\ExtendedEvents\LoginAudit'
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

-- Start the extended event session
ALTER EVENT SESSION [LoginAudit] ON SERVER
STATE = START;
```

#### 6.2 Compliance and Regulatory Requirements

**SOX (Sarbanes-Oxley) Compliance**:
```sql
-- Implement access logging
CREATE TABLE dbo.AuditLog (
    AuditID BIGINT IDENTITY(1,1) PRIMARY KEY,
    EventType NVARCHAR(50) NOT NULL,
    TableName NVARCHAR(128) NOT NULL,
    ColumnName NVARCHAR(128),
    OldValue NVARCHAR(MAX),
    NewValue NVARCHAR(MAX),
    UserName NVARCHAR(128) NOT NULL,
    ApplicationName NVARCHAR(128),
    HostName NVARCHAR(128),
    EventTime DATETIME2 NOT NULL DEFAULT GETDATE(),
    RowKey NVARCHAR(128)  -- To identify the affected row
);

-- Audit trigger for critical tables
CREATE TRIGGER tr_Audit_EmployeeSalary
ON dbo.EmployeeSalaries
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Log old values
    IF EXISTS (SELECT 1 FROM deleted)
    BEGIN
        INSERT INTO dbo.AuditLog (
            EventType, TableName, ColumnName, OldValue, UserName, ApplicationName, HostName, RowKey
        )
        SELECT 
            'UPDATE/DELETE',
            'EmployeeSalaries',
            'Salary',
            CAST(d.Salary AS NVARCHAR(MAX)),
            SYSTEM_USER,
            APP_NAME(),
            HOST_NAME(),
            CAST(d.EmployeeID AS NVARCHAR(128))
        FROM deleted d;
    END
    
    -- Log new values
    IF EXISTS (SELECT 1 FROM inserted)
    BEGIN
        INSERT INTO dbo.AuditLog (
            EventType, TableName, ColumnName, NewValue, UserName, ApplicationName, HostName, RowKey
        )
        SELECT 
            'INSERT/UPDATE',
            'EmployeeSalaries',
            'Salary',
            CAST(i.Salary AS NVARCHAR(MAX)),
            SYSTEM_USER,
            APP_NAME(),
            HOST_NAME(),
            CAST(i.EmployeeID AS NVARCHAR(128))
        FROM inserted i;
    END
END;
```

**GDPR Compliance**:
```sql
-- Data discovery for personal information
CREATE VIEW vw_PersonalDataInventory AS
SELECT 
    t.name AS TableName,
    c.name AS ColumnName,
    c.system_type_id,
    CASE 
        WHEN c.name LIKE '%ssn%' THEN 'SSN'
        WHEN c.name LIKE '%email%' THEN 'Email'
        WHEN c.name LIKE '%phone%' THEN 'Phone'
        WHEN c.name LIKE '%address%' THEN 'Address'
        WHEN c.name LIKE '%birth%' THEN 'Birth Date'
        WHEN c.name LIKE '%name%' THEN 'Name'
        ELSE 'Other'
    END AS DataType,
    (SELECT COUNT(*) FROM sys.database_principals) AS AccessCount
FROM sys.tables t
INNER JOIN sys.columns c ON t.object_id = c.object_id
WHERE c.name LIKE '%ssn%' 
OR c.name LIKE '%email%'
OR c.name LIKE '%phone%'
OR c.name LIKE '%address%'
OR c.name LIKE '%birth%'
OR c.name LIKE '%name%';

-- Data retention policy
CREATE PROCEDURE sp_ApplyDataRetention
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Archive customer data older than 7 years (GDPR compliance)
    INSERT INTO dbo.CustomerArchive
    SELECT * FROM dbo.Customers
    WHERE CreatedDate < DATEADD(YEAR, -7, GETDATE());
    
    -- Delete old customer data
    DELETE FROM dbo.Customers
    WHERE CreatedDate < DATEADD(YEAR, -7, GETDATE());
    
    -- Log retention action
    INSERT INTO dbo.AuditLog (EventType, TableName, UserName, EventTime)
    VALUES ('Data Retention', 'Customers', SYSTEM_USER, GETDATE());
END;
```

#### 6.3 Security Monitoring and Alerting

**Automated Security Monitoring**:
```sql
-- Monitor failed login attempts
CREATE EVENT SESSION [FailedLogins] ON SERVER
ADD EVENT sqlserver.sql_statement_completed(
    WHERE ([sqlserver].[database_id]=(1) AND [sqlserver].[statement] LIKE '%login%failed%')
)
ADD TARGET package0.event_file(
    SET filename = N'C:\ExtendedEvents\FailedLogins.xel'
);

-- Monitor permission changes
CREATE PROCEDURE sp_MonitorPermissionChanges
AS
BEGIN
    -- Alert on role membership changes
    IF EXISTS (
        SELECT 1 FROM sys.database_principals 
        WHERE create_date > DATEADD(HOUR, -1, GETDATE())
        AND type IN ('R')  -- Roles
    )
    BEGIN
        -- Send alert email or log to monitoring system
        INSERT INTO dbo.SecurityAlerts (AlertType, AlertMessage, AlertTime)
        VALUES ('Role Change', 'New database role created', GETDATE());
    END
    
    -- Monitor database creation
    IF EXISTS (
        SELECT 1 FROM sys.databases 
        WHERE create_date > DATEADD(HOUR, -1, GETDATE())
    )
    BEGIN
        INSERT INTO dbo.SecurityAlerts (AlertType, AlertMessage, AlertTime)
        VALUES ('Database Creation', 'New database created', GETDATE());
    END
END;

-- Schedule monitoring job
USE msdb;
EXEC dbo.sp_add_job
    @job_name = N'Security Monitoring',
    @enabled = 1,
    @description = N'Monitors security-related changes';

EXEC sp_add_jobstep
    @job_name = N'Security Monitoring',
    @step_name = N'Check for Security Changes',
    @command = N'EXEC sp_MonitorPermissionChanges;';
```

## Practical Exercises

### Exercise 1: Authentication and Authorization Setup

**Objective**: Implement a comprehensive authentication and authorization system for a multi-department organization

**Scenario**: Design security for TechCorp with the following requirements:

**Departments**:
- IT (System administrators, developers)
- HR (HR managers, HR specialists)
- Finance (Accountants, analysts)
- Sales (Sales managers, representatives)
- Marketing (Marketing managers, coordinators)

**Security Requirements**:
- Windows Authentication for all users
- Department-based access control
- Role separation (read-only vs. update access)
- Administrative access for IT only
- Audit logging for sensitive data access

**Tasks**:

1. **Create Windows Domain Groups**
```powershell
# Create Active Directory groups
New-ADGroup -Name "TechCorp_IT_Admins" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "TechCorp_IT_Developers" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "TechCorp_HR_Managers" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "TechCorp_HR_Specialists" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "TechCorp_Finance_Accountants" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "TechCorp_Finance_Analysts" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "TechCorp_Sales_Managers" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "TechCorp_Sales_Reps" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "TechCorp_Marketing_Managers" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "TechCorp_Marketing_Coords" -GroupScope Global -GroupCategory Security
```

2. **Create Server-Level Security**
```sql
-- Create Windows logins from domain groups
CREATE LOGIN [DOMAIN\TechCorp_IT_Admins] FROM WINDOWS;
CREATE LOGIN [DOMAIN\TechCorp_IT_Developers] FROM WINDOWS;
CREATE LOGIN [DOMAIN\TechCorp_HR_Managers] FROM WINDOWS;
CREATE LOGIN [DOMAIN\TechCorp_HR_Specialists] FROM WINDOWS;
CREATE LOGIN [DOMAIN\TechCorp_Finance_Accountants] FROM WINDOWS;
CREATE LOGIN [DOMAIN\TechCorp_Finance_Analysts] FROM WINDOWS;
CREATE LOGIN [DOMAIN\TechCorp_Sales_Managers] FROM WINDOWS;
CREATE LOGIN [DOMAIN\TechCorp_Sales_Reps] FROM WINDOWS;
CREATE LOGIN [DOMAIN\TechCorp_Marketing_Managers] FROM WINDOWS;
CREATE LOGIN [DOMAIN\TechCorp_Marketing_Coords] FROM WINDOWS;

-- Assign server roles
ALTER SERVER ROLE [sysadmin] ADD MEMBER [DOMAIN\TechCorp_IT_Admins];
ALTER SERVER ROLE [dbcreator] ADD MEMBER [DOMAIN\TechCorp_IT_Developers];
ALTER SERVER ROLE [dbcreator] ADD MEMBER [DOMAIN\TechCorp_IT_Admins];
```

3. **Create Database and Schema Structure**
```sql
-- Create department-specific schemas
CREATE SCHEMA IT AUTHORIZATION dbo;
CREATE SCHEMA HR AUTHORIZATION dbo;
CREATE SCHEMA Finance AUTHORIZATION dbo;
CREATE SCHEMA Sales AUTHORIZATION dbo;
CREATE SCHEMA Marketing AUTHORIZATION dbo;

-- Create database users mapped to domain groups
CREATE USER [DOMAIN\TechCorp_IT_Admins] FOR LOGIN [DOMAIN\TechCorp_IT_Admins];
CREATE USER [DOMAIN\TechCorp_IT_Developers] FOR LOGIN [DOMAIN\TechCorp_IT_Developers];
CREATE USER [DOMAIN\TechCorp_HR_Managers] FOR LOGIN [DOMAIN\TechCorp_HR_Managers];
CREATE USER [DOMAIN\TechCorp_HR_Specialists] FOR LOGIN [DOMAIN\TechCorp_HR_Specialists];
CREATE USER [DOMAIN\TechCorp_Finance_Accountants] FOR LOGIN [DOMAIN\TechCorp_Finance_Accountants];
CREATE USER [DOMAIN\TechCorp_Finance_Analysts] FOR LOGIN [DOMAIN\TechCorp_Finance_Analysts];
CREATE USER [DOMAIN\TechCorp_Sales_Managers] FOR LOGIN [DOMAIN\TechCorp_Sales_Managers];
CREATE USER [DOMAIN\TechCorp_Sales_Reps] FOR LOGIN [DOMAIN\TechCorp_Sales_Reps];
CREATE USER [DOMAIN\TechCorp_Marketing_Managers] FOR LOGIN [DOMAIN\TechCorp_Marketing_Managers];
CREATE USER [DOMAIN\TechCorp_Marketing_Coords] FOR LOGIN [DOMAIN\TechCorp_Marketing_Coords];
```

4. **Implement Role-Based Permissions**
```sql
-- Create department-specific roles
CREATE ROLE IT_Admins;
CREATE ROLE IT_Developers;
CREATE ROLE HR_Managers;
CREATE ROLE HR_Specialists;
CREATE ROLE Finance_Accountants;
CREATE ROLE Finance_Analysts;
CREATE ROLE Sales_Managers;
CREATE ROLE Sales_Reps;
CREATE ROLE Marketing_Managers;
CREATE ROLE Marketing_Coords;

-- Grant permissions to roles
-- IT permissions
GRANT ALL ON SCHEMA::IT TO IT_Admins;
GRANT SELECT ON SCHEMA::HR TO IT_Admins;
GRANT SELECT ON SCHEMA::Finance TO IT_Admins;
GRANT SELECT ON SCHEMA::Sales TO IT_Admins;
GRANT SELECT ON SCHEMA::Marketing TO IT_Admins;

GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::IT TO IT_Developers;
GRANT SELECT ON SCHEMA::Production TO IT_Developers;

-- HR permissions
GRANT ALL ON SCHEMA::HR TO HR_Managers;
GRANT SELECT, INSERT, UPDATE ON SCHEMA::HR TO HR_Specialists;
GRANT SELECT ON dbo.vw_EmployeeSummary TO HR_Specialists;

-- Finance permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::Finance TO Finance_Accountants;
GRANT SELECT ON SCHEMA::Finance TO Finance_Analysts;
GRANT EXECUTE ON dbo.sp_FinancialReports TO Finance_Analysts;

-- Sales permissions
GRANT ALL ON SCHEMA::Sales TO Sales_Managers;
GRANT SELECT, INSERT, UPDATE ON SCHEMA::Sales TO Sales_Reps;
GRANT SELECT ON dbo.vw_SalesReports TO Sales_Reps;

-- Marketing permissions
GRANT ALL ON SCHEMA::Marketing TO Marketing_Managers;
GRANT SELECT, INSERT, UPDATE, DELETE ON dbo.Marketing.Campaigns TO Marketing_Managers;
GRANT SELECT ON dbo.vw_CustomerAnalysis TO Marketing_Coords;
```

5. **Assign Groups to Roles**
```sql
-- Map domain groups to database roles
ALTER ROLE IT_Admins ADD MEMBER [DOMAIN\TechCorp_IT_Admins];
ALTER ROLE IT_Developers ADD MEMBER [DOMAIN\TechCorp_IT_Developers];
ALTER ROLE HR_Managers ADD MEMBER [DOMAIN\TechCorp_HR_Managers];
ALTER ROLE HR_Specialists ADD MEMBER [DOMAIN\TechCorp_HR_Specialists];
ALTER ROLE Finance_Accountants ADD MEMBER [DOMAIN\TechCorp_Finance_Accountants];
ALTER ROLE Finance_Analysts ADD MEMBER [DOMAIN\TechCorp_Finance_Analysts];
ALTER ROLE Sales_Managers ADD MEMBER [DOMAIN\TechCorp_Sales_Managers];
ALTER ROLE Sales_Reps ADD MEMBER [DOMAIN\TechCorp_Sales_Reps];
ALTER ROLE Marketing_Managers ADD MEMBER [DOMAIN\TechCorp_Marketing_Managers];
ALTER ROLE Marketing_Coords ADD MEMBER [DOMAIN\TechCorp_Marketing_Coords];
```

**Deliverable**: Complete security implementation with documentation of all permissions and access levels

### Exercise 2: Encryption Implementation

**Objective**: Implement comprehensive encryption for sensitive data

**Scenario**: Implement encryption for customer financial data including TDE, backup encryption, and column-level encryption

**Requirements**:
- Encrypt database files at rest (TDE)
- Encrypt backup files
- Encrypt sensitive customer data columns
- Maintain application functionality
- Performance impact monitoring

**Tasks**:

1. **Implement Transparent Data Encryption**
```sql
-- Create master key
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'MasterKey_P@ssw0rd123!';

-- Create certificate for TDE
CREATE CERTIFICATE TDEServerCertificate
WITH SUBJECT = 'TDE Database Certificate',
EXPIRY_DATE = '2030-12-31';

-- Enable TDE on database
USE CustomerDB;
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDEServerCertificate;

ALTER DATABASE CustomerDB
SET ENCRYPTION ON;

-- Monitor encryption status
SELECT 
    database_id,
    name,
    is_encrypted,
    encryption_state_desc,
    encryption_state,
    percent_complete,
    key_algorithm,
    key_length
FROM sys.dm_database_encryption_keys
ORDER BY name;
```

2. **Implement Backup Encryption**
```sql
-- Create backup certificate
USE master;
CREATE CERTIFICATE BackupCert
WITH SUBJECT = 'Backup Encryption Certificate',
EXPIRY_DATE = '2030-12-31';

-- Backup with encryption
BACKUP DATABASE CustomerDB
TO DISK = 'C:\Backups\CustomerDB_Encrypted.bak'
WITH 
    ENCRYPTION (
        ALGORITHM = AES_256,
        SERVER CERTIFICATE = BackupCert
    ),
    COMPRESSION,
    INIT,
    SKIP,
    STATS = 10;

-- Backup with different algorithm
BACKUP DATABASE CustomerDB
TO DISK = 'C:\Backups\CustomerDB_Encrypted_TRIPLEDES.bak'
WITH 
    ENCRYPTION (
        ALGORITHM = TRIPLE_DES_3KEY,
        SERVER CERTIFICATE = BackupCert
    ),
    COMPRESSION;
```

3. **Implement Column-Level Encryption**
```sql
-- Create column master key (using certificate store)
CREATE COLUMN MASTER KEY [CMK_CustomerSensitive]
WITH (
    KEY_STORE_PROVIDER_NAME = 'MSSQL_CERTIFICATE_STORE',
    KEY_PATH = 'CurrentUser/My/1234567890ABCDEF1234567890ABCDEF12345678'
);

-- Create column encryption key
CREATE COLUMN ENCRYPTION KEY [CEK_CustomerSensitive]
WITH VALUES (
    COLUMN_MASTER_KEY = [CMK_CustomerSensitive],
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x...  -- Encrypted CEK value generated by application
);

-- Create table with encrypted columns
CREATE TABLE dbo.CustomerSensitive (
    CustomerID INT IDENTITY(1,1) PRIMARY KEY,
    CustomerName NVARCHAR(100) NOT NULL,
    SSN CHAR(11) COLLATE Latin1_General_BIN2 ENCRYPTED 
        WITH (
            COLUMN_ENCRYPTION_KEY = [CEK_CustomerSensitive],
            ENCRYPTION_TYPE = DETERMINISTIC,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ),
    CreditCardNumber NVARCHAR(20) COLLATE Latin1_General_BIN2 ENCRYPTED
        WITH (
            COLUMN_ENCRYPTION_KEY = [CEK_CustomerSensitive],
            ENCRYPTION_TYPE = RANDOMIZED,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ),
    BankAccountNumber NVARCHAR(20) COLLATE Latin1_General_BIN2 ENCRYPTED
        WITH (
            COLUMN_ENCRYPTION_KEY = [CEK_CustomerSensitive],
            ENCRYPTION_TYPE = DETERMINISTIC,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ),
    Email NVARCHAR(100) COLLATE Latin1_General_BIN2
        -- Not encrypted for searchability
);

-- Insert test data (requires .NET application or SSMS with Always Encrypted support)
-- Application must use SqlConnectionStringBuilder with ColumnEncryptionSetting = Enabled
```

4. **Performance Monitoring**
```sql
-- Monitor encryption impact on queries
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Test query performance on encrypted vs unencrypted data
SELECT 
    CustomerID,
    CustomerName,
    SSN,
    CreditCardNumber,
    Email
FROM dbo.CustomerSensitive
WHERE SSN = '123-45-6789';  -- Deterministic encryption allows equality search

-- Check CPU usage during encryption operations
SELECT 
    database_id,
    cpu_time,
    total_elapsed_time,
    logical_reads,
    physical_reads
FROM sys.dm_exec_requests
WHERE database_id = DB_ID('CustomerDB');
```

**Deliverable**: Complete encryption implementation with performance analysis and documentation

### Exercise 3: Security Auditing Implementation

**Objective**: Implement comprehensive auditing for compliance and security monitoring

**Scenario**: Implement auditing for a healthcare database requiring HIPAA compliance

**Requirements**:
- Monitor all data access to patient records
- Audit administrative actions
- Track login attempts and failures
- Implement data change logging
- Create audit reporting capabilities

**Tasks**:

1. **Create Server Audit**
```sql
-- Create server audit for security events
CREATE SERVER AUDIT [HIPAA_Audit]
TO FILE (
    FILEPATH = 'C:\SQLAudit\HIPAA\',
    MAXSIZE = 2 GB,
    MAX_ROLLOVER_FILES = 100,
    RESERVE_DISK_SPACE = ON
)
WITH (
    QUEUE_DELAY = 1000,
    ON_FAILURE = CONTINUE,
    AUDIT_GUID = 'ABCDEF12-3456-7890-ABCD-EF1234567890'
);

-- Enable the audit
ALTER SERVER AUDIT [HIPAA_Audit] WITH (STATE = ON);
```

2. **Create Audit Specifications**
```sql
-- Create server audit specification
CREATE SERVER AUDIT SPECIFICATION [HIPAA_Server_AuditSpec]
FOR SERVER AUDIT [HIPAA_Audit]
ADD (AUDIT_CHANGE_GROUP),
ADD (FAILED_LOGIN_GROUP),
ADD (LOGIN_CHANGE_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (SERVER_OBJECT_CHANGE_GROUP),
ADD (SERVER_OPERATION_GROUP),
ADD (SERVER_PERMISSION_CHANGE_GROUP),
ADD (SCHEMA_OBJECT_ACCESS_GROUP),
ADD (FAILED_AUTHENTICATION_GROUP)
WITH (STATE = ON);

-- Create database audit specification for patient data
USE PatientDB;
CREATE DATABASE AUDIT SPECIFICATION [HIPAA_Database_AuditSpec]
FOR SERVER AUDIT [HIPAA_Audit]
ADD (INSERT ON dbo.Patients BY PUBLIC),
ADD (UPDATE ON dbo.Patients BY PUBLIC),
ADD (DELETE ON dbo.Patients BY PUBLIC),
ADD (SELECT ON dbo.Patients BY PUBLIC),
ADD (INSERT ON dbo.MedicalRecords BY PUBLIC),
ADD (UPDATE ON dbo.MedicalRecords BY PUBLIC),
ADD (SELECT ON dbo.MedicalRecords BY PUBLIC),
ADD (SELECT ON dbo.PatientBilling BY PUBLIC)
WITH (STATE = ON);
```

3. **Create Audit Log Table**
```sql
USE PatientDB;
CREATE TABLE dbo.AuditLog (
    AuditID BIGINT IDENTITY(1,1) PRIMARY KEY,
    EventType NVARCHAR(50) NOT NULL,
    EventCategory NVARCHAR(50) NOT NULL,
    TableName NVARCHAR(128),
    ColumnName NVARCHAR(128),
    OldValue NVARCHAR(MAX),
    NewValue NVARCHAR(MAX),
    UserName NVARCHAR(128) NOT NULL,
    ApplicationName NVARCHAR(128),
    HostName NVARCHAR(128),
    SessionID INT,
    EventTime DATETIME2 NOT NULL DEFAULT GETDATE(),
    RecordKey NVARCHAR(128),  -- To identify affected records
    AdditionalInfo NVARCHAR(MAX)  -- For additional context
);

-- Create index for audit log queries
CREATE INDEX IX_AuditLog_EventTime ON dbo.AuditLog(EventTime);
CREATE INDEX IX_AuditLog_UserName ON dbo.AuditLog(UserName);
CREATE INDEX IX_AuditLog_TableName ON dbo.AuditLog(TableName);
CREATE INDEX IX_AuditLog_EventType ON dbo.AuditLog(EventType);
```

4. **Create Audit Triggers**
```sql
-- Audit trigger for Patients table
CREATE TRIGGER tr_Audit_Patients
ON dbo.Patients
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @EventType NVARCHAR(50),
            @UserName NVARCHAR(128) = SYSTEM_USER,
            @AppName NVARCHAR(128) = APP_NAME(),
            @HostName NVARCHAR(128) = HOST_NAME();
    
    -- Handle inserts
    IF EXISTS (SELECT 1 FROM inserted) AND NOT EXISTS (SELECT 1 FROM deleted)
    BEGIN
        SET @EventType = 'INSERT';
        INSERT INTO dbo.AuditLog (
            EventType, EventCategory, TableName, NewValue, 
            UserName, ApplicationName, HostName, RecordKey
        )
        SELECT 
            @EventType,
            'DATA_CHANGE',
            'Patients',
            CONVERT(NVARCHAR(MAX), 
                (SELECT PatientID, FirstName, LastName, SSN, DateOfBirth 
                 FROM inserted FOR JSON PATH)),
            @UserName,
            @AppName,
            @HostName,
            CAST(i.PatientID AS NVARCHAR(128))
        FROM inserted i;
    END
    
    -- Handle updates
    IF EXISTS (SELECT 1 FROM inserted) AND EXISTS (SELECT 1 FROM deleted)
    BEGIN
        SET @EventType = 'UPDATE';
        
        -- Log old values
        INSERT INTO dbo.AuditLog (
            EventType, EventCategory, TableName, OldValue,
            UserName, ApplicationName, HostName, RecordKey
        )
        SELECT 
            @EventType,
            'DATA_CHANGE',
            'Patients',
            CONVERT(NVARCHAR(MAX), 
                (SELECT PatientID, FirstName, LastName, SSN, DateOfBirth 
                 FROM deleted FOR JSON PATH)),
            @UserName,
            @AppName,
            @HostName,
            CAST(d.PatientID AS NVARCHAR(128))
        FROM deleted d;
        
        -- Log new values
        INSERT INTO dbo.AuditLog (
            EventType, EventCategory, TableName, NewValue,
            UserName, ApplicationName, HostName, RecordKey
        )
        SELECT 
            @EventType,
            'DATA_CHANGE',
            'Patients',
            CONVERT(NVARCHAR(MAX), 
                (SELECT PatientID, FirstName, LastName, SSN, DateOfBirth 
                 FROM inserted FOR JSON PATH)),
            @UserName,
            @AppName,
            @HostName,
            CAST(i.PatientID AS NVARCHAR(128))
        FROM inserted i;
    END
    
    -- Handle deletes
    IF EXISTS (SELECT 1 FROM deleted) AND NOT EXISTS (SELECT 1 FROM inserted)
    BEGIN
        SET @EventType = 'DELETE';
        INSERT INTO dbo.AuditLog (
            EventType, EventCategory, TableName, OldValue,
            UserName, ApplicationName, HostName, RecordKey
        )
        SELECT 
            @EventType,
            'DATA_CHANGE',
            'Patients',
            CONVERT(NVARCHAR(MAX), 
                (SELECT PatientID, FirstName, LastName, SSN, DateOfBirth 
                 FROM deleted FOR JSON PATH)),
            @UserName,
            @AppName,
            @HostName,
            CAST(d.PatientID AS NVARCHAR(128))
        FROM deleted d;
    END
END;
```

5. **Create Audit Reporting**
```sql
-- Create audit summary view
CREATE VIEW dbo.vw_AuditSummary AS
SELECT 
    EventType,
    TableName,
    COUNT(*) as EventCount,
    COUNT(DISTINCT UserName) as UniqueUsers,
    MIN(EventTime) as FirstEvent,
    MAX(EventTime) as LastEvent
FROM dbo.AuditLog
GROUP BY EventType, TableName;

-- Create detailed audit report
CREATE PROCEDURE sp_AuditReport
    @StartDate DATETIME = NULL,
    @EndDate DATETIME = NULL,
    @UserName NVARCHAR(128) = NULL,
    @TableName NVARCHAR(128) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        EventTime,
        EventType,
        EventCategory,
        TableName,
        ColumnName,
        ISNULL(OldValue, '') as OldValue,
        ISNULL(NewValue, '') as NewValue,
        UserName,
        ApplicationName,
        HostName,
        RecordKey
    FROM dbo.AuditLog
    WHERE (@StartDate IS NULL OR EventTime >= @StartDate)
    AND (@EndDate IS NULL OR EventTime <= @EndDate)
    AND (@UserName IS NULL OR UserName = @UserName)
    AND (@TableName IS NULL OR TableName = @TableName)
    ORDER BY EventTime DESC;
END;

-- Create compliance report for patient access
CREATE PROCEDURE sp_PatientAccessReport
    @PatientID INT,
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        EventTime,
        EventType,
        UserName,
        ApplicationName,
        HostName,
        OldValue,
        NewValue
    FROM dbo.AuditLog
    WHERE RecordKey = CAST(@PatientID AS NVARCHAR(128))
    AND EventTime BETWEEN @StartDate AND @EndDate
    AND (EventType IN ('INSERT', 'UPDATE', 'DELETE', 'SELECT') 
         OR TableName IN ('Patients', 'MedicalRecords', 'PatientBilling'))
    ORDER BY EventTime DESC;
END;
```

**Deliverable**: Complete auditing system with compliance reports and monitoring capabilities

### Exercise 4: Service Account Security Hardening

**Objective**: Configure secure service accounts for production SQL Server environment

**Scenario**: Configure service accounts for a production SQL Server installation with multiple databases and applications

**Requirements**:
- Use Group Managed Service Accounts (gMSA) for all SQL Server services
- Implement principle of least privilege
- Secure file system permissions
- Network access restrictions
- Monitoring and alerting for service account activities

**Tasks**:

1. **Create Group Managed Service Account**
```powershell
# Create gMSA for SQL Server Database Engine
New-ADServiceAccount -Name SQLService -DNSHostName sqlprod01.contoso.com

# Create gMSA for SQL Server Agent
New-ADServiceAccount -Name SQLAgent -DNSHostName sqlprod01.contoso.com

# Create gMSA for SQL Server Analysis Services
New-ADServiceAccount -Name SSASService -DNSHostName sqlprod01.contoso.com

# Create gMSA for SQL Server Reporting Services
New-ADServiceAccount -Name SSRSService -DNSHostName sqlprod01.contoso.com

# Install service accounts on SQL Server
Install-AdServiceAccount -Identity SQLService
Install-AdServiceAccount -Identity SQLAgent
Install-AdServiceAccount -Identity SSASService
Install-AdServiceAccount -Identity SSRSService

# Test service account configuration
Test-AdServiceAccount -Identity SQLService
```

2. **Configure Service Account Permissions**
```powershell
# Grant file system permissions for SQL Server Database Engine
$sqlService = "CONTOSO\SQLService$"
$dataPath = "C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\DATA"
$logPath = "C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\LOG"
$backupPath = "E:\SQLBackups"

# Grant permissions on data directory
icacls $dataPath /grant "$sqlService:(OI)(CI)F" /T

# Grant permissions on log directory
icacls $logPath /grant "$sqlService:(OI)(CI)F" /T

# Grant permissions on backup directory
icacls $backupPath /grant "$sqlService:(OI)(CI)F" /T

# Grant permissions on temp directory
$tempPath = "C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\Temp"
icacls $tempPath /grant "$sqlService:(OI)(CI)F" /T
```

3. **Configure Windows Privileges**
```powershell
# Add SQL Server service accounts to necessary groups
Add-LocalGroupMember -Group "Logon as a service" -Member "$env:USERDOMAIN\SQLService$"
Add-LocalGroupMember -Group "Logon as a service" -Member "$env:USERDOMAIN\SQLAgent$"

# Grant specific user rights using secedit
secedit /export /cfg C:\temp\security_policy.inf

# Edit security_policy.inf to add rights for service accounts
# Then import the updated policy
secedit /configure /cfg C:\temp\security_policy.inf
```

4. **Create Service Account Monitoring**
```sql
-- Create table for service account monitoring
CREATE TABLE dbo.ServiceAccountLog (
    LogID BIGINT IDENTITY(1,1) PRIMARY KEY,
    ServiceName NVARCHAR(128) NOT NULL,
    EventType NVARCHAR(50) NOT NULL, -- START, STOP, ERROR
    EventMessage NVARCHAR(MAX),
    EventTime DATETIME2 NOT NULL DEFAULT GETDATE(),
    AdditionalInfo NVARCHAR(MAX)
);

-- Create stored procedure to log service events
CREATE PROCEDURE sp_LogServiceEvent
    @ServiceName NVARCHAR(128),
    @EventType NVARCHAR(50),
    @EventMessage NVARCHAR(MAX),
    @AdditionalInfo NVARCHAR(MAX) = NULL
AS
BEGIN
    INSERT INTO dbo.ServiceAccountLog (ServiceName, EventType, EventMessage, AdditionalInfo)
    VALUES (@ServiceName, @EventType, @EventMessage, @AdditionalInfo);
    
    -- Alert on service errors
    IF @EventType = 'ERROR'
    BEGIN
        -- Send alert email or create incident ticket
        PRINT 'Service Error Detected: ' + @ServiceName + ' - ' + @EventMessage;
    END
END;

-- Create view to monitor service account activity
CREATE VIEW dbo.vw_ServiceAccountActivity AS
SELECT 
    ServiceName,
    EventType,
    COUNT(*) as EventCount,
    MIN(EventTime) as FirstEvent,
    MAX(EventTime) as LastEvent
FROM dbo.ServiceAccountLog
WHERE EventTime > DATEADD(HOUR, -24, GETDATE())
GROUP BY ServiceName, EventType;
```

5. **Configure Network Security**
```powershell
# Configure Windows Firewall rules for SQL Server
New-NetFirewallRule -DisplayName "SQL Server Database Engine" `
    -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow

New-NetFirewallRule -DisplayName "SQL Server Analysis Services" `
    -Direction Inbound -Protocol TCP -LocalPort 2383 -Action Allow

New-NetFirewallRule -DisplayName "SQL Server Reporting Services" `
    -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow

New-NetFirewallRule -DisplayName "SQL Server Integration Services" `
    -Direction Inbound -Protocol TCP -LocalPort 135 -Action Allow
```

**Deliverable**: Secure service account configuration with monitoring and documentation

## Real-World Scenarios

### Scenario 1: Healthcare System Security Implementation

**Background**: MedCare Health System operates 15 hospitals and 200 clinics, processing over 1 million patient records. They require HIPAA compliance, real-time access control, and comprehensive auditing for all patient data access.

**Security Challenges**:
- Complex user base: 5,000+ healthcare professionals, IT staff, administrators
- Multi-location access requirements
- Different access levels by role (physicians, nurses, billing, administration)
- Real-time data access monitoring
- Regulatory compliance and audit requirements
- Integration with existing Active Directory infrastructure

**Current Security Issues**:
- No centralized access control
- Shared database accounts
- No audit trail for data access
- Inconsistent password policies
- No data encryption
- Manual access management process

**DBA Solution Implementation**:

1. **Authentication Infrastructure**
```sql
-- Implement Windows Authentication with domain integration
-- Map domain groups to database roles for scalability

-- Healthcare roles mapping
CREATE LOGIN [MEDCARE\HCP_Physicians] FROM WINDOWS;
CREATE LOGIN [MEDCARE\HCP_Nurses] FROM WINDOWS;
CREATE LOGIN [MEDCARE\HCP_Administrators] FROM WINDOWS;
CREATE LOGIN [MEDCARE\HCP_Billing] FROM WINDOWS;
CREATE LOGIN [MEDCARE\HCP_IT] FROM WINDOWS;

-- System administration roles
ALTER SERVER ROLE [sysadmin] ADD MEMBER [MEDCARE\HCP_IT_SysAdmins];
ALTER SERVER ROLE [securityadmin] ADD MEMBER [MEDCARE\HCP_IT_Security];
ALTER SERVER ROLE [dbcreator] ADD MEMBER [MEDCARE\HCP_IT_Developers];
```

2. **Database-Level Security**
```sql
-- Create application-specific database roles
CREATE ROLE Physicians;
CREATE ROLE Nurses;
CREATE ROLE AdministrativeStaff;
CREATE ROLE BillingStaff;
CREATE ROLE ITStaff;
CREATE ROLE ComplianceAuditor;

-- Grant permissions based on role
-- Physicians can view and update patient records
GRANT SELECT, INSERT, UPDATE ON dbo.PatientRecords TO Physicians;
GRANT SELECT, INSERT, UPDATE ON dbo.MedicalOrders TO Physicians;
GRANT SELECT ON dbo.MedicalHistory TO Physicians;

-- Nurses have limited update permissions
GRANT SELECT, INSERT, UPDATE ON dbo.PatientRecords TO Nurses;
GRANT SELECT ON dbo.MedicalOrders TO Nurses;
GRANT SELECT ON dbo.MedicationAdministration TO Nurses;

-- Administrative staff can view patient information
GRANT SELECT ON dbo.PatientRecords TO AdministrativeStaff;
GRANT SELECT ON dbo.BillingRecords TO AdministrativeStaff;

-- Billing staff can access billing and insurance information
GRANT SELECT, INSERT, UPDATE ON dbo.BillingRecords TO BillingStaff;
GRANT SELECT ON dbo.InsuranceRecords TO BillingStaff;
GRANT EXECUTE ON dbo.sp_GenerateBills TO BillingStaff;

-- IT staff can manage system objects
GRANT ALL ON SCHEMA::IT TO ITStaff;
GRANT SELECT ON dbo.PatientRecords TO ITStaff;  -- For troubleshooting

-- Compliance auditor can view audit logs
GRANT SELECT ON dbo.AuditLog TO ComplianceAuditor;
GRANT SELECT ON dbo.vw_PatientAccessReport TO ComplianceAuditor;
```

3. **Row-Level Security Implementation**
```sql
-- Create function to restrict data access by user department
CREATE FUNCTION dbo.fn_GetUserDepartment()
RETURNS INT
AS
BEGIN
    RETURN CASE USER_NAME()
        WHEN 'MEDCARE\HCP_Physicians' THEN 1
        WHEN 'MEDCARE\HCP_Nurses' THEN 2
        WHEN 'MEDCARE\HCP_Administrators' THEN 3
        ELSE 0
    END;
END;

-- Implement row-level security for patient records
CREATE SECURITY POLICY PatientAccessPolicy
ADD FILTER PREDICATE dbo.fn_securitypredicate(PatientDepartmentID)
ON dbo.PatientRecords
FOR ALL
WITH (STATE = ON);

-- More granular security function
CREATE FUNCTION dbo.fn_PatientAccessControl(@PatientID INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 
    FROM dbo.PatientDepartmentAssignment pa
    INNER JOIN dbo.PatientAccessLog pal ON pa.PatientID = pal.PatientID
    WHERE pa.PatientID = @PatientID
    AND pal.UserName = USER_NAME()
    AND (pal.AccessType = 'FULL' OR 
         (pal.AccessType = 'READ_ONLY' AND CONTEXT_INFO() = 'READONLY'))
);
```

4. **Comprehensive Auditing**
```sql
-- Implement audit trail for patient data access
CREATE TABLE dbo.PatientAccessLog (
    AccessID BIGINT IDENTITY(1,1) PRIMARY KEY,
    PatientID INT NOT NULL,
    UserName NVARCHAR(128) NOT NULL,
    AccessType NVARCHAR(20) NOT NULL, -- READ, WRITE, DELETE, EXPORT
    AccessTime DATETIME2 NOT NULL DEFAULT GETDATE(),
    ApplicationName NVARCHAR(128),
    HostName NVARCHAR(128),
    SessionID INT,
    ReasonCode NVARCHAR(50),  -- Emergency, Routine, etc.
    Department NVARCHAR(100),
    IsEmergencyAccess BIT NOT NULL DEFAULT 0
);

-- Create index for patient access queries
CREATE INDEX IX_PatientAccessLog_PatientTime ON dbo.PatientAccessLog(PatientID, AccessTime DESC);
CREATE INDEX IX_PatientAccessLog_UserTime ON dbo.PatientAccessLog(UserName, AccessTime DESC);
CREATE INDEX IX_PatientAccessLog_Emergency ON dbo.PatientAccessLog(IsEmergencyAccess, AccessTime DESC);

-- Audit trigger for patient record access
CREATE TRIGGER tr_Audit_PatientRecordAccess
ON dbo.PatientRecords
AFTER SELECT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Log access to sensitive patient data
    -- Only log access to specific fields (SSN, Date of Birth, etc.)
    INSERT INTO dbo.PatientAccessLog (
        PatientID, UserName, AccessType, ApplicationName, HostName, SessionID
    )
    SELECT 
        PatientID,
        SYSTEM_USER,
        'READ',
        APP_NAME(),
        HOST_NAME(),
        @@SPID
    FROM inserted
    WHERE DATEADD(HOUR, -1, GETDATE()) > (
        SELECT ISNULL(MAX(AccessTime), '1900-01-01')
        FROM dbo.PatientAccessLog
        WHERE PatientID = inserted.PatientID
        AND UserName = SYSTEM_USER
        AND AccessType = 'READ'
    );
END;
```

5. **Compliance Reporting**
```sql
-- Create compliance reports for HIPAA audit
CREATE PROCEDURE sp_HIPAAComplianceReport
    @StartDate DATETIME,
    @EndDate DATETIME,
    @ComplianceType NVARCHAR(50) = 'ALL'  -- ALL, ACCESS, MODIFICATION, EMERGENCY_ACCESS
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        pal.PatientID,
        p.PatientName,  -- Encrypted or hashed for privacy
        pal.UserName,
        pal.AccessType,
        pal.AccessTime,
        pal.ApplicationName,
        pal.HostName,
        pal.Department,
        pal.IsEmergencyAccess,
        al.EventType,
        al.EventTime as ModificationTime
    FROM dbo.PatientAccessLog pal
    INNER JOIN dbo.PatientRecords p ON pal.PatientID = p.PatientID
    LEFT JOIN dbo.AuditLog al ON pal.PatientID = CAST(al.RecordKey AS INT)
        AND al.TableName = 'PatientRecords'
        AND al.EventTime BETWEEN pal.AccessTime AND DATEADD(MINUTE, 5, pal.AccessTime)
    WHERE pal.AccessTime BETWEEN @StartDate AND @EndDate
    AND (@ComplianceType = 'ALL' OR 
         (@ComplianceType = 'ACCESS' AND pal.AccessType = 'READ') OR
         (@ComplianceType = 'MODIFICATION' AND al.EventType IN ('INSERT', 'UPDATE', 'DELETE')) OR
         (@ComplianceType = 'EMERGENCY_ACCESS' AND pal.IsEmergencyAccess = 1))
    ORDER BY pal.AccessTime DESC;
END;

-- Create patient data export audit
CREATE PROCEDURE sp_PatientDataExportAudit
    @PatientID INT,
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    SELECT 
        UserName,
        AccessTime,
        ApplicationName,
        HostName,
        ReasonCode,
        ExportMethod,  -- PDF, CSV, etc.
        RecordsExported,
        ExportDestination
    FROM dbo.PatientAccessLog
    WHERE PatientID = @PatientID
    AND AccessTime BETWEEN @StartDate AND @EndDate
    AND AccessType = 'EXPORT'
    ORDER BY AccessTime DESC;
END;
```

**Learning Outcome**: Implementing healthcare compliance with granular access controls, comprehensive auditing, and regulatory reporting capabilities

### Scenario 2: Financial Services Multi-Region Security

**Background**: GlobalFinance operates in 12 countries with different regulatory requirements (SOX, GDPR, PCI DSS). They need a unified security model that handles multi-currency, multi-regulatory, and cross-border data access requirements.

**Current Security Challenges**:
- Different regulatory requirements per region
- Complex permission model for financial transactions
- Real-time fraud detection and prevention
- Audit requirements for all financial activities
- Data residency requirements
- Integration with multiple banking systems

**DBA Security Solution**:

1. **Multi-Region Authentication**
```sql
-- Create region-specific login patterns
CREATE LOGIN [GLOBAL\FIN_US_Admins] FROM WINDOWS;
CREATE LOGIN [GLOBAL\FIN_EU_Admins] FROM WINDOWS;
CREATE LOGIN [GLOBAL\FIN_APAC_Admins] FROM WINDOWS;

-- Create region-specific database roles
CREATE ROLE US_FinancialStaff;
CREATE ROLE EU_FinancialStaff;
CREATE ROLE APAC_FinancialStaff;
CREATE ROLE GlobalAdmins;
CREATE ROLE Auditors;
CREATE ROLE ComplianceOfficers;

-- Map regions to roles
ALTER ROLE US_FinancialStaff ADD MEMBER [GLOBAL\FIN_US_*];
ALTER ROLE EU_FinancialStaff ADD MEMBER [GLOBAL\FIN_EU_*];
ALTER ROLE APAC_FinancialStaff ADD MEMBER [GLOBAL\FIN_APAC_*];
ALTER ROLE GlobalAdmins ADD MEMBER [GLOBAL\FIN_US_Admins, GLOBAL\FIN_EU_Admins, GLOBAL\FIN_APAC_Admins];
```

2. **Financial Transaction Security**
```sql
-- Create transaction security schema
CREATE SCHEMA FinancialSecurity AUTHORIZATION dbo;

-- Implement dual authorization for high-value transactions
CREATE TABLE dbo.DualAuthorization (
    TransactionID BIGINT PRIMARY KEY,
    PrimaryApprover NVARCHAR(128) NOT NULL,
    SecondaryApprover NVARCHAR(128) NULL,
    ApprovalLevel NVARCHAR(20) NOT NULL, -- STANDARD, SUPERVISOR, MANAGER
    TransactionAmount DECIMAL(18,2) NOT NULL,
    CurrencyCode NVARCHAR(3) NOT NULL,
    ApprovalStatus NVARCHAR(20) NOT NULL DEFAULT 'PENDING', -- PENDING, APPROVED, REJECTED
    PrimaryApprovalTime DATETIME2 NULL,
    SecondaryApprovalTime DATETIME2 NULL,
    Region NVARCHAR(3) NOT NULL, -- US, EU, APAC
    ComplianceFlags NVARCHAR(MAX), -- JSON for regulatory flags
    AuditTrail NVARCHAR(MAX)  -- JSON audit trail
);

-- Function to determine required approval level
CREATE FUNCTION dbo.fn_GetRequiredApprovalLevel(
    @TransactionAmount DECIMAL(18,2),
    @Region NVARCHAR(3),
    @TransactionType NVARCHAR(20)
)
RETURNS NVARCHAR(20)
AS
BEGIN
    DECLARE @RequiredLevel NVARCHAR(20) = 'STANDARD';
    
    -- Amount-based approval requirements
    IF @TransactionAmount > 1000000
        SET @RequiredLevel = 'MANAGER';
    ELSE IF @TransactionAmount > 100000
        SET @RequiredLevel = 'SUPERVISOR';
    
    -- Transaction type requirements
    IF @TransactionType IN ('WIRE_TRANSFER', 'INTERNATIONAL_TRANSFER')
        IF @RequiredLevel = 'STANDARD'
            SET @RequiredLevel = 'SUPERVISOR';
    
    -- Regional requirements
    IF @Region = 'EU' AND @TransactionAmount > 500000  -- PSD2 requirements
        SET @RequiredLevel = 'MANAGER';
    
    RETURN @RequiredLevel;
END;

-- Stored procedure for dual authorization
CREATE PROCEDURE FinancialSecurity.sp_ProcessTransactionWithApproval
    @TransactionID BIGINT,
    @PrimaryApprover NVARCHAR(128),
    @TransactionAmount DECIMAL(18,2),
    @CurrencyCode NVARCHAR(3),
    @TransactionType NVARCHAR(20),
    @Region NVARCHAR(3)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @RequiredLevel NVARCHAR(20);
    DECLARE @CurrentUser NVARCHAR(128) = SYSTEM_USER;
    
    -- Determine required approval level
    SET @RequiredLevel = dbo.fn_GetRequiredApprovalLevel(@TransactionAmount, @Region, @TransactionType);
    
    -- Record transaction for approval
    INSERT INTO dbo.DualAuthorization (
        TransactionID, PrimaryApprover, ApprovalLevel, 
        TransactionAmount, CurrencyCode, Region
    )
    VALUES (
        @TransactionID, @PrimaryApprover, @RequiredLevel,
        @TransactionAmount, @CurrencyCode, @Region
    );
    
    -- Log approval requirement
    INSERT INTO dbo.FinancialAuditLog (
        EventType, EventData, UserName, Region, ComplianceFlags
    )
    VALUES (
        'TRANSACTION_SUBMITTED',
        JSON_QUERY(
            OBJECT_QUERY(
                '{TransactionID: @TransactionID, RequiredLevel: @RequiredLevel, Amount: @Amount}',
                '$.TransactionID', @TransactionID,
                '$.RequiredLevel', @RequiredLevel,
                '$.Amount', @TransactionAmount
            )
        ),
        @CurrentUser,
        @Region,
        JSON_QUERY('{RequiresDualAuth: true, ComplianceRequired: true}')
    );
END;
```

3. **Fraud Detection Integration**
```sql
-- Create fraud detection audit table
CREATE TABLE dbo.FraudDetectionLog (
    AlertID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TransactionID BIGINT,
    CustomerID INT,
    AlertType NVARCHAR(50), -- UNUSUAL_AMOUNT, LOCATION_ANOMALY, VELOCITY_CHECK
    AlertScore DECIMAL(5,2), -- 0.00 to 10.00
    AlertDetails NVARCHAR(MAX), -- JSON details
    Status NVARCHAR(20) DEFAULT 'OPEN', -- OPEN, INVESTIGATED, FALSE_POSITIVE, CONFIRMED_FRAUD
    AssignedTo NVARCHAR(128),
    CreatedTime DATETIME2 NOT NULL DEFAULT GETDATE(),
    ResolvedTime DATETIME2 NULL,
    Resolution NVARCHAR(MAX)
);

-- Create indexes for fraud detection queries
CREATE INDEX IX_FraudDetection_TransactionID ON dbo.FraudDetectionLog(TransactionID);
CREATE INDEX IX_FraudDetection_CustomerID ON dbo.FraudDetectionLog(CustomerID);
CREATE INDEX IX_FraudDetection_Status ON dbo.FraudDetectionLog(Status, CreatedTime DESC);
CREATE INDEX IX_FraudDetection_Score ON dbo.FraudDetectionLog(AlertScore DESC, CreatedTime DESC);

-- Trigger for automatic fraud detection
CREATE TRIGGER tr_FraudDetection
ON dbo.Transactions
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @TransactionID BIGINT;
    DECLARE @CustomerID INT;
    DECLARE @Amount DECIMAL(18,2);
    DECLARE @TransactionType NVARCHAR(30);
    DECLARE @Location NVARCHAR(100);
    DECLARE @AlertScore DECIMAL(5,2) = 0;
    DECLARE @AlertType NVARCHAR(50);
    DECLARE @AlertDetails NVARCHAR(MAX);
    
    -- Process each inserted transaction
    DECLARE transaction_cursor CURSOR FOR
    SELECT TransactionID, CustomerID, Amount, TransactionType, Location
    FROM inserted;
    
    OPEN transaction_cursor;
    FETCH NEXT FROM transaction_cursor INTO @TransactionID, @CustomerID, @Amount, @TransactionType, @Location;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Check for unusual transaction amount
        IF @Amount > 500000  -- High-value transaction
        BEGIN
            SET @AlertScore = @AlertScore + 3;
            SET @AlertType = 'UNUSUAL_AMOUNT';
            
            SET @AlertDetails = JSON_QUERY(
                OBJECT_QUERY(
                    '{Threshold: 500000, ActualAmount: @Amount, RiskLevel: High}',
                    '$.ActualAmount', @Amount
                )
            );
        END
        
        -- Check transaction velocity (too many transactions in short time)
        IF EXISTS (
            SELECT 1 FROM dbo.Transactions
            WHERE CustomerID = @CustomerID
            AND TransactionDate > DATEADD(HOUR, -1, GETDATE())
            AND TransactionID != @TransactionID
        )
        BEGIN
            SET @AlertScore = @AlertScore + 2;
            IF @AlertType IS NULL
                SET @AlertType = 'VELOCITY_CHECK';
            ELSE
                SET @AlertType = @AlertType + ', VELOCITY_CHECK';
        END
        
        -- Insert fraud alert if score exceeds threshold
        IF @AlertScore >= 5
        BEGIN
            INSERT INTO dbo.FraudDetectionLog (
                TransactionID, CustomerID, AlertType, AlertScore, AlertDetails
            )
            VALUES (
                @TransactionID, @CustomerID, @AlertType, @AlertScore, @AlertDetails
            );
            
            -- Log security event
            INSERT INTO dbo.FinancialAuditLog (
                EventType, EventData, ComplianceFlags, SecurityLevel
            )
            VALUES (
                'FRAUD_ALERT',
                JSON_QUERY(
                    OBJECT_QUERY(
                        '{TransactionID: @TransactionID, CustomerID: @CustomerID, AlertScore: @AlertScore, AlertType: @AlertType}',
                        '$.TransactionID', @TransactionID,
                        '$.CustomerID', @CustomerID,
                        '$.AlertScore', @AlertScore,
                        '$.AlertType', @AlertType
                    )
                ),
                '{"PCI_DSS": true, "SOX": true, "GDPR": false}',
                'HIGH'
            );
        END
        
        FETCH NEXT FROM transaction_cursor INTO @TransactionID, @CustomerID, @Amount, @TransactionType, @Location;
    END
    
    CLOSE transaction_cursor;
    DEALLOCATE transaction_cursor;
END;
```

4. **Compliance and Regulatory Reporting**
```sql
-- SOX compliance report for financial controls
CREATE PROCEDURE sp_SOXComplianceReport
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    -- Control framework mapping
    SELECT 
        'SOX_Section_404' as ComplianceFramework,
        al.EventType as ControlActivity,
        COUNT(*) as ControlExecutions,
        SUM(CASE WHEN al.EventType = 'APPROVED' THEN 1 ELSE 0 END) as SuccessfulControls,
        SUM(CASE WHEN al.EventType = 'REJECTED' THEN 1 ELSE 0 END) as ControlExceptions,
        AVG(CASE WHEN al.EventType = 'APPROVED' 
            THEN CAST(JSON_VALUE(al.EventData, '$.ProcessingTime') AS DECIMAL(10,2))
            ELSE NULL END) as AverageProcessingTime
    FROM dbo.FinancialAuditLog al
    WHERE al.EventTime BETWEEN @StartDate AND @EndDate
    AND al.EventType IN ('TRANSACTION_SUBMITTED', 'APPROVED', 'REJECTED')
    GROUP BY al.EventType;
    
    -- PCI DSS compliance for credit card transactions
    SELECT 
        'PCI_DSS' as ComplianceFramework,
        COUNT(*) as TotalTransactions,
        SUM(CASE WHEN fal.AlertScore > 0 THEN 1 ELSE 0 END) as FraudAlerts,
        COUNT(DISTINCT CASE WHEN t.CardType IS NOT NULL THEN t.CardType END) as CardTypesProcessed,
        MAX(t.Amount) as MaximumTransactionAmount
    FROM dbo.Transactions t
    LEFT JOIN dbo.FraudDetectionLog fal ON t.TransactionID = fal.TransactionID
    WHERE t.TransactionDate BETWEEN @StartDate AND @EndDate
    GROUP BY 'PCI_DSS';
    
    -- GDPR compliance for EU customer data
    SELECT 
        'GDPR' as ComplianceFramework,
        al.UserName as DataController,
        COUNT(*) as DataAccessEvents,
        COUNT(DISTINCT al.EventData) as UniqueDataTypesAccessed,
        SUM(CASE WHEN al.ComplianceFlags LIKE '%EU_Resident%' THEN 1 ELSE 0 END) as EUDataAccesses
    FROM dbo.FinancialAuditLog al
    WHERE al.Region = 'EU'
    AND al.EventTime BETWEEN @StartDate AND @EndDate
    GROUP BY al.UserName;
END;
```

**Learning Outcome**: Implementing multi-region security with complex compliance requirements, fraud detection integration, and comprehensive regulatory reporting

### Scenario 3: E-commerce Security Hardening

**Background**: ShopEasy is an e-commerce platform handling 2 million customer transactions daily, requiring PCI DSS compliance, fraud prevention, and scalable security architecture.

**Security Requirements**:
- PCI DSS Level 1 compliance
- Real-time fraud detection
- Customer data protection
- API security for mobile applications
- Integration with payment gateways
- Scalable authentication for peak traffic (Black Friday)

**DBA Security Implementation**:

1. **PCI DSS Compliance Implementation**
```sql
-- Create PCI DSS compliant credit card data table
CREATE TABLE dbo.PaymentCards (
    CardID BIGINT IDENTITY(1,1) PRIMARY KEY,
    CardHolderID BIGINT NOT NULL, -- Encrypted reference to customer
    -- Store only tokenized card information
    CardToken NVARCHAR(128) NOT NULL,
    CardBrand NVARCHAR(20), -- VISA, MASTERCARD, etc.
    Last4Digits CHAR(4), -- Only last 4 digits (PCI DSS requirement)
    ExpirationMonth TINYINT,
    ExpirationYear SMALLINT,
    CardholderName NVARCHAR(100), -- May be encrypted
    BillingAddressID BIGINT, -- Reference to address table
    IsPrimary BIT DEFAULT 0,
    IsActive BIT DEFAULT 1,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETDATE,
    LastUsedDate DATETIME2 NULL
    -- NO FULL CREDIT CARD NUMBERS STORED
);

-- Create payment tokenization table
CREATE TABLE dbo.PaymentTokens (
    TokenID NVARCHAR(128) PRIMARY KEY,
    OriginalCardData NVARCHAR(MAX), -- Encrypted original card data
    TokenizationMethod NVARCHAR(50), -- HASH, TOKEN, etc.
    TokenizationProvider NVARCHAR(100), -- Payment gateway provider
    CreatedDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    ExpiresDate DATETIME2 NOT NULL,
    IsActive BIT DEFAULT 1,
    PCI_ComplianceLevel NVARCHAR(20) NOT NULL DEFAULT 'LEVEL_1'
);

-- Create sensitive data access log (PCI requirement)
CREATE TABLE dbo.PCIAccessLog (
    AccessID BIGINT IDENTITY(1,1) PRIMARY KEY,
    UserName NVARCHAR(128) NOT NULL,
    AccessType NVARCHAR(20) NOT NULL, -- VIEW, EXPORT, MODIFY
    CardholderDataAccessed NVARCHAR(500), -- What data was accessed
    AccessMethod NVARCHAR(100), -- Application, API, Report
    IPAddress NVARCHAR(45),
    UserAgent NVARCHAR(500),
    AccessTime DATETIME2 NOT NULL DEFAULT GETDATE(),
    ComplianceNote NVARCHAR(500),
    AuditRequired BIT DEFAULT 1
);

-- PCI DSS audit trigger for cardholder data
CREATE TRIGGER tr_PCI_CardholderDataAccess
ON dbo.PaymentCards
AFTER SELECT, INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AccessType NVARCHAR(20);
    
    -- Determine access type
    IF EXISTS (SELECT 1 FROM inserted) AND EXISTS (SELECT 1 FROM deleted)
        SET @AccessType = 'MODIFY';
    ELSE IF EXISTS (SELECT 1 FROM inserted)
        SET @AccessType = 'INSERT';
    ELSE IF EXISTS (SELECT 1 FROM deleted)
        SET @AccessType = 'DELETE';
    ELSE
        SET @AccessType = 'VIEW';
    
    -- Log PCI-sensitive data access
    INSERT INTO dbo.PCIAccessLog (
        UserName, AccessType, CardholderDataAccessed, 
        AccessMethod, ComplianceNote
    )
    SELECT 
        SYSTEM_USER,
        @AccessType,
        'PaymentCard: Token=' + CardToken + ', Last4=' + Last4Digits + ', Brand=' + CardBrand,
        APP_NAME(),
        'PCI DSS Compliance Log - Cardholder Data Access'
    FROM inserted
    WHERE CardToken IS NOT NULL;
END;
```

2. **API Security Implementation**
```sql
-- Create API key management for mobile apps
CREATE TABLE dbo.APIKeys (
    APIKeyID NVARCHAR(128) PRIMARY KEY,
    ApplicationName NVARCHAR(100) NOT NULL,
    ApplicationVersion NVARCHAR(20),
    CustomerID BIGINT NULL, -- NULL for anonymous access
    AllowedMethods NVARCHAR(100), -- GET, POST, PUT, DELETE
    RateLimitPerHour INT DEFAULT 1000,
    RateLimitPerDay INT DEFAULT 10000,
    IsActive BIT DEFAULT 1,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    LastUsedDate DATETIME2 NULL,
    IPWhitelist NVARCHAR(MAX), -- JSON array of allowed IPs
    PermissionScopes NVARCHAR(MAX), -- JSON permissions
    ExpiresDate DATETIME2 NULL,
    CreatedBy NVARCHAR(128) NOT NULL
);

-- Create API access log
CREATE TABLE dbo.APIAccessLog (
    LogID BIGINT IDENTITY(1,1) PRIMARY KEY,
    APIKeyID NVARCHAR(128) NOT NULL,
    RequestMethod NVARCHAR(10) NOT NULL,
    RequestPath NVARCHAR(500) NOT NULL,
    ResponseStatus INT NOT NULL,
    ResponseTimeMS INT,
    IPAddress NVARCHAR(45),
    UserAgent NVARCHAR(500),
    RequestSizeBytes INT,
    ResponseSizeBytes INT,
    RequestTime DATETIME2 NOT NULL DEFAULT GETDATE(),
    ErrorMessage NVARCHAR(MAX),
    RateLimited BIT DEFAULT 0,
    ComplianceFlags NVARCHAR(MAX) -- JSON compliance information
);

-- Create API rate limiting stored procedure
CREATE PROCEDURE sp_APIAccessControl
    @APIKeyID NVARCHAR(128),
    @RequestMethod NVARCHAR(10),
    @RequestPath NVARCHAR(500),
    @IPAddress NVARCHAR(45),
    @UserAgent NVARCHAR(500)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Allowed BIT = 0;
    DECLARE @RateLimitPerHour INT;
    DECLARE @RateLimitPerDay INT;
    DECLARE @ErrorMessage NVARCHAR(MAX);
    
    -- Validate API key
    SELECT 
        @RateLimitPerHour = RateLimitPerHour,
        @RateLimitPerDay = RateLimitPerDay
    FROM dbo.APIKeys
    WHERE APIKeyID = @APIKeyID 
    AND IsActive = 1
    AND (ExpiresDate IS NULL OR ExpiresDate > GETDATE());
    
    IF @RateLimitPerHour IS NULL
    BEGIN
        SET @ErrorMessage = 'Invalid or expired API key';
        SET @Allowed = 0;
    END
    ELSE
    BEGIN
        -- Check hourly rate limit
        DECLARE @HourlyCount INT;
        SELECT @HourlyCount = COUNT(*)
        FROM dbo.APIAccessLog
        WHERE APIKeyID = @APIKeyID
        AND RequestTime > DATEADD(HOUR, -1, GETDATE());
        
        IF @HourlyCount >= @RateLimitPerHour
        BEGIN
            SET @ErrorMessage = 'Hourly rate limit exceeded';
            SET @Allowed = 0;
        END
        ELSE
        BEGIN
            -- Check daily rate limit
            DECLARE @DailyCount INT;
            SELECT @DailyCount = COUNT(*)
            FROM dbo.APIAccessLog
            WHERE APIKeyID = @APIKeyID
            AND RequestTime > DATEADD(DAY, -1, GETDATE());
            
            IF @DailyCount >= @RateLimitPerDay
            BEGIN
                SET @ErrorMessage = 'Daily rate limit exceeded';
                SET @Allowed = 0;
            END
            ELSE
            BEGIN
                SET @Allowed = 1;
            END
        END
    END
    
    -- Log the API access attempt
    INSERT INTO dbo.APIAccessLog (
        APIKeyID, RequestMethod, RequestPath, ResponseStatus, 
        IPAddress, UserAgent, ErrorMessage, RateLimited
    )
    VALUES (
        @APIKeyID, @RequestMethod, @RequestPath,
        CASE WHEN @Allowed = 1 THEN 200 ELSE 429 END,
        @IPAddress, @UserAgent,
        @ErrorMessage,
        CASE WHEN @ErrorMessage LIKE '%rate limit%' THEN 1 ELSE 0 END
    );
    
    -- Log suspicious activity for fraud detection
    IF @IPAddress IS NOT NULL
    BEGIN
        -- Check for multiple failed attempts from same IP
        DECLARE @FailedAttempts INT;
        SELECT @FailedAttempts = COUNT(*)
        FROM dbo.APIAccessLog
        WHERE IPAddress = @IPAddress
        AND ResponseStatus >= 400
        AND RequestTime > DATEADD(MINUTE, -10, GETDATE());
        
        IF @FailedAttempts >= 5
        BEGIN
            -- Insert fraud detection alert
            INSERT INTO dbo.FraudDetectionLog (
                CustomerID, AlertType, AlertScore, AlertDetails
            )
            VALUES (
                NULL, -- No specific customer yet
                'SUSPICIOUS_API_ACTIVITY',
                8.0,
                JSON_QUERY(
                    OBJECT_QUERY(
                        '{IPAddress: @IP, FailedAttempts: @Attempts, TimeWindow: 10}',
                        '$.IP', @IPAddress,
                        '$.FailedAttempts', @FailedAttempts
                    )
                )
            );
        END
    END
    
    -- Return access control result
    SELECT 
        @Allowed as AccessGranted,
        @ErrorMessage as ErrorMessage,
        @RateLimitPerHour as RemainingHourlyRequests,
        @RateLimitPerDay as RemainingDailyRequests;
END;
```

3. **Customer Data Protection**
```sql
-- Create customer data access control
CREATE TABLE dbo.CustomerDataAccessLog (
    AccessID BIGINT IDENTITY(1,1) PRIMARY KEY,
    CustomerID BIGINT NOT NULL,
    AccessingUser NVARCHAR(128) NOT NULL,
    AccessType NVARCHAR(20) NOT NULL, -- VIEW, EXPORT, DELETE, UPDATE
    DataCategories NVARCHAR(500), -- Personal, Financial, Behavioral, etc.
    Purpose NVARCHAR(200), -- Business justification
    AccessMethod NVARCHAR(100), -- Web, MobileApp, API, AdminPanel
    IPAddress NVARCHAR(45),
    SessionID NVARCHAR(128),
    AccessTime DATETIME2 NOT NULL DEFAULT GETDATE(),
    GDPR_LegalBasis NVARCHAR(50), -- CONSENT, CONTRACT, LEGAL_OBLIGATION, etc.
    RetentionPeriod INT, -- Days to retain this access record
    AuditRequired BIT DEFAULT 1
);

-- Create data retention policies
CREATE TABLE dbo.DataRetentionPolicies (
    PolicyID INT IDENTITY(1,1) PRIMARY KEY,
    DataCategory NVARCHAR(100) NOT NULL, -- Personal, Financial, Transaction, etc.
    RetentionPeriodDays INT NOT NULL,
    DisposalMethod NVARCHAR(50), -- DELETE, ARCHIVE, ANONYMIZE
    LegalBasis NVARCHAR(100), -- GDPR Article, SOX Requirement, etc.
    IsActive BIT DEFAULT 1,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETDATE()
);

-- Insert standard retention policies
INSERT INTO dbo.DataRetentionPolicies (DataCategory, RetentionPeriodDays, DisposalMethod, LegalBasis)
VALUES 
    ('Customer_Personal_Data', 2555, 'DELETE', 'GDPR Article 5(1)(e) - Storage limitation'),
    ('Transaction_Data', 2555, 'ANONYMIZE', 'SOX Section 404 - Financial record retention'),
    ('API_Access_Logs', 90, 'DELETE', 'PCI DSS Requirement 10.7'),
    ('Fraud_Investigation_Data', 1825, 'ANONYMIZE', 'Legal requirements for fraud prevention'),
    ('Security_Logs', 365, 'DELETE', 'Industry best practices');

-- Create GDPR data access request handling
CREATE PROCEDURE sp_HandleGDPRRequest
    @CustomerID BIGINT,
    @RequestType NVARCHAR(20), -- ACCESS, RECTIFICATION, ERASURE, PORTABILITY
    @RequestDetails NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @RequestID BIGINT;
    DECLARE @GDPR_LegalBasis NVARCHAR(50);
    
    -- Determine legal basis for processing
    SET @GDPR_LegalBasis = CASE @RequestType
        WHEN 'ACCESS' THEN 'CONSENT'
        WHEN 'RECTIFICATION' THEN 'CONTRACT'
        WHEN 'ERASURE' THEN 'CONSENT_WITHDRAWAL'
        WHEN 'PORTABILITY' THEN 'CONTRACT'
        ELSE 'LEGAL_OBLIGATION'
    END;
    
    -- Create request record
    INSERT INTO dbo.GDPRRequests (
        CustomerID, RequestType, RequestDetails, LegalBasis, Status
    )
    VALUES (
        @CustomerID, @RequestType, @RequestDetails, @GDPR_LegalBasis, 'PENDING'
    );
    
    SET @RequestID = SCOPE_IDENTITY();
    
    -- Log the GDPR request for audit
    INSERT INTO dbo.CustomerDataAccessLog (
        CustomerID, AccessingUser, AccessType, DataCategories,
        Purpose, AccessMethod, GDPR_LegalBasis, RetentionPeriod
    )
    VALUES (
        @CustomerID, SYSTEM_USER, @RequestType,
        'Personal_Data_Export',
        'GDPR Data Subject Rights Request',
        'Admin_Portal',
        @GDPR_LegalBasis,
        2555 -- 7 years for legal compliance
    );
    
    -- Trigger appropriate action based on request type
    IF @RequestType = 'ACCESS'
    BEGIN
        -- Export customer data
        SELECT @RequestID as RequestID,
               (SELECT * FROM dbo.Customers WHERE CustomerID = @CustomerID FOR JSON PATH) as CustomerData,
               (SELECT * FROM dbo.Transactions WHERE CustomerID = @CustomerID FOR JSON PATH) as TransactionHistory,
               (SELECT * FROM dbo.PaymentCards WHERE CardHolderID = @CustomerID FOR JSON PATH) as PaymentData;
    END
    ELSE IF @RequestType = 'ERASURE'
    BEGIN
        -- Implement right to be forgotten
        -- Note: This is complex and may require multiple steps
        -- and legal review for each data type
        
        -- Mark customer for deletion
        UPDATE dbo.Customers
        SET IsDeleted = 1,
            DeletionRequestDate = GETDATE()
        WHERE CustomerID = @CustomerID;
        
        -- Schedule data deletion based on retention policies
        INSERT INTO dbo.DataDeletionSchedule (
            CustomerID, DataCategory, ScheduledDeletionDate, RequestID
        )
        SELECT 
            @CustomerID,
            drp.DataCategory,
            DATEADD(DAY, drp.RetentionPeriodDays, GETDATE()),
            @RequestID
        FROM dbo.DataRetentionPolicies drp
        WHERE drp.IsActive = 1
        AND drp.DisposalMethod = 'DELETE';
    END
    
    SELECT @RequestID as RequestID, @RequestType as RequestType;
END;
```

**Learning Outcome**: Implementing e-commerce security with PCI DSS compliance, API security, customer data protection, and GDPR compliance

## Summary and Key Takeaways

This week provided comprehensive coverage of SQL Server security and permissions management:

1. **Security Architecture**: Understanding SQL Server's layered security model and defense-in-depth approach
2. **Authentication Methods**: Windows Authentication, SQL Server Authentication, and Azure AD integration
3. **Authorization Models**: Role-based security, permissions management, and ownership chains
4. **Advanced Security Features**: TDE, Always Encrypted, Row-Level Security, and Dynamic Data Masking
5. **Service Account Security**: Proper configuration of service accounts for production environments
6. **Auditing and Compliance**: Comprehensive auditing for SOX, HIPAA, PCI DSS, and GDPR requirements

Key security principles:
- **Principle of Least Privilege**: Grant only the minimum necessary permissions
- **Defense in Depth**: Multiple layers of security controls
- **Regular Security Reviews**: Periodic assessment of permissions and access
- **Monitoring and Alerting**: Continuous monitoring for security events
- **Compliance Integration**: Embed compliance requirements into security design
- **Documentation and Training**: Maintain documentation and provide security training

Security is not a one-time setup but an ongoing process that requires constant vigilance and adaptation to new threats and regulatory requirements.

## Additional Resources

### Microsoft Documentation
- SQL Server Security Best Practices: https://docs.microsoft.com/sql/sql-server/security/sql-server-security-best-practices
- Authentication in SQL Server: https://docs.microsoft.com/sql/relational-databases/security/authentication-access/sql-server-authentication-mode
- Encryption in SQL Server: https://docs.microsoft.com/sql/relational-databases/security/encryption/sql-server-encryption

### Compliance Resources
- PCI DSS Requirements: https://www.pcisecuritystandards.org/
- GDPR Compliance Guide: https://gdpr.eu/
- SOX Compliance Framework: https://www.sox.com/

### Security Tools
- SQL Server Audit: Built-in auditing capabilities
- Extended Events: Advanced monitoring and auditing
- SQL Vulnerability Assessment: Automated security scanning
- Advanced Threat Protection: Real-time threat detection

---

**Next Week Preview**: Backup and Recovery Basics - We'll explore backup strategies, recovery models, backup media management, and disaster recovery planning for SQL Server environments.