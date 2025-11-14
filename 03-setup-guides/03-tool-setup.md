# SQL Server Tool Setup and Configuration Guide

## Table of Contents
1. [SSMS Advanced Configuration](#ssms-advanced)
2. [Visual Studio and SQL Server Data Tools](#vs-ssdt)
3. [Additional Development Tools](#additional-tools)
4. [Database Design and Modeling Tools](#design-tools)
5. [Performance Monitoring Tools](#performance-tools)
6. [Database Administration Tools](#admin-tools)
7. [Source Control Integration](#source-control)
8. [IDE Customization and Templates](#ide-customization)
9. [Automation and Scripting Tools](#automation-tools)

## SSMS Advanced Configuration {#ssms-advanced}

### Step 1: SSMS Interface Optimization

**Complete SSMS Settings Configuration:**
```
1. Launch SQL Server Management Studio
2. Go to Tools > Options
3. Configure each section as follows:
```

**General Settings:**
```sql
-- Import/export settings
-- Save SSMS settings to file for backup:
-- Tools > Import and Export Settings > Export settings
-- Location: C:\SSMSSettings\SSMS_Default.ssmssettings

-- Restore settings:
-- Tools > Import and Export Settings > Import settings
```

**Environment Configuration:**
```
Options > Environment > General:
- Environment Layout: Tabbed Documents
- Show startup page: Blank environment
- At startup: Open Object Explorer
- Reset window layout: Clear all toolbars
```

**Editor Configuration:**
```
Options > Text Editor > All Languages > Tabs:
- Indenting: Smart
- Tab size: 4
- Indent size: 4
- Insert spaces: Checked
- Keep tabs: Unchecked

Options > Text Editor > Transact-SQL > General:
- List members: Checked
- Parameter information: Checked
- Auto list members: Checked
- Hide advanced members: Unchecked
```

**Query Results Configuration:**
```
Options > Query Results > SQL Server > Results to Grid:
- Display results in a separate tab: Checked
- Switch to results tab after query executes: Checked
- Include column headers when copying or saving results: Checked
- Save query results file as: Unicode (UTF-8)

Options > Query Results > SQL Server > Results to Text:
- Output format: Delimited by comma
- Include column headers in the result set: Checked
- Right align numeric values: Unchecked
```

### Step 2: SSMS Custom Templates

**Create Custom Template Library:**
```sql
-- Create template directory structure
-- Location: %AppData%\Microsoft\SQL Server Management Studio\CurrentVersion\Templates\Script\SQL

-- 1. Common Table Scripts
-- Templates\SQL\Tables\CreateTable.sql
CREATE TABLE [dbo].[TableName]
(
    [Column1] INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [Column2] NVARCHAR(100) NULL,
    [Column3] DATETIME2 NOT NULL DEFAULT GETDATE(),
    [CreatedBy] NVARCHAR(50) NOT NULL DEFAULT SUSER_SNAME(),
    [ModifiedBy] NVARCHAR(50) NULL,
    [ModifiedDate] DATETIME2 NULL,
    [IsActive] BIT NOT NULL DEFAULT 1
)
ON [PRIMARY];

-- 2. Index Templates
-- Templates\SQL\Indexes\CreateIndexWithInclude.sql
CREATE NONCLUSTERED INDEX [IX_TableName_ColumnName]
ON [dbo].[TableName] ([ColumnName])
INCLUDE ([Column1], [Column2], [Column3])
WITH (ONLINE = ON, SORT_IN_TEMPDB = ON, MAXDOP = 0);

-- 3. Stored Procedure Templates
-- Templates\SQL\Stored Procedures\BasicCRUD.sql
CREATE PROCEDURE [dbo].[sp_TableName_Insert]
    @Column1 NVARCHAR(100),
    @Column2 INT = NULL,
    @Column3 DATETIME2 = NULL,
    @CreatedBy NVARCHAR(50) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        INSERT INTO [dbo].[TableName] 
            ([Column1], [Column2], [Column3], [CreatedBy])
        VALUES 
            (@Column1, @Column2, @Column3, ISNULL(@CreatedBy, SUSER_SNAME()));
        
        SELECT SCOPE_IDENTITY() AS NewID;
    END TRY
    BEGIN CATCH
        THROW;
    END CATCH
END;
```

### Step 3: SSMS Keyboard Shortcuts

**Custom Keyboard Mapping:**
```
Go to Tools > Options > Environment > Keyboard

Configure these essential shortcuts:

Standard Editing:
- Comment Selection: Ctrl+K, Ctrl+C
- Uncomment Selection: Ctrl+K, Ctrl+U
- Increase Line Indent: Tab
- Decrease Line Indent: Shift+Tab
- Make Selection Uppercase: Ctrl+Shift+U
- Make Selection Lowercase: Ctrl+Shift+L

Query Execution:
- Execute: F5
- Execute with Estimates Only: Ctrl+L
- Parse: Ctrl+Shift+P
- Cancel Executing Query: Alt+Break

Navigation:
- Find Next: F3
- Find Previous: Shift+F3
- Bookmark Toggle: Ctrl+K, Ctrl+K
- Go to Next Bookmark: Ctrl+K, Ctrl+N
- Go to Previous Bookmark: Ctrl+K, Ctrl+P

Database Objects:
- Generate Script: Alt+F1 (configure in Tools > Options)
- New Query Window: Ctrl+N
- Refresh: F5
```

### Step 4: SSMS Color Themes

**Create Custom Theme Configuration:**
```xml
<!-- Custom SSMS Theme Configuration -->
<!-- Location: %AppData%\Microsoft\SQL Server Management Studio\CurrentVersion\Themes\ -->

<!-- Create file: CustomDarkTheme.vstheme -->
<?xml version="1.0" encoding="utf-8"?>
<Themes>
  <Theme Name="CustomDark" GUID="{12345678-1234-1234-1234-123456789012}">
    <Category Name="Text Editor" GUID="{A471F74C-B6B6-4A5E-B4A8-C4F5E8B12345}">
      <Property Name="Default" Foreground="#D4D4D4" Background="#1E1E1E" />
      <Property Name="Keyword" Foreground="#569CD6" />
      <Property Name="String" Foreground="#CE9178" />
      <Property Name="Comment" Foreground="#6A9955" />
      <Property Name="Number" Foreground="#B5CEA8" />
      <Property Name="Operator" Foreground="#D4D4D4" />
      <Property Name="Identifier" Foreground="#9CDCFE" />
      <Property Name="Preprocessor" Foreground="#C586C0" />
    </Category>
    <Category Name="SQL Editor" GUID="{B582F85C-B7C7-4D6F-A5F7-E9F6F9C23456}">
      <Property Name="Batch Separator" Foreground="#D4D4D4" />
      <Property Name="Comment" Foreground="#6A9955" />
      <Property Name="Function Name" Foreground="#DCDCAA" />
      <Property Name="Keyword" Foreground="#569CD6" />
      <Property Name="Line Number" Foreground="#858585" />
      <Property Name="Number" Foreground="#B5CEA8" />
      <Property Name="Operator" Foreground="#D4D4D4" />
      <Property Name="SQL String" Foreground="#CE9178" />
      <Property Name="Syntax Error" Foreground="#F44747" />
    </Category>
  </Theme>
</Themes>
```

## Visual Studio and SQL Server Data Tools {#vs-ssdt}

### Step 1: SSDT Installation and Setup

**Download and Install SSDT:**
```
1. Download Visual Studio with SSDT workload:
   - Visual Studio 2022 Community/Professional/Enterprise
   - Include "Data storage and processing" workload
   - Include "SQL Server Data Tools" component

2. Download standalone SSDT:
   - SSDT-Setup.exe from Microsoft
   - Version 17.9 or later for SQL Server 2022 support

3. Install Analysis Services:
   - During installation, select Analysis Services tools
   - Include Reporting Services project templates
```

**Configure SSDT Project Templates:**
```xml
<!-- Project Template Configuration -->
<!-- Location: %USERPROFILE%\Documents\Visual Studio 2022\Templates\ProjectTemplates\ -->

<!-- Create: SQL Server Database Project -->
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <Configuration Condition=" '$(Configuration)' == '' ">Debug</Configuration>
    <Platform Condition=" '$(Platform)' == '' ">AnyCPU</Platform>
    <Name>DatabaseProject</Name>
    <ProjectTypeGuids>{2150E333-8FDC-42A3-9474-1A3956D46DE8};{91A77BA4-6A45-4BEA-B86F-D33EC71C8ECB}</ProjectTypeGuids>
    <TargetDatabaseName>$(ProjectName)</TargetDatabaseName>
  </PropertyGroup>
  <PropertyGroup Condition="'$(Configuration)|$(Platform)' == 'Debug|AnyCPU'">
    <OutputPath>bin\Debug\</OutputPath>
    <TargetDatabaseName>$(ProjectName)_Debug</TargetDatabaseName>
  </PropertyGroup>
  <PropertyGroup Condition="'$(Configuration)|$(Platform)' == 'Release|AnyCPU'">
    <OutputPath>bin\Release\</OutputPath>
    <TargetDatabaseName>$(ProjectName)_Production</TargetDatabaseName>
  </PropertyGroup>
</Project>
```

### Step 2: Database Project Structure

**Standard Project Structure:**
```
DatabaseProject/
├── dacpac_files/          # Deployment packages
├── Scripts/              # Database scripts
│   ├── PreDeployment/    # Pre-deployment scripts
│   ├── PostDeployment/   # Post-deployment scripts
│   └── Migrations/       # Version-controlled changes
├── Schema Objects/       # Database objects
│   ├── Tables/          # Table definitions
│   ├── Views/           # View definitions
│   ├── Stored Procedures/ # Stored procedures
│   ├── Functions/       # User-defined functions
│   ├── Security/        # Users, roles, permissions
│   └── Types/           # Custom data types
└── Tests/               # Database unit tests
    ├── Test Data/       # Test data scripts
    └── Test Suites/     # Test execution scripts
```

**Example Database Project Creation Script:**
```bash
# Create new database project using sqlpackage
sqlpackage /Action:Create /TargetFile:DatabaseProject.dacpac /Properties:DatabaseName=MyDatabase

# Publish database project
sqlpackage /Action:Publish /TargetFile:DatabaseProject.dacpac /TargetServerName:localhost /TargetDatabaseName:MyDatabase

# Extract existing database to project
sqlpackage /Action:Extract /SourceServerName:localhost /SourceDatabaseName:ExistingDB /TargetFile:DatabaseProject.dacpac
```

### Step 3: SSDT Code Analysis Rules

**Configure Code Analysis Rules:**
```xml
<!-- Create Custom Code Analysis Rules -->
<!-- Location: DatabaseProject\Properties\CustomCodeAnalysisRules.xml -->

<CodeAnalysisRuleSet Name="CustomDatabaseRules">
  <Rule HintLevel="Error">
    <RuleId>SR0001</RuleId>
    <Description>Tables must have primary key</Description>
    <RuleExpression>NOT EXISTS (SELECT 1 FROM sys.tables t INNER JOIN sys.indexes i ON t.object_id = i.object_id WHERE t.is_ms_shipped = 0 AND i.type = 1 AND t.object_id = OBJECT_ID('$(TableName)'))</RuleExpression>
    <Severity>Error</Severity>
  </Rule>
  
  <Rule HintLevel="Warning">
    <RuleId>SR0002</RuleId>
    <Description>Indexes should be named with consistent convention</Description>
    <RuleExpression>NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name NOT LIKE 'IX_%' AND name NOT LIKE 'PK_%' AND name NOT LIKE 'UQ_%')</RuleExpression>
    <Severity>Warning</Severity>
  </Rule>
  
  <Rule HintLevel="Information">
    <RuleId>SR0003</RuleId>
    <Description>Tables should have audit columns</Description>
    <RuleExpression>NOT EXISTS (SELECT 1 FROM sys.columns WHERE column_id IN (SELECT column_id FROM sys.columns WHERE object_id = OBJECT_ID('$(TableName)') AND name LIKE '%Created%' OR name LIKE '%Modified%' OR name LIKE '%Date%'))</RuleExpression>
    <Severity>Information</Severity>
  </Rule>
</CodeAnalysisRuleSet>
```

### Step 4: Continuous Integration Setup

**Create Build and Deployment Scripts:**
```xml
<!-- Create: DatabaseProject.Build.proj -->
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003" DefaultTargets="Build">
  <PropertyGroup>
    <Configuration Condition="'$(Configuration)' == ''">Debug</Configuration>
    <DatabaseProjectFile>DatabaseProject.sqlproj</DatabaseProjectFile>
    <OutputDacpacFile>$(Configuration)\DatabaseProject.dacpac</OutputDacpacFile>
    <PublishProfileFile>$(Configuration)\Database.publish.xml</PublishProfileFile>
  </PropertyGroup>
  
  <Target Name="Clean">
    <RemoveDir Directories="$(Configuration)" />
    <RemoveDir Directories="bin" />
    <RemoveDir Directories="obj" />
  </Target>
  
  <Target Name="Build">
    <MSBuild Projects="$(DatabaseProjectFile)" Properties="Configuration=$(Configuration);Platform=AnyCPU" />
  </Target>
  
  <Target Name="Publish" DependsOnTargets="Build">
    <Message Text="Publishing database project to $(PublishProfileFile)" Importance="high" />
    <Exec Command="sqlpackage /Action:Publish /SourceFile:&quot;$(OutputDacpacFile)&quot; /Profile:&quot;$(PublishProfileFile)&quot;" />
  </Target>
  
  <Target Name="Test" DependsOnTargets="Build">
    <Message Text="Running database unit tests" Importance="high" />
    <Exec Command="vstest.con.exe Tests\DatabaseProject.Tests.dll /TestAdapterPath:bin\$(Configuration)\Extensions\" />
  </Target>
</Project>
```

## Additional Development Tools {#additional-tools}

### Step 1: PowerShell and SQL Server Module

**Install SQL Server PowerShell Module:**
```powershell
# Install SqlServer module
Install-Module -Name SqlServer -Force -AllowClobber

# Update to latest version
Update-Module -Name SqlServer

# List available commands
Get-Command -Module SqlServer | Select-Object Name, ModuleName | Sort-Object Name
```

**SQL Server PowerShell Functions:**
```powershell
# Create comprehensive SQL Server management functions
function Connect-SqlServer {
    param(
        [Parameter(Mandatory=$true)]
        [string]$ServerName,
        
        [Parameter(Mandatory=$false)]
        [string]$DatabaseName = "master",
        
        [Parameter(Mandatory=$false)]
        [PSCredential]$Credential
    )
    
    try {
        if ($Credential) {
            $connection = New-Object Microsoft.SqlServer.Management.Common.ServerConnection($ServerName, $Credential.UserName, $Credential.GetNetworkCredential().Password)
        } else {
            $connection = New-Object Microsoft.SqlServer.Management.Common.ServerConnection($ServerName)
        }
        
        $connection.Connect()
        $server = New-Object Microsoft.SqlServer.Management.Smo.Server($connection)
        Write-Host "Connected to $ServerName successfully" -ForegroundColor Green
        
        return $server
    }
    catch {
        Write-Error "Failed to connect to $ServerName : $($_.Exception.Message)"
        return $null
    }
}

function Backup-SqlDatabase {
    param(
        [Parameter(Mandatory=$true)]
        [Microsoft.SqlServer.Management.Smo.Server]$Server,
        
        [Parameter(Mandatory=$true)]
        [string]$DatabaseName,
        
        [Parameter(Mandatory=$true)]
        [string]$BackupPath,
        
        [Parameter(Mandatory=$false)]
        [switch]$Compress,
        
        [Parameter(Mandatory=$false)]
        [switch]$Encrypt
    )
    
    try {
        $database = $Server.Databases[$DatabaseName]
        if (-not $database) {
            throw "Database $DatabaseName not found"
        }
        
        $backup = New-Object Microsoft.SqlServer.Management.Smo.Backup
        $backup.Action = "Database"
        $backup.Database = $DatabaseName
        $backup.Devices.AddDevice($BackupPath, "File")
        
        if ($Compress) {
            $backup.CompressionOption = "On"
        }
        
        if ($Encrypt) {
            $backup.EncryptionOption = "On"
        }
        
        $backup.SqlBackup($Server)
        Write-Host "Backup completed: $BackupPath" -ForegroundColor Green
    }
    catch {
        Write-Error "Backup failed: $($_.Exception.Message)"
    }
}

function Restore-SqlDatabase {
    param(
        [Parameter(Mandatory=$true)]
        [Microsoft.SqlServer.Management.Smo.Server]$Server,
        
        [Parameter(Mandatory=$true)]
        [string]$DatabaseName,
        
        [Parameter(Mandatory=$true)]
        [string]$BackupPath,
        
        [Parameter(Mandatory=$false)]
        [string]$DataFilePath,
        
        [Parameter(Mandatory=$false)]
        [string]$LogFilePath,
        
        [Parameter(Mandatory=$false)]
        [switch]$Overwrite,
        
        [Parameter(Mandatory=$false)]
        [switch]$Verify
    )
    
    try {
        $restore = New-Object Microsoft.SqlServer.Management.Smo.Restore
        $restore.Action = "Database"
        $restore.Database = $DatabaseName
        $restore.Devices.AddDevice($BackupPath, "File")
        
        if ($DataFilePath) {
            $restore.RelocateFiles.Add($DatabaseName, $DataFilePath)
        }
        
        if ($LogFilePath) {
            $restore.RelocateFiles.Add($DatabaseName + "_Log", $LogFilePath)
        }
        
        if ($Overwrite) {
            $restore.ReplaceDatabase = $true
        }
        
        $restore.SqlRestore($Server)
        
        if ($Verify) {
            $verify = New-Object Microsoft.SqlServer.Management.Smo.Restore
            $verify.Devices.AddDevice($BackupPath, "File")
            $verify.CheckReadOnly()
            if ($verify.SqlVerify($Server)) {
                Write-Host "Backup file verified successfully" -ForegroundColor Green
            }
        }
        
        Write-Host "Database $DatabaseName restored from $BackupPath" -ForegroundColor Green
    }
    catch {
        Write-Error "Restore failed: $($_.Exception.Message)"
    }
}
```

### Step 2: Azure Data Studio Configuration

**Install and Configure Azure Data Studio:**
```json
// Create Azure Data Studio settings
// Location: %AppData%\Roaming\Azure Data Studio\User\settings.json

{
    "mssql.connections": [
        {
            "server": "localhost",
            "database": "",
            "authenticationType": "SqlLogin",
            "emptyPasswordInput": false,
            "savePassword": true,
            "profileName": "Local SQL Server",
            "group": "Development",
            "databaseProfile": {
                "name": "master"
            }
        }
    ],
    
    "files.autoSave": "afterDelay",
    "files.autoSaveDelay": 1000,
    "workbench.startupEditor": "welcomePage",
    
    "mssql.maxRecentConnections": 20,
    "mssql.recentConnections": [
        {
            "server": "localhost\\SQLEXPRESS",
            "serverVersion": "15.00.2000",
            "serverEdition": "Developer Edition",
            "isDocker": false,
            "connectionTimeout": 15,
            "queryTimeout": 30,
            "savePassword": false,
            "emptyPasswordInput": false,
            "authenticationType": "SqlLogin",
            "user": "sa",
            "database": "",
            "profileName": "Local Express Instance"
        }
    ],
    
    "mssql.editor.saveResultsToFile": true,
    "mssql.editor.format.queryExecuteOnMultipleQueries": true,
    "mssql.editor.format.saveQueryResultsToFile": true
}
```

**Azure Data Studio Extensions:**
```bash
# Install essential extensions
code --install-extension ms-mssql.mssql
code --install-extension Microsoft.vscode-json
code --install-extension ms-vscode.vscode-json
code --install-extension ms-vscode.vscode-powershell
code --install-extension ms-vscode-remote.remote-containers
code --install-extension ms-vscode.vscode-git-base
code --install-extension ms-toolsai.jupyter
code --install-extension ms-azuretools.vscode-docker
code --install-extension ms-vscode.azure-resource-tools
```

### Step 3: Third-Party Tools Installation

**Install Redgate SQL Toolbelt:**
```powershell
# Download and install Redgate SQL Toolbelt
$DownloadUrl = "https://download.red-gate.com/dispensable/SQLToolbelt/SQLToolbelt.exe"
$OutputPath = "$env:TEMP\SQLToolbelt.exe"

# Download installer
Invoke-WebRequest -Uri $DownloadUrl -OutFile $OutputPath

# Silent install
$Process = Start-Process -FilePath $OutputPath -ArgumentList "/SILENT" -Wait -PassThru

# Verify installation
$RedgateTools = @("SQL Prompt", "SQL Source Control", "SQL Compare", "SQL Data Compare", "SQL Monitor")
foreach ($Tool in $RedgateTools) {
    $Check = Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | 
             Where-Object {$_.DisplayName -like "*$Tool*"}
    if ($Check) {
        Write-Host "$Tool installed successfully" -ForegroundColor Green
    } else {
        Write-Host "$Tool not found" -ForegroundColor Yellow
    }
}
```

**Install ApexSQL Tools:**
```powershell
# Download ApexSQL Complete
$DownloadUrl = "https://www.apexsql.com/ftp/apexsql-complete.msi"
$OutputPath = "$env:TEMP\apexsql-complete.msi"

Invoke-WebRequest -Uri $DownloadUrl -OutFile $OutputPath

# Silent install
Start-Process -FilePath "msiexec.exe" -ArgumentList "/i `"$OutputPath`" /quiet" -Wait
```

## Database Design and Modeling Tools {#design-tools}

### Step 1: SQL Server Data Modeling

**Install and Configure Visio Database Templates:**
```
1. Install Microsoft Visio with Database Model Diagram template
2. Enable Database Model Diagram Add-in
3. Configure reverse engineering options:
   - Connection String: Server=(local);Database=master;Trusted_Connection=Yes
   - SQL Authentication: Available for non-domain environments
```

**Create Database Documentation with Visio:**
```xml
<!-- Database documentation template structure -->
<DatabaseDocumentation>
    <Database name="ProductionDB" version="1.0">
        <Overview>
            <Purpose>Main production database for ERP system</Purpose>
            <DataOwner>IT Department</DataOwner>
            <BusinessOwner>Operations</BusinessOwner>
            <Compliance>SOX, GDPR</Compliance>
        </Overview>
        
        <Security>
            <Authentication>Windows Authentication + SQL Authentication</Authentication>
            <Encryption>TDE enabled</Encryption>
            <BackupEncryption>AES 256</BackupEncryption>
        </Security>
        
        <Performance>
            <ExpectedUsers>500</ExpectedUsers>
            <ExpectedTransactions>10000/hour</ExpectedTransactions>
            <RecoveryTimeObjective>4 hours</RecoveryTimeObjective>
            <RecoveryPointObjective>15 minutes</RecoveryPointObjective>
        </Performance>
        
        <Schemas>
            <Schema name="dbo" description="Default schema for production tables"/>
            <Schema name="archive" description="Historical data archive"/>
            <Schema name="etl" description="ETL process staging tables"/>
            <Schema name="reports" description="Reporting and analytics views"/>
        </Schemas>
        
        <Tables>
            <Table name="Users" priority="Critical" size="Medium">
                <Description>Application user accounts and profiles</Description>
                <Rows>~50,000</Rows>
                <GrowthRate>10% annually</GrowthRate>
            </Table>
            <Table name="Transactions" priority="Critical" size="Large">
                <Description>Financial transaction records</Description>
                <Rows>~10,000,000</Rows>
                <GrowthRate>25% annually</GrowthRate>
            </Table>
        </Tables>
    </Database>
</DatabaseDocumentation>
```

### Step 2: ERD Generation and Maintenance

**Create ERD Script for SQL Server:**
```sql
-- Generate Entity Relationship Diagram data
SELECT 
    t.name AS TableName,
    c.name AS ColumnName,
    c.column_id,
    ty.name AS DataType,
    c.max_length,
    c.precision,
    c.scale,
    c.is_nullable,
    c.is_identity,
    ISNULL(dc.definition, '') AS DefaultConstraint,
    ISNULL(fk.name, '') AS ForeignKeyTable,
    ISNULL(fkc.name, '') AS ForeignKeyColumn,
    i.name AS PrimaryKey,
    i.is_primary_key,
    i.is_unique,
    i.is_unique_constraint
FROM sys.tables t
INNER JOIN sys.columns c ON t.object_id = c.object_id
INNER JOIN sys.types ty ON c.user_type_id = ty.user_type_id
LEFT JOIN sys.default_constraints dc ON c.default_object_id = dc.object_id
LEFT JOIN sys.foreign_keys fk ON c.object_id = fk.parent_object_id
LEFT JOIN sys.foreign_key_columns fkc ON fk.object_id = fkc.constraint_object_id
LEFT JOIN sys.indexes i ON t.object_id = i.object_id
LEFT JOIN sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id AND c.column_id = ic.column_id
WHERE t.is_ms_shipped = 0
ORDER BY t.name, c.column_id;
```

### Step 3: Database Documentation Tools

**Create Documentation Database:**
```sql
-- Create database documentation system
USE master;
GO

CREATE DATABASE DocumentationDB;
GO

USE DocumentationDB;
GO

-- Create documentation tables
CREATE TABLE DatabaseDocumentation (
    DocumentationID INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(128) NOT NULL,
    ObjectType NVARCHAR(50) NOT NULL, -- Table, View, Procedure, Function
    ObjectName NVARCHAR(128) NOT NULL,
    Description NVARCHAR(MAX),
    BusinessPurpose NVARCHAR(MAX),
    DataOwner NVARCHAR(100),
    LastUpdated DATETIME2 DEFAULT GETDATE(),
    UpdatedBy NVARCHAR(100) DEFAULT SUSER_SNAME(),
    Compliance NVARCHAR(200),
    SecurityClass NVARCHAR(50), -- Public, Internal, Confidential, Restricted
    RetentionPolicy NVARCHAR(100),
    ArchiveLocation NVARCHAR(500),
    LastReviewDate DATETIME2,
    NextReviewDate DATETIME2,
    CONSTRAINT UQ_DatabaseDocumentation UNIQUE (DatabaseName, ObjectType, ObjectName)
);

CREATE TABLE ColumnDocumentation (
    ColumnDocID INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(128) NOT NULL,
    TableName NVARCHAR(128) NOT NULL,
    ColumnName NVARCHAR(128) NOT NULL,
    BusinessName NVARCHAR(128),
    Description NVARCHAR(MAX),
    DataDomain NVARCHAR(100),
    DataQualityRules NVARCHAR(MAX),
    CalculationLogic NVARCHAR(MAX),
    BusinessValidation NVARCHAR(MAX),
    LastUpdated DATETIME2 DEFAULT GETDATE(),
    UpdatedBy NVARCHAR(100) DEFAULT SUSER_SNAME(),
    CONSTRAINT UQ_ColumnDocumentation UNIQUE (DatabaseName, TableName, ColumnName)
);

CREATE TABLE DataLineage (
    LineageID INT IDENTITY(1,1) PRIMARY KEY,
    SourceDatabase NVARCHAR(128),
    SourceTable NVARCHAR(128),
    SourceColumn NVARCHAR(128),
    TargetDatabase NVARCHAR(128),
    TargetTable NVARCHAR(128),
    TargetColumn NVARCHAR(128),
    TransformationLogic NVARCHAR(MAX),
    ETLProcess NVARCHAR(128),
    LastExecuted DATETIME2,
    SuccessRate DECIMAL(5,2),
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    CreatedBy NVARCHAR(100) DEFAULT SUSER_SNAME()
);
```

**Documentation Management Procedures:**
```sql
-- Create procedure to populate documentation from existing databases
CREATE PROCEDURE sp_GenerateDocumentation
    @DatabaseName NVARCHAR(128)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Populate database documentation
    INSERT INTO DocumentationDB.dbo.DatabaseDocumentation (
        DatabaseName, ObjectType, ObjectName, Description, LastUpdated, UpdatedBy
    )
    SELECT 
        @DatabaseName,
        CASE 
            WHEN o.type = 'U' THEN 'Table'
            WHEN o.type = 'V' THEN 'View'
            WHEN o.type = 'P' THEN 'Stored Procedure'
            WHEN o.type = 'FN' THEN 'Function'
            WHEN o.type = 'IF' THEN 'Function'
            ELSE 'Unknown'
        END,
        o.name,
        ISNULL(p.value, 'Auto-generated documentation'),
        GETDATE(),
        SUSER_SNAME()
    FROM msdb.dbo.sysdatabases d
    INNER JOIN master.sys.databases db ON d.name = db.name
    INNER JOIN sys.objects o ON db.database_id = o.object_id
    LEFT JOIN sys.extended_properties p ON o.object_id = p.major_id AND p.minor_id = 0
    WHERE d.name = @DatabaseName
    AND o.is_ms_shipped = 0
    AND NOT EXISTS (
        SELECT 1 FROM DocumentationDB.dbo.DatabaseDocumentation dd
        WHERE dd.DatabaseName = @DatabaseName
        AND dd.ObjectType = CASE 
            WHEN o.type = 'U' THEN 'Table'
            WHEN o.type = 'V' THEN 'View'
            WHEN o.type = 'P' THEN 'Stored Procedure'
            WHEN o.type = 'FN' THEN 'Function'
            WHEN o.type = 'IF' THEN 'Function'
            ELSE 'Unknown'
        END
        AND dd.ObjectName = o.name
    );
    
    -- Populate column documentation
    INSERT INTO DocumentationDB.dbo.ColumnDocumentation (
        DatabaseName, TableName, ColumnName, BusinessName, Description, LastUpdated, UpdatedBy
    )
    SELECT 
        @DatabaseName,
        t.name,
        c.name,
        c.name AS BusinessName, -- Default to column name
        ISNULL(p.value, 'Auto-generated documentation'),
        GETDATE(),
        SUSER_SNAME()
    FROM msdb.dbo.sysdatabases d
    INNER JOIN master.sys.databases db ON d.name = db.name
    INNER JOIN sys.tables t ON db.database_id = t.object_id
    INNER JOIN sys.columns c ON t.object_id = c.object_id
    LEFT JOIN sys.extended_properties p ON c.object_id = p.major_id AND c.column_id = p.minor_id
    WHERE d.name = @DatabaseName
    AND NOT EXISTS (
        SELECT 1 FROM DocumentationDB.dbo.ColumnDocumentation cd
        WHERE cd.DatabaseName = @DatabaseName
        AND cd.TableName = t.name
        AND cd.ColumnName = c.name
    );
    
    PRINT 'Documentation generated for database: ' + @DatabaseName;
END;
```

## Performance Monitoring Tools {#performance-tools}

### Step 1: Extended Events Setup

**Create Comprehensive Extended Events Session:**
```sql
-- Create Extended Events session for performance monitoring
CREATE EVENT SESSION [PerformanceMonitor] ON SERVER 
ADD EVENT sqlserver.auto_stats(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.query_hash,
        sqlserver.query_plan_hash,
        sqlserver.session_id,
        sqlserver.sql_text
    )
),
ADD EVENT sqlserver.blocked_process_report(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.session_id,
        sqlserver.sql_text,
        sqlserver.username
    )
),
ADD EVENT sqlserver.deadlock_graph(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.session_id,
        sqlserver.username
    )
),
ADD EVENT sqlserver.module_end(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.query_hash,
        sqlserver.query_plan_hash,
        sqlserver.session_id
    )
    WHERE (
        [duration]>(1000000) -- Greater than 1 second
    )
),
ADD EVENT sqlserver.rpc_completed(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.query_hash,
        sqlserver.query_plan_hash,
        sqlserver.session_id,
        sqlserver.sql_text
    )
    WHERE (
        [duration]>(1000000) -- Greater than 1 second
    )
)
ADD TARGET package0.event_file(
    SET filename=N'PerformanceMonitor.xel',
    max_file_size=(100), -- 100 MB
    max_rollover_files=(5) -- Keep 5 files
)
WITH (
    MAX_MEMORY=4096 KB,
    EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY=30 SECONDS,
    MAX_EVENT_SIZE=0 KB,
    MEMORY_PARTITION_MODE=NONE,
    TRACK_CAUSALITY=ON,
    STARTUP_STATE=ON
);
GO

-- Start the session
ALTER EVENT SESSION [PerformanceMonitor] ON SERVER STATE = START;
GO

-- Query Extended Events data
SELECT 
    event_data.value('(@timestamp)[1]', 'DATETIME2') AS EventTime,
    event_data.value('(action[@name="sqlserver.client_app_name"]/value)[1]', 'NVARCHAR(200)') AS ClientApp,
    event_data.value('(action[@name="sqlserver.session_id"]/value)[1]', 'INT') AS SessionID,
    event_data.value('(action[@name="sqlserver.database_name"]/value)[1]', 'NVARCHAR(128)') AS DatabaseName,
    event_data.value('(action[@name="sqlserver.sql_text"]/value)[1]', 'NVARCHAR(MAX)') AS SQLText,
    event_data.value('(action[@name="sqlserver.query_hash"]/value)[1]', 'NVARCHAR(100)') AS QueryHash,
    CASE 
        WHEN event_data.value('(@name)[1]', 'NVARCHAR(100)') = 'sqlserver.module_end' THEN
            CAST(event_data.value('(data[@name="duration"]/value)[1]', 'BIGINT') / 1000000.0 AS DECIMAL(18,3))
        WHEN event_data.value('(@name)[1]', 'NVARCHAR(100)') = 'sqlserver.rpc_completed' THEN
            CAST(event_data.value('(data[@name="duration"]/value)[1]', 'BIGINT') / 1000000.0 AS DECIMAL(18,3))
    END AS DurationSeconds,
    CASE event_data.value('(@name)[1]', 'NVARCHAR(100)')
        WHEN 'sqlserver.module_end' THEN 'Module End'
        WHEN 'sqlserver.rpc_completed' THEN 'RPC Completed'
        WHEN 'sqlserver.auto_stats' THEN 'Auto Stats'
        WHEN 'sqlserver.blocked_process_report' THEN 'Blocked Process'
        WHEN 'sqlserver.deadlock_graph' THEN 'Deadlock'
    END AS EventType
FROM sys.fn_xe_file_target_read_file('PerformanceMonitor*.xel', NULL, NULL, NULL)
CROSS APPLY event_data.nodes('event') AS T(event_data)
ORDER BY EventTime DESC;
```

### Step 2: Performance Dashboard Creation

**Create Performance Monitoring Database:**
```sql
-- Create performance monitoring database
USE master;
GO

CREATE DATABASE PerformanceMonitor;
GO

USE PerformanceMonitor;
GO

-- Create performance metrics table
CREATE TABLE PerformanceMetrics (
    MetricID INT IDENTITY(1,1) PRIMARY KEY,
    CollectionTime DATETIME2 DEFAULT GETDATE(),
    ServerName NVARCHAR(128),
    DatabaseName NVARCHAR(128),
    CounterName NVARCHAR(100),
    CounterValue BIGINT,
    CounterUnit NVARCHAR(20),
    ObjectName NVARCHAR(100),
    InstanceName NVARCHAR(100)
);

-- Create query performance table
CREATE TABLE QueryPerformance (
    QueryID INT IDENTITY(1,1) PRIMARY KEY,
    CollectionTime DATETIME2 DEFAULT GETDATE(),
    ServerName NVARCHAR(128),
    DatabaseName NVARCHAR(128),
    QueryText NVARCHAR(MAX),
    ExecutionCount BIGINT,
    TotalElapsedTime BIGINT,
    AvgElapsedTime DECIMAL(18,3),
    MinElapsedTime DECIMAL(18,3),
    MaxElapsedTime DECIMAL(18,3),
    TotalLogicalReads BIGINT,
    AvgLogicalReads DECIMAL(18,3),
    TotalCPU BIGINT,
    AvgCPU DECIMAL(18,3)
);

-- Create procedure to collect performance data
CREATE PROCEDURE sp_CollectPerformanceData
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @ServerName NVARCHAR(128) = @@SERVERNAME;
    
    -- Collect performance counters
    INSERT INTO PerformanceMetrics (ServerName, CounterName, CounterValue, CounterUnit, ObjectName, InstanceName)
    SELECT 
        @ServerName,
        rtrim(counter_name) AS CounterName,
        CAST(cntr_value AS BIGINT) AS CounterValue,
        CASE 
            WHEN counter_name LIKE '%per sec%' THEN 'per second'
            WHEN counter_name LIKE '%bytes%' THEN 'bytes'
            WHEN counter_name LIKE '%count%' THEN 'count'
            ELSE 'count'
        END AS CounterUnit,
        rtrim(object_name) AS ObjectName,
        rtrim(instance_name) AS InstanceName
    FROM sys.dm_os_performance_counters
    WHERE counter_name IN (
        'Batch Requests/sec',
        'SQL Compilations/sec',
        'SQL Recompilations/sec',
        'Buffer cache hit ratio',
        'Buffer cache hit ratio base',
        'Page life expectancy',
        'Free list stalls/sec',
        'Lazy writes/sec',
        'Checkpoint pages/sec',
        'Database pages'
    );
    
    -- Collect query performance data
    INSERT INTO QueryPerformance (ServerName, DatabaseName, QueryText, ExecutionCount, TotalElapsedTime, AvgElapsedTime, MinElapsedTime, MaxElapsedTime, TotalLogicalReads, AvgLogicalReads, TotalCPU, AvgCPU)
    SELECT 
        @ServerName,
        DB_NAME(st.dbid) AS DatabaseName,
        SUBSTRING(st.text, 1, 1000) AS QueryText,
        qs.execution_count AS ExecutionCount,
        qs.total_elapsed_time / 1000 AS TotalElapsedTime, -- Convert to milliseconds
        qs.total_elapsed_time / 1000.0 / qs.execution_count AS AvgElapsedTime,
        qs.min_elapsed_time / 1000.0 AS MinElapsedTime,
        qs.max_elapsed_time / 1000.0 AS MaxElapsedTime,
        qs.total_logical_reads AS TotalLogicalReads,
        qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
        qs.total_worker_time / 1000 AS TotalCPU, -- Convert to milliseconds
        qs.total_worker_time / 1000.0 / qs.execution_count AS AvgCPU
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
    WHERE qs.last_execution_time > DATEADD(minute, -5, GETDATE()) -- Last 5 minutes
    AND qs.execution_count > 1
    ORDER BY qs.total_elapsed_time DESC
    OPTION (RECOMPILE);
    
    -- Remove old data (keep only last 30 days)
    DELETE FROM PerformanceMetrics WHERE CollectionTime < DATEADD(day, -30, GETDATE());
    DELETE FROM QueryPerformance WHERE CollectionTime < DATEADD(day, -30, GETDATE());
    
    PRINT 'Performance data collected at ' + CONVERT(VARCHAR(30), GETDATE(), 121);
END;
```

### Step 3: Automated Alerting System

**Create Alert Monitoring System:**
```sql
-- Create alerting configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName NVARCHAR(100) NOT NULL,
    AlertType NVARCHAR(50), -- Performance, Resource, Error
    CounterName NVARCHAR(100),
    ThresholdValue DECIMAL(18,2),
    ThresholdOperator NVARCHAR(10), -- >, <, >=, <=, =
    EvaluationPeriodMinutes INT,
    AlertSeverity NVARCHAR(20), -- Low, Medium, High, Critical
    EmailRecipients NVARCHAR(500),
    IsEnabled BIT DEFAULT 1,
    LastEvaluation DATETIME2,
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    CreatedBy NVARCHAR(100) DEFAULT SUSER_SNAME()
);

-- Insert default alert configurations
INSERT INTO AlertConfiguration (AlertName, AlertType, CounterName, ThresholdValue, ThresholdOperator, EvaluationPeriodMinutes, AlertSeverity, EmailRecipients)
VALUES 
    ('High CPU Usage', 'Resource', 'Processor(_Total)\% Processor Time', 80, '>', 15, 'High', 'dba@company.com'),
    ('Low Memory', 'Resource', 'Memory\Available MBytes', 500, '<', 10, 'Medium', 'dba@company.com'),
    ('Long Running Queries', 'Performance', 'QueryElapsedTime', 30, '>', 5, 'Medium', 'dba@company.com'),
    ('Deadlock Detection', 'Performance', 'Deadlocks', 1, '>', 0, 'High', 'dba@company.com'),
    ('Connection Failures', 'Error', 'Connection Errors', 10, '>', 10, 'High', 'dba@company.com');

-- Create alert evaluation procedure
CREATE PROCEDURE sp_EvaluateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertID INT;
    DECLARE @AlertName NVARCHAR(100);
    DECLARE @CounterName NVARCHAR(100);
    DECLARE @ThresholdValue DECIMAL(18,2);
    DECLARE @ThresholdOperator NVARCHAR(10);
    DECLARE @CurrentValue DECIMAL(18,2);
    DECLARE @AlertSeverity NVARCHAR(20);
    DECLARE @EmailRecipients NVARCHAR(500);
    
    -- Cursor through enabled alerts
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertID, AlertName, CounterName, ThresholdValue, ThresholdOperator, AlertSeverity, EmailRecipients
    FROM AlertConfiguration
    WHERE IsEnabled = 1;
    
    OPEN alert_cursor;
    
    FETCH NEXT FROM alert_cursor INTO @AlertID, @AlertName, @CounterName, @ThresholdValue, @ThresholdOperator, @AlertSeverity, @EmailRecipients;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Evaluate different alert types
        SET @CurrentValue = CASE @CounterName
            WHEN 'Processor(_Total)\% Processor Time' THEN
                (SELECT CAST(cntr_value AS DECIMAL(18,2)) FROM sys.dm_os_performance_counters WHERE counter_name = '% Processor Time' AND object_name LIKE '%Processor%')
            WHEN 'Memory\Available MBytes' THEN
                (SELECT CAST(cntr_value AS DECIMAL(18,2)) FROM sys.dm_os_performance_counters WHERE counter_name = 'Available MBytes')
            WHEN 'QueryElapsedTime' THEN
                (SELECT TOP 1 CAST(total_elapsed_time / 1000.0 / execution_count AS DECIMAL(18,2)) FROM sys.dm_exec_query_stats WHERE total_elapsed_time / 1000.0 / execution_count > @ThresholdValue ORDER BY total_elapsed_time DESC)
            WHEN 'Deadlocks' THEN
                (SELECT COUNT(*) FROM sys.dm_os_wait_stats WHERE wait_type = 'LCK_M_DEADLOCK')
            ELSE 0
        END;
        
        -- Check if alert condition is met
        IF CASE @ThresholdOperator
            WHEN '>' THEN CASE WHEN @CurrentValue > @ThresholdValue THEN 1 ELSE 0 END
            WHEN '<' THEN CASE WHEN @CurrentValue < @ThresholdValue THEN 1 ELSE 0 END
            WHEN '>=' THEN CASE WHEN @CurrentValue >= @ThresholdValue THEN 1 ELSE 0 END
            WHEN '<=' THEN CASE WHEN @CurrentValue <= @ThresholdValue THEN 1 ELSE 0 END
            WHEN '=' THEN CASE WHEN @CurrentValue = @ThresholdValue THEN 1 ELSE 0 END
            ELSE 0
        END = 1
        BEGIN
            -- Log alert
            INSERT INTO AlertLog (AlertID, AlertName, TriggeredValue, ThresholdValue, Severity, TriggeredTime, EmailSent)
            VALUES (@AlertID, @AlertName, @CurrentValue, @ThresholdValue, @AlertSeverity, GETDATE(), 0);
            
            -- Send email alert (requires database mail configuration)
            DECLARE @EmailSubject NVARCHAR(200) = 'SQL Server Alert: ' + @AlertName;
            DECLARE @EmailBody NVARCHAR(MAX) = 'Alert: ' + @AlertName + CHAR(13) + CHAR(10) +
                'Server: ' + @@SERVERNAME + CHAR(13) + CHAR(10) +
                'Current Value: ' + CAST(@CurrentValue AS NVARCHAR(20)) + CHAR(13) + CHAR(10) +
                'Threshold: ' + @ThresholdOperator + ' ' + CAST(@ThresholdValue AS NVARCHAR(20)) + CHAR(13) + CHAR(10) +
                'Time: ' + CONVERT(VARCHAR(30), GETDATE(), 121);
            
            -- Uncomment and configure database mail to send alerts
            -- EXEC msdb.dbo.sp_send_dbmail
            --     @profile_name = 'Default',
            --     @recipients = @EmailRecipients,
            --     @subject = @EmailSubject,
            --     @body = @EmailBody;
            
            PRINT 'Alert triggered: ' + @AlertName + ' (Value: ' + CAST(@CurrentValue AS NVARCHAR(20)) + ')';
        END
        
        -- Update last evaluation time
        UPDATE AlertConfiguration
        SET LastEvaluation = GETDATE()
        WHERE AlertID = @AlertID;
        
        FETCH NEXT FROM alert_cursor INTO @AlertID, @AlertName, @CounterName, @ThresholdValue, @ThresholdOperator, @AlertSeverity, @EmailRecipients;
    END
    
    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
    
    PRINT 'Alert evaluation completed at ' + CONVERT(VARCHAR(30), GETDATE(), 121);
END;
```

This comprehensive tool setup guide covers SSMS optimization, Visual Studio configuration, PowerShell automation, performance monitoring, and documentation systems. Each section provides practical, implementable solutions for database development and administration teams.

For additional configuration details or troubleshooting specific tools, refer to the respective vendor documentation or consult with your database administration team.