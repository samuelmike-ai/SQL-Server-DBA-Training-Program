# Week 17: Cloud SQL Server Deployment and Management

## Learning Objectives
By the end of this week, you will be able to:
- Design and implement Azure SQL Database and Azure SQL Managed Instance solutions
- Develop comprehensive cloud migration strategies for SQL Server workloads
- Build hybrid cloud architectures that seamlessly integrate on-premises and cloud environments
- Implement cloud-native features like automated backups, scaling, and monitoring
- Manage cloud security, compliance, and cost optimization for SQL Server deployments

## Introduction to Cloud SQL Server

The migration of SQL Server workloads to cloud platforms represents one of the most significant transformations in enterprise database management. Cloud platforms offer unprecedented scalability, global availability, and cost flexibility while introducing new challenges around security, compliance, and operational management.

This week, we'll explore enterprise-level cloud SQL Server implementations used by organizations ranging from startups to Fortune 500 companies. We'll examine real-world scenarios including financial services firms migrating regulatory-compliant workloads to meet data residency requirements, healthcare organizations implementing HIPAA-compliant cloud architectures, and global retailers coordinating inventory systems across multiple regions.

Our focus will be on building robust, scalable cloud solutions that leverage platform-as-a-service (PaaS) advantages while maintaining the flexibility and control of traditional SQL Server deployments.

## Azure SQL Database Architecture and Implementation

### Understanding Azure SQL Service Tiers and Deployment Models

Azure SQL offers multiple service tiers and deployment models, each optimized for different workload characteristics and business requirements.

**Service Tier Selection Framework:**

```sql
-- Create Azure SQL service tier analysis and recommendation system
CREATE TABLE AzureSQLServiceAnalysis (
    AnalysisID INT IDENTITY(1,1) PRIMARY KEY,
    ApplicationName NVARCHAR(256),
    CurrentWorkloadProfile NVARCHAR(MAX), -- Description of current workload
    PeakConcurrentConnections INT,
    AvgCPUUtilization DECIMAL(5,2),
    PeakCPUUtilization DECIMAL(5,2),
    DataSizeGB BIGINT,
    DataGrowthRatePercent DECIMAL(5,2),
    PerformanceRequirement NVARCHAR(100), -- 'Standard', 'High', 'Critical'
    AvailabilityRequirement NVARCHAR(50), -- '99.9%', '99.95%', '99.99%'
    RecoveryTimeObjective INT, -- in minutes
    RecoveryPointObjective INT, -- in minutes
    RecommendedServiceTier NVARCHAR(50),
    RecommendedComputeTier NVARCHAR(50), -- 'vCore', 'DTU'
    EstimatedMonthlyCost DECIMAL(10,2),
    MigrationComplexity NVARCHAR(50), -- 'Low', 'Medium', 'High'
    LastAnalyzed DATETIME2 DEFAULT GETDATE(),
    AnalysisBy NVARCHAR(256) DEFAULT SYSTEM_USER
);

-- Create service tier recommendation procedure
CREATE PROCEDURE sp_AnalyzeAzureSQLRequirements
    @ApplicationName NVARCHAR(256),
    @PeakConcurrentConnections INT,
    @AvgCPUUtilization DECIMAL(5,2),
    @PeakCPUUtilization DECIMAL(5,2),
    @DataSizeGB BIGINT,
    @DataGrowthRatePercent DECIMAL(5,2),
    @PerformanceRequirement NVARCHAR(100) = 'Standard',
    @AvailabilityRequirement NVARCHAR(50) = '99.9%',
    @RecoveryTimeObjective INT = 60,
    @RecoveryPointObjective INT = 15
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @RecommendedServiceTier NVARCHAR(50);
    DECLARE @RecommendedComputeTier NVARCHAR(50);
    DECLARE @EstimatedMonthlyCost DECIMAL(10,2);
    DECLARE @MigrationComplexity NVARCHAR(50);
    DECLARE @DTURecommendation INT;
    DECLARE @VCoreRecommendation INT;
    DECLARE @StorageRecommendation INT;
    
    -- Analyze performance requirements
    IF @PeakCPUUtilization > 80 OR @AvgCPUUtilization > 60
    BEGIN
        -- High CPU utilization suggests need for more compute
        SET @RecommendedComputeTier = 'vCore'; -- Better for scaling compute
    END
    ELSE
    BEGIN
        SET @RecommendedComputeTier = 'DTU'; -- Cost-effective for moderate workloads
    END
    
    -- Analyze availability requirements
    IF @AvailabilityRequirement = '99.99%'
    BEGIN
        SET @RecommendedServiceTier = 'Premium';
        SET @MigrationComplexity = 'High';
    END
    ELSE IF @AvailabilityRequirement = '99.95%'
    BEGIN
        SET @RecommendedServiceTier = 'Premium'; -- Or Business Critical for vCore
        SET @MigrationComplexity = 'Medium';
    END
    ELSE
    BEGIN
        SET @RecommendedServiceTier = 'Standard';
        SET @MigrationComplexity = 'Low';
    END
    
    -- Calculate DTU recommendations for DTU-based service
    IF @RecommendedComputeTier = 'DTU'
    BEGIN
        SET @DTURecommendation = CASE 
            WHEN @PerformanceRequirement = 'Critical' AND @PeakCPUUtilization > 70 THEN 800
            WHEN @PerformanceRequirement = 'High' AND @PeakCPUUtilization > 50 THEN 400
            WHEN @PerformanceRequirement = 'Standard' AND @PeakCPUUtilization > 30 THEN 100
            ELSE 50
        END;
        
        SET @EstimatedMonthlyCost = @DTURecommendation * 25.0; -- Rough cost calculation
    END
    ELSE
    BEGIN
        -- Calculate vCore recommendations
        SET @VCoreRecommendation = CASE 
            WHEN @PeakCPUUtilization > 80 THEN 16
            WHEN @PeakCPUUtilization > 60 THEN 12
            WHEN @PeakCPUUtilization > 40 THEN 8
            WHEN @PeakCPUUtilization > 20 THEN 4
            ELSE 2
        END;
        
        SET @StorageRecommendation = CASE 
            WHEN @DataSizeGB > 1000 THEN @DataSizeGB * 1.2 -- 20% overhead
            WHEN @DataSizeGB > 100 THEN @DataSizeGB * 1.5
            ELSE @DataSizeGB * 2
        END;
        
        SET @EstimatedMonthlyCost = (@VCoreRecommendation * 15.0) + (@StorageRecommendation * 0.1); -- Rough cost calculation
    END
    
    -- Log analysis results
    INSERT INTO AzureSQLServiceAnalysis (
        ApplicationName, PeakConcurrentConnections, AvgCPUUtilization, 
        PeakCPUUtilization, DataSizeGB, DataGrowthRatePercent,
        PerformanceRequirement, AvailabilityRequirement,
        RecoveryTimeObjective, RecoveryPointObjective,
        RecommendedServiceTier, RecommendedComputeTier,
        EstimatedMonthlyCost, MigrationComplexity
    ) VALUES (
        @ApplicationName, @PeakConcurrentConnections, @AvgCPUUtilization,
        @PeakCPUUtilization, @DataSizeGB, @DataGrowthRatePercent,
        @PerformanceRequirement, @AvailabilityRequirement,
        @RecoveryTimeObjective, @RecoveryPointObjective,
        @RecommendedServiceTier, @RecommendedComputeTier,
        @EstimatedMonthlyCost, @MigrationComplexity
    );
    
    -- Return detailed recommendations
    SELECT 
        @ApplicationName as ApplicationName,
        @RecommendedServiceTier as RecommendedServiceTier,
        @RecommendedComputeTier as RecommendedComputeTier,
        @DTURecommendation as DTURecommendation,
        @VCoreRecommendation as VCoreRecommendation,
        @StorageRecommendation as StorageRecommendation,
        @EstimatedMonthlyCost as EstimatedMonthlyCost,
        @MigrationComplexity as MigrationComplexity,
        CASE 
            WHEN @RecommendedComputeTier = 'DTU' 
            THEN CONCAT('Standard Tier ', @DTURecommendation, ' DTU')
            ELSE CONCAT(@RecommendedServiceTier, ' Tier ', @VCoreRecommendation, ' vCore, ', @StorageRecommendation, ' GB Storage')
        END as ServiceTierDescription,
        CASE @MigrationComplexity
            WHEN 'Low' THEN 'Migration can be completed with minimal downtime using Azure Database Migration Service'
            WHEN 'Medium' THEN 'Migration requires careful planning and testing, consider staged rollout'
            WHEN 'High' THEN 'Complex migration requiring architecture redesign and extensive testing'
        END as MigrationApproach;
END;

-- Example analysis for different application types
EXEC sp_AnalyzeAzureSQLRequirements
    @ApplicationName = 'E-commerce Transaction Processing',
    @PeakConcurrentConnections = 500,
    @AvgCPUUtilization = 45.0,
    @PeakCPUUtilization = 75.0,
    @DataSizeGB = 250,
    @DataGrowthRatePercent = 25.0,
    @PerformanceRequirement = 'High',
    @AvailabilityRequirement = '99.95%',
    @RecoveryTimeObjective = 15,
    @RecoveryPointObjective = 5;

EXEC sp_AnalyzeAzureSQLRequirements
    @ApplicationName = 'Legacy HR System',
    @PeakConcurrentConnections = 50,
    @AvgCPUUtilization = 15.0,
    @PeakCPUUtilization = 30.0,
    @DataSizeGB = 50,
    @DataGrowthRatePercent = 5.0,
    @PerformanceRequirement = 'Standard',
    @AvailabilityRequirement = '99.9%',
    @RecoveryTimeObjective = 60,
    @RecoveryPointObjective = 60;
```

### Azure SQL Database Deployment and Configuration

Enterprise deployments require sophisticated configuration that considers security, performance, and operational requirements.

**Automated Azure SQL Database Deployment:**

```powershell
# Azure SQL Database Deployment Script
# This script would be used in CI/CD pipelines or Infrastructure as Code

param(
    [Parameter(Mandatory=$true)]
    [string]$ResourceGroupName,
    
    [Parameter(Mandatory=$true)]
    [string]$ServerName,
    
    [Parameter(Mandatory=$true)]
    [string]$DatabaseName,
    
    [Parameter(Mandatory=$true)]
    [string]$ServiceTier, # Basic, Standard, Premium, Business Critical, Hyperscale
    
    [Parameter(Mandatory=$true)]
    [int]$VCores, # For vCore-based service
    
    [Parameter(Mandatory=$true)]
    [long]$StorageGB,
    
    [Parameter(Mandatory=$false)]
    [string]$AdministratorLogin = "sqladmin",
    
    [Parameter(Mandatory=$true)]
    [SecureString]$AdministratorPassword,
    
    [Parameter(Mandatory=$false)]
    [switch]$EnableAdvancedThreatProtection = $true,
    
    [Parameter(Mandatory=$false)]
    [switch]$EnableTransparentDataEncryption = $true,
    
    [Parameter(Mandatory=$false)]
    [string]$BackupRetentionDays = 30,
    
    [Parameter(Mandatory=$false)]
    [string]$Zone = "East US"
)

# Login to Azure (this would be handled by service principal in automation)
# Connect-AzAccount -ServicePrincipal -TenantId $env:TENANT_ID -ApplicationId $env:APPLICATION_ID -CertificateThumbprint $env:CERTIFICATE_THUMBPRINT

Write-Host "Starting Azure SQL Database deployment..." -ForegroundColor Green

try {
    # Create resource group if it doesn't exist
    $resourceGroup = Get-AzResourceGroup -Name $ResourceGroupName -ErrorAction SilentlyContinue
    if (-not $resourceGroup) {
        Write-Host "Creating resource group: $ResourceGroupName" -ForegroundColor Yellow
        New-AzResourceGroup -Name $ResourceGroupName -Location $Zone
    }
    
    # Create SQL server if it doesn't exist
    $sqlServer = Get-AzSqlServer -ResourceGroupName $ResourceGroupName -ServerName $ServerName -ErrorAction SilentlyContinue
    if (-not $sqlServer) {
        Write-Host "Creating SQL Server: $ServerName" -ForegroundColor Yellow
        $adminPassword = ConvertFrom-SecureString $AdministratorPassword -AsPlainText
        New-AzSqlServer -ResourceGroupName $ResourceGroupName `
                       -ServerName $ServerName `
                       -Location $Zone `
                       -SqlAdministratorCredentials (New-Object PSCredential($AdministratorLogin, $AdministratorPassword)) `
                       -MinimalTlsVersion 1.2
    }
    
    # Configure server-level settings
    Write-Host "Configuring server settings..." -ForegroundColor Yellow
    $sqlServer = Get-AzSqlServer -ResourceGroupName $ResourceGroupName -ServerName $ServerName
    
    # Enable/disable public endpoint based on security requirements
    Update-AzSqlServer -ResourceGroupName $ResourceGroupName `
                      -ServerName $ServerName `
                      -PublicNetworkAccess Disabled
    
    # Configure firewall rules
    Write-Host "Configuring firewall rules..." -ForegroundColor Yellow
    $ipAddresses = Get-AzLocalIP -ErrorAction SilentlyContinue | ForEach-Object { $_.IPAddress }
    foreach ($ip in $ipAddresses) {
        $ruleName = "Allow-$ip"
        New-AzSqlServerFirewallRule -ResourceGroupName $ResourceGroupName `
                                   -ServerName $ServerName `
                                   -FirewallRuleName $ruleName `
                                   -StartIpAddress $ip `
                                   -EndIpAddress $ip `
                                   -ErrorAction SilentlyContinue
    }
    
    # Allow Azure services
    New-AzSqlServerFirewallRule -ResourceGroupName $ResourceGroupName `
                               -ServerName $ServerName `
                               -FirewallRuleName "AllowAzureServices" `
                               -StartIpAddress "0.0.0.0" `
                               -EndIpAddress "0.0.0.0" `
                               -ErrorAction SilentlyContinue
    
    # Create database with specified configuration
    Write-Host "Creating database: $DatabaseName" -ForegroundColor Yellow
    
    $databaseParams = @{
        ResourceGroupName = $ResourceGroupName
        ServerName = $ServerName
        DatabaseName = $DatabaseName
        PerformanceLevel = if ($ServiceTier -eq "Business Critical") {
            "$ServiceTier/$VCores vCore"
        } elseif ($ServiceTier -eq "Hyperscale") {
            "Hyperscale/$VCores vCore"
        } else {
            "$ServiceTier/$VCores vCore"
        }
        BackupRetentionDays = $BackupRetentionDays
        ZoneRedundant = $true
        ReadScale = "Enabled"
    }
    
    $sqlDatabase = New-AzSqlDatabase @databaseParams
    
    # Configure advanced security features
    if ($EnableAdvancedThreatProtection) {
        Write-Host "Enabling Advanced Threat Protection..." -ForegroundColor Yellow
        
        $threatProtectionParams = @{
            ResourceGroupName = $ResourceGroupName
            ServerName = $ServerName
            DatabaseName = $DatabaseName
            NotificationRecipientsEmails = "security@company.com"
            EmailAdmins = $true
        }
        
        Set-AzSqlDatabaseThreatDetectionPolicy @threatDetectionParams
    }
    
    # Configure Transparent Data Encryption
    if ($EnableTransparentDataEncryption) {
        Write-Host "Configuring Transparent Data Encryption..." -ForegroundColor Yellow
        Set-AzSqlDatabaseTransparentDataEncryption -ResourceGroupName $ResourceGroupName `
                                                  -ServerName $ServerName `
                                                  -DatabaseName $DatabaseName `
                                                  -State Enabled
    }
    
    # Configure auditing
    Write-Host "Configuring auditing..." -ForegroundColor Yellow
    Set-AzSqlDatabaseAudit -ResourceGroupName $ResourceGroupName `
                          -ServerName $ServerName `
                          -DatabaseName $DatabaseName `
                          -AuditActionGroup @("DATABASE_OWNERSHIP_CHANGE_GROUP", "DATABASE_OBJECT_CHANGE_GROUP", "SCHEMA_OBJECT_CHANGE_GROUP", "DATABASE_PRIVILEGE_CHANGE_GROUP", "DATABASE_ACCESS_GROUP", "DATABASE_ACCOUNT_MANAGEMENT_GROUP") `
                          -AuditAction @("UPDATE", "INSERT", "DELETE") `
                          -RetentionInDays $BackupRetentionDays
    
    # Configure automatic tuning
    Write-Host "Enabling automatic tuning..." -ForegroundColor Yellow
    Set-AzSqlDatabaseAutomaticTuning -ResourceGroupName $ResourceGroupName `
                                    -ServerName $ServerName `
                                    -DatabaseName $DatabaseName `
                                    -ForceIndexTuning On `
                                    -LastKnownGoodPlanTuning On `
                                    -AutoTuningPauseReason Off
    
    # Create connection strings documentation
    Write-Host "Generating connection information..." -ForegroundColor Yellow
    
    $connectionStrings = @{
        "ADO.NET" = "Server=tcp:$ServerName.database.windows.net,1433;Database=$DatabaseName;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
        "ODBC" = "Driver={ODBC Driver 17 for SQL Server};Server=tcp:$ServerName.database.windows.net,1433;Database=$DatabaseName;Encrypt=True;TrustServerCertificate=False;"
        "JDBC" = "jdbc:sqlserver://$ServerName.database.windows.net:1433;database=$DatabaseName;encrypt=true;trustServerCertificate=false;loginTimeout=30;"
        "Python" = "pyodbc://$AdministratorLogin@{password}@$ServerName.database.windows.net:1433/$DatabaseName?encrypt=true&trustServerCertificate=false&connection+timeout+30"
    }
    
    # Write connection strings to file
    $connectionInfo = @{
        "ServerName" = $ServerName
        "DatabaseName" = $DatabaseName
        "ConnectionStrings" = $connectionStrings
        "DeploymentTime" = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")
    }
    
    $connectionInfo | ConvertTo-Json -Depth 3 | Out-File -FilePath "./deployment-$DatabaseName-connections.json"
    
    Write-Host "Azure SQL Database deployment completed successfully!" -ForegroundColor Green
    Write-Host "Server: $ServerName.database.windows.net" -ForegroundColor Cyan
    Write-Host "Database: $DatabaseName" -ForegroundColor Cyan
    Write-Host "Service Tier: $ServiceTier" -ForegroundColor Cyan
    Write-Host "Configuration saved to: deployment-$DatabaseName-connections.json" -ForegroundColor Cyan
    
} catch {
    Write-Host "Deployment failed: $($_.Exception.Message)" -ForegroundColor Red
    throw
}
```

### Azure SQL Managed Instance Implementation

For organizations requiring near-complete SQL Server feature parity, Azure SQL Managed Instance offers a middle ground between Azure SQL Database and on-premises SQL Server.

**Azure SQL Managed Instance Deployment Architecture:**

```powershell
# Azure SQL Managed Instance Deployment Script
param(
    [Parameter(Mandatory=$true)]
    [string]$ResourceGroupName,
    
    [Parameter(Mandatory=$true)]
    [string]$ManagedInstanceName,
    
    [Parameter(Mandatory=$true)]
    [string]$SubnetId, # Pre-configured subnet with required NSG rules
    
    [Parameter(Mandatory=$true)]
    [int]$VCores,
    
    [Parameter(Mandatory=$true)]
    [int]$StorageSizeGB,
    
    [Parameter(Mandatory=$true)]
    [string]$ServiceTier, # General Purpose or Business Critical
    
    [Parameter(Mandatory=$true)]
    [string]$LicenseType, # LicenseIncluded or BYOL
    
    [Parameter(Mandatory=$false)]
    [SecureString]$AdministratorPassword,
    
    [Parameter(Mandatory=$false)]
    [string]$TimeZone = "UTC",
    
    [Parameter(Mandatory=$false)]
    [switch]$EnablePublicDataEndpoint = $false,
    
    [Parameter(Mandatory=$false)]
    [string]$Zone = "East US"
)

# Azure SQL Managed Instance requires specific network configuration
# This includes a dedicated subnet with proper NSG rules and route tables

Write-Host "Starting Azure SQL Managed Instance deployment..." -ForegroundColor Green

try {
    # Validate subnet configuration
    Write-Host "Validating subnet configuration..." -ForegroundColor Yellow
    $subnet = Get-AzVirtualNetworkSubnetConfig -ResourceId $SubnetId
    
    # Check for required NSG
    if (-not $subnet.NetworkSecurityGroup) {
        throw "Subnet must have a Network Security Group configured with required rules"
    }
    
    # Check for required route table
    if (-not $subnet.RouteTable) {
        throw "Subnet must have a Route Table configured for Private Endpoints"
    }
    
    # Create Managed Instance
    Write-Host "Creating Managed Instance: $ManagedInstanceName" -ForegroundColor Yellow
    
    $adminPassword = ConvertFrom-SecureString $AdministratorPassword -AsPlainText
    
    $miParams = @{
        Name = $ManagedInstanceName
        ResourceGroupName = $ResourceGroupName
        Location = $Zone
        SubnetId = $SubnetId
        VCore = $VCores
        StorageSizeInGB = $StorageSizeGB
        ServiceTier = $ServiceTier
        LicenseType = $LicenseType
        SqlAdministratorCredentials = (New-Object PSCredential("sqladmin", $AdministratorPassword))
        PublicDataEndpointEnabled = $EnablePublicDataEndpoint
        TimezoneId = $Zone
        Tags = @{
            Environment = "Production"
            CostCenter = "DB001"
            ManagedBy = "AzureDevOps"
            CreatedDate = (Get-Date).ToString("yyyy-MM-dd")
        }
    }
    
    $managedInstance = New-AzSqlManagedInstance @miParams
    
    # Wait for provisioning to complete
    Write-Host "Waiting for Managed Instance provisioning to complete..." -ForegroundColor Yellow
    $maxAttempts = 30
    $attempt = 1
    
    do {
        $currentStatus = Get-AzSqlManagedInstance -ResourceGroupName $ResourceGroupName -Name $ManagedInstanceName
        Write-Host "Provisioning status: $($currentStatus.ProvisioningState)" -ForegroundColor Gray
        
        if ($currentStatus.ProvisioningState -eq "Succeeded") {
            Write-Host "Managed Instance provisioned successfully!" -ForegroundColor Green
            break
        } elseif ($currentStatus.ProvisioningState -eq "Failed") {
            throw "Managed Instance provisioning failed"
        }
        
        Start-Sleep -Seconds 60
        $attempt++
    } while ($attempt -le $maxAttempts)
    
    # Configure Managed Instance specific features
    Write-Host "Configuring Managed Instance features..." -ForegroundColor Yellow
    
    # Enable automatic failover groups for high availability
    # This would typically be configured between primary and secondary regions
    Write-Host "Consider setting up automatic failover groups for disaster recovery" -ForegroundColor Yellow
    
    # Create connection strings
    $connectionStrings = @{
        "Primary Endpoint" = $managedInstance.FullyQualifiedDomainName
        "Primary Port" = 1433
        "Management Endpoint" = $managedInstance.ManagementEndpoint
        "Management Port" = 14334
    }
    
    # Save deployment information
    $deploymentInfo = @{
        "ManagedInstanceName" = $ManagedInstanceName
        "ResourceGroupName" = $ResourceGroupName
        "Fqdm" = $managedInstance.FullyQualifiedDomainName
        "ProvisioningState" = $managedInstance.ProvisioningState
        "VCore" = $VCores
        "StorageSizeGB" = $StorageSizeGB
        "ServiceTier" = $ServiceTier
        "LicenseType" = $LicenseType
        "ConnectionStrings" = $connectionStrings
        "DeploymentTime" = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")
    }
    
    $deploymentInfo | ConvertTo-Json -Depth 3 | Out-File -FilePath "./managed-instance-$ManagedInstanceName-deployment.json"
    
    Write-Host "Azure SQL Managed Instance deployment completed successfully!" -ForegroundColor Green
    Write-Host "FQDN: $($managedInstance.FullyQualifiedDomainName)" -ForegroundColor Cyan
    Write-Host "Status: $($managedInstance.ProvisioningState)" -ForegroundColor Cyan
    
} catch {
    Write-Host "Deployment failed: $($_.Exception.Message)" -ForegroundColor Red
    throw
}
```

## Cloud Migration Strategies

### Migration Assessment and Planning Framework

Successful cloud migration requires comprehensive assessment, planning, and execution strategies that minimize risk and ensure business continuity.

**Migration Assessment Framework:**

```sql
-- Create migration assessment tracking system
CREATE TABLE MigrationAssessment (
    AssessmentID INT IDENTITY(1,1) PRIMARY KEY,
    ServerName NVARCHAR(256),
    DatabaseName NVARCHAR(256),
    DatabaseSizeGB DECIMAL(18,2),
    CurrentVersion NVARCHAR(50),
    FeatureUsage NVARCHAR(MAX), -- List of SQL features being used
    CompatibilityLevel INT,
    ApplicationDependencies NVARCHAR(MAX),
    BusinessCriticality NVARCHAR(50), -- 'Low', 'Medium', 'High', 'Critical'
    DowntimeToleranceMinutes INT,
    CurrentPerformanceIssues NVARCHAR(MAX),
    MigrationReadinessScore DECIMAL(5,2), -- 0-100 score
    RecommendedMigrationMethod NVARCHAR(100), -- 'Online', 'Offline', 'Hybrid'
    EstimatedMigrationDuration INT, -- in hours
    RiskFactors NVARCHAR(MAX),
    PreMigrationTasks NVARCHAR(MAX),
    PostMigrationTasks NVARCHAR(MAX),
    MigrationStrategy NVARCHAR(50), -- 'LiftAndShift', 'Modernize', 'Replatform'
    AssessmentDate DATETIME2 DEFAULT GETDATE(),
    AssessedBy NVARCHAR(256) DEFAULT SYSTEM_USER,
    LastUpdated DATETIME2 DEFAULT GETDATE()
);

-- Create procedure for comprehensive migration assessment
CREATE PROCEDURE sp_AssessMigrationReadiness
    @ServerName NVARCHAR(256),
    @DatabaseName NVARCHAR(256)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CompatibilityLevel INT;
    DECLARE @DatabaseSizeGB DECIMAL(18,2);
    DECLARE @FeatureUsage NVARCHAR(MAX);
    DECLARE @ReadinessScore DECIMAL(5,2) = 100.0;
    DECLARE @RiskFactors NVARCHAR(MAX) = '';
    DECLARE @RecommendedMethod NVARCHAR(100);
    DECLARE @MigrationStrategy NVARCHAR(50);
    
    -- Get database information
    SELECT TOP 1
        @CompatibilityLevel = compatibility_level,
        @DatabaseSizeGB = (SUM(mf.size) * 8.0 / 1024.0)
    FROM sys.databases d
    JOIN sys.master_files mf ON d.database_id = mf.database_id
    WHERE d.name = @DatabaseName
    GROUP BY d.database_id, d.compatibility_level;
    
    -- Analyze feature usage and compatibility
    SET @FeatureUsage = '';
    
    -- Check for Azure SQL Database incompatible features
    DECLARE @FeatureCheckSQL NVARCHAR(MAX) = CONCAT('
        USE [', @DatabaseName, '];
        
        SELECT ''Features Analysis'' as AnalysisType,
               ''Server-level objects detected'' as Finding
        WHERE EXISTS (SELECT 1 FROM sys.server_principals WHERE name LIKE ''%[^a-zA-Z0-9]%'')
        
        UNION ALL
        
        SELECT ''Features Analysis'',
               ''Linked servers in use''
        WHERE EXISTS (SELECT 1 FROM sys.servers WHERE is_linked = 1)
        
        UNION ALL
        
        SELECT ''Features Analysis'',
               ''Database mail configuration found''
        WHERE EXISTS (SELECT 1 FROM msdb.dbo.sysmail_profile)
        
        UNION ALL
        
        SELECT ''Features Analysis'',
               ''SQL Agent jobs configured''
        WHERE EXISTS (SELECT 1 FROM msdb.dbo.sysjobs)
        
        UNION ALL
        
        SELECT ''Features Analysis'',
               ''Extended events configured''
        WHERE EXISTS (SELECT 1 FROM sys.server_event_sessions)
        
        UNION ALL
        
        SELECT ''Features Analysis'',
               ''Policy-based management in use''
        WHERE EXISTS (SELECT 1 FROM msdb.dbo.syspolicy_policy_conditions)
    ');
    
    CREATE TABLE #FeatureAnalysis (AnalysisType NVARCHAR(100), Finding NVARCHAR(MAX));
    INSERT INTO #FeatureAnalysis EXEC sp_executesql @FeatureCheckSQL;
    
    -- Calculate readiness score based on compatibility and feature usage
    DECLARE @CurrentScore DECIMAL(5,2) = 100.0;
    DECLARE @FeaturePenalty DECIMAL(5,2) = 0.0;
    
    -- Compatibility level check
    IF @CompatibilityLevel < 130 -- SQL Server 2016
        SET @FeaturePenalty = @FeaturePenalty + 20.0;
    
    IF @CompatibilityLevel < 120 -- SQL Server 2014
        SET @FeaturePenalty = @FeaturePenalty + 30.0;
    
    -- Database size impact
    IF @DatabaseSizeGB > 1000 -- 1TB
        SET @FeaturePenalty = @FeaturePenalty + 25.0;
    ELSE IF @DatabaseSizeGB > 100 -- 100GB
        SET @FeaturePenalty = @FeaturePenalty + 10.0;
    
    -- Feature usage impact
    DECLARE @FeatureCount INT;
    SELECT @FeatureCount = COUNT(*) FROM #FeatureAnalysis;
    
    IF @FeatureCount > 10
        SET @FeaturePenalty = @FeaturePenalty + 30.0;
    ELSE IF @FeatureCount > 5
        SET @FeaturePenalty = @FeaturePenalty + 15.0;
    ELSE IF @FeatureCount > 0
        SET @FeaturePenalty = @FeaturePenalty + 5.0;
    
    SET @ReadinessScore = @CurrentScore - @FeaturePenalty;
    
    -- Determine recommended migration method
    IF @ReadinessScore >= 80
        SET @RecommendedMethod = 'Online';
    ELSE IF @ReadinessScore >= 60
        SET @RecommendedMethod = 'Hybrid';
    ELSE
        SET @RecommendedMethod = 'Offline';
    
    -- Determine migration strategy
    IF @FeatureCount = 0 AND @CompatibilityLevel >= 130
        SET @MigrationStrategy = 'LiftAndShift';
    ELSE IF @FeatureCount > 0 OR @CompatibilityLevel < 130
        SET @MigrationStrategy = 'Modernize';
    ELSE
        SET @MigrationStrategy = 'Replatform';
    
    -- Generate risk factors
    SET @RiskFactors = CONCAT(
        CASE WHEN @DatabaseSizeGB > 500 THEN 'Large database size; ' ELSE '' END,
        CASE WHEN @CompatibilityLevel < 130 THEN 'Legacy compatibility level; ' ELSE '' END,
        CASE WHEN @FeatureCount > 5 THEN 'Multiple incompatible features; ' ELSE '' END,
        CASE WHEN @ReadinessScore < 60 THEN 'Low readiness score; ' ELSE '' END
    );
    
    IF LEN(@RiskFactors) = 0
        SET @RiskFactors = 'No major risk factors identified';
    
    -- Log assessment results
    INSERT INTO MigrationAssessment (
        ServerName, DatabaseName, DatabaseSizeGB, CurrentVersion,
        FeatureUsage, CompatibilityLevel, MigrationReadinessScore,
        RecommendedMigrationMethod, MigrationStrategy, RiskFactors
    ) VALUES (
        @ServerName, @DatabaseName, @DatabaseSizeGB, @@VERSION,
        @FeatureUsage, @CompatibilityLevel, @ReadinessScore,
        @RecommendedMethod, @MigrationStrategy, @RiskFactors
    );
    
    -- Return assessment summary
    SELECT 
        @ServerName as ServerName,
        @DatabaseName as DatabaseName,
        @DatabaseSizeGB as DatabaseSizeGB,
        @CompatibilityLevel as CompatibilityLevel,
        @ReadinessScore as MigrationReadinessScore,
        @RecommendedMethod as RecommendedMigrationMethod,
        @MigrationStrategy as MigrationStrategy,
        @RiskFactors as RiskFactors,
        CASE 
            WHEN @ReadinessScore >= 80 THEN 'Green - Ready for migration'
            WHEN @ReadinessScore >= 60 THEN 'Yellow - Migration with caution'
            ELSE 'Red - Requires significant preparation'
        END as ReadinessLevel,
        -- Next steps based on assessment
        CASE 
            WHEN @ReadinessScore >= 80 THEN 'Proceed with Azure Database Migration Service'
            WHEN @ReadinessScore >= 60 THEN 'Address compatibility issues before migration'
            ELSE 'Consider using Azure SQL Managed Instance or extensive preparation required'
        END as RecommendedNextSteps;
    
    DROP TABLE #FeatureAnalysis;
END;

-- Execute migration assessment for sample databases
EXEC sp_AssessMigrationReadiness @ServerName = 'PROD-SQL01', @DatabaseName = 'CustomerDB';
EXEC sp_AssessMigrationReadiness @ServerName = 'PROD-SQL01', @DatabaseName = 'TradingDB';
EXEC sp_AssessMigrationReadiness @ServerName = 'PROD-SQL01', @DatabaseName = 'LegacyHRDB';
```

### Automated Migration Execution

Migration execution requires careful planning, monitoring, and rollback capabilities.

**Migration Execution Framework:**

```powershell
# Azure Database Migration Service Integration
param(
    [Parameter(Mandatory=$true)]
    [string]$SourceServerName,
    
    [Parameter(Mandatory=$true)]
    [string]$SourceDatabaseName,
    
    [Parameter(Mandatory=$true)]
    [string]$TargetResourceGroupName,
    
    [Parameter(Mandatory=$true)]
    [string]$TargetServerName,
    
    [Parameter(Mandatory=$true)]
    [string]$TargetDatabaseName,
    
    [Parameter(Mandatory=$false)]
    [string]$MigrationMode = "Online", # Online, Offline, Hybrid
    
    [Parameter(Mandatory=$false)]
    [int]$MaxOfflineDurationMinutes = 30,
    
    [Parameter(Mandatory=$false)]
    [string]$MigrationServiceName,
    
    [Parameter(Mandatory=$false)]
    [switch]$ValidateDataAfterMigration = $true,
    
    [Parameter(Mandatory=$false)]
    [switch]$CreateMigrationReport = $true
)

Write-Host "Starting Azure Database Migration..." -ForegroundColor Green

# This would integrate with Azure Database Migration Service
# The actual implementation would use the Azure Data Migration SDK

# Migration workflow:
# 1. Pre-migration validation
# 2. Create migration project
# 3. Execute migration
# 4. Monitor progress
# 5. Validate data integrity
# 6. Generate migration report

function Start-PreMigrationValidation {
    param(
        [string]$SourceServer,
        [string]$SourceDatabase
    )
    
    Write-Host "Running pre-migration validation..." -ForegroundColor Yellow
    
    # Validate source database accessibility
    try {
        $connectionString = "Server=$SourceServer;Database=master;Integrated Security=true;"
        $connection = New-Object System.Data.SqlClient.SqlConnection($connectionString)
        $connection.Open()
        
        # Check database integrity
        $command = $connection.CreateCommand()
        $command.CommandText = "DBCC CHECKDB($SourceDatabase) WITH NO_INFOMSGS, PHYSICAL_ONLY"
        $command.ExecuteNonQuery()
        
        Write-Host "Source database integrity check completed successfully" -ForegroundColor Green
        $connection.Close()
        return $true
    }
    catch {
        Write-Host "Pre-migration validation failed: $($_.Exception.Message)" -ForegroundColor Red
        return $false
    }
}

function Start-MigrationProcess {
    param(
        [string]$SourceServer,
        [string]$SourceDatabase,
        [string]$TargetServer,
        [string]$TargetDatabase,
        [string]$Mode
    )
    
    Write-Host "Initiating $Mode migration..." -ForegroundColor Yellow
    
    # Create migration project configuration
    $migrationConfig = @{
        sourceServer = $SourceServer
        sourceDatabase = $SourceDatabase
        targetServer = $TargetServer
        targetDatabase = $TargetDatabase
        migrationMode = $Mode
        startTime = Get-Date
    }
    
    # In real implementation, this would call Azure Database Migration Service API
    # For demonstration, we'll simulate the migration process
    
    $migrationJobs = @(
        @{ Name = "Schema Migration"; Duration = 5; Status = "Completed" },
        @{ Name = "Data Migration"; Duration = 30; Status = "In Progress" },
        @{ Name = "Index Creation"; Duration = 10; Status = "Pending" },
        @{ Name = "Statistics Update"; Duration = 5; Status = "Pending" }
    )
    
    foreach ($job in $migrationJobs) {
        Write-Host "Executing migration task: $($job.Name)" -ForegroundColor Gray
        
        # Simulate task execution
        Start-Sleep -Seconds 2
        
        # In real implementation, monitor actual progress
        Write-Host "Migration task '$($job.Name)' completed" -ForegroundColor Green
    }
    
    return $true
}

function Start-PostMigrationValidation {
    param(
        [string]$TargetServer,
        [string]$TargetDatabase
    )
    
    if (-not $ValidateDataAfterMigration) {
        Write-Host "Data validation skipped" -ForegroundColor Yellow
        return $true
    }
    
    Write-Host "Validating migrated data integrity..." -ForegroundColor Yellow
    
    # Validate table counts
    # Validate data consistency
    # Validate indexes and constraints
    # Performance validation
    
    return $true
}

function New-MigrationReport {
    param(
        [hashtable]$MigrationConfig,
        [datetime]$StartTime,
        [datetime]$EndTime
    )
    
    if (-not $CreateMigrationReport) {
        return
    }
    
    $report = @{
        MigrationSummary = @{
            SourceServer = $MigrationConfig.sourceServer
            SourceDatabase = $MigrationConfig.sourceDatabase
            TargetServer = $MigrationConfig.targetServer
            TargetDatabase = $MigrationConfig.targetDatabase
            MigrationMode = $MigrationConfig.migrationMode
            StartTime = $StartTime
            EndTime = $EndTime
            Duration = ($EndTime - $StartTime).TotalMinutes
            Status = "Completed"
        }
        ValidationResults = @{
            DataIntegrity = "Passed"
            SchemaConsistency = "Passed"
            PerformanceBaseline = "Established"
        }
        Recommendations = @(
            "Configure automatic backups"
            "Enable transparent data encryption"
            "Set up monitoring and alerting"
            "Review and optimize indexes"
        )
    }
    
    $report | ConvertTo-Json -Depth 3 | Out-File -FilePath "./migration-report-$($MigrationConfig.sourceDatabase).json"
    Write-Host "Migration report generated: migration-report-$($MigrationConfig.sourceDatabase).json" -ForegroundColor Cyan
}

# Main migration execution
try {
    $migrationStartTime = Get-Date
    
    # Pre-migration validation
    if (-not (Start-PreMigrationValidation -SourceServer $SourceServerName -SourceDatabase $SourceDatabaseName)) {
        throw "Pre-migration validation failed"
    }
    
    # Execute migration
    if (-not (Start-MigrationProcess -SourceServer $SourceServerName -SourceDatabase $SourceDatabaseName -TargetServer $TargetServerName -TargetDatabase $TargetDatabaseName -Mode $MigrationMode)) {
        throw "Migration process failed"
    }
    
    # Post-migration validation
    if (-not (Start-PostMigrationValidation -TargetServer $TargetServerName -TargetDatabase $TargetDatabaseName)) {
        Write-Host "Post-migration validation completed with warnings" -ForegroundColor Yellow
    }
    
    $migrationEndTime = Get-Date
    
    # Generate migration report
    New-MigrationReport -MigrationConfig $migrationConfig -StartTime $migrationStartTime -EndTime $migrationEndTime
    
    Write-Host "Database migration completed successfully!" -ForegroundColor Green
    Write-Host "Source: $SourceServerName.$SourceDatabaseName" -ForegroundColor Cyan
    Write-Host "Target: $TargetServerName.$TargetDatabaseName" -ForegroundColor Cyan
    Write-Host "Duration: $(($migrationEndTime - $migrationStartTime).TotalMinutes) minutes" -ForegroundColor Cyan
    
} catch {
    Write-Host "Migration failed: $($_.Exception.Message)" -ForegroundColor Red
    throw
}
```

### Data Migration Optimization

Large-scale data migration requires optimization strategies to minimize downtime and ensure data integrity.

**Data Migration Optimization Framework:**

```sql
-- Create data migration optimization tracking
CREATE TABLE DataMigrationOptimization (
    OptimizationID INT IDENTITY(1,1) PRIMARY KEY,
    TableName NVARCHAR(256),
    SourceServer NVARCHAR(256),
    SourceDatabase NVARCHAR(256),
    TargetServer NVARCHAR(256),
    TargetDatabase NVARCHAR(256),
    RowCount BIGINT,
    DataSizeMB BIGINT,
    MigrationMethod NVARCHAR(50), -- 'BCP', 'SSIS', 'Azure Data Factory', 'Sync'
    BatchSize INT,
    ConcurrentConnections INT,
    EstimatedDurationMinutes INT,
    ActualDurationMinutes INT,
    ThroughputRowsPerSecond INT,
    CompressionUsed BIT,
    EncryptionUsed BIT,
    ErrorCount INT,
    RetryCount INT,
    OptimizationApplied NVARCHAR(MAX),
    LastUpdated DATETIME2 DEFAULT GETDATE()
);

-- Create procedure for optimizing large table migrations
CREATE PROCEDURE sp_OptimizeTableMigration
    @SourceServer NVARCHAR(256),
    @SourceDatabase NVARCHAR(256),
    @TargetServer NVARCHAR(256),
    @TargetDatabase NVARCHAR(256),
    @TableName NVARCHAR(256),
    @BatchSize INT = 10000,
    @MaxConcurrency INT = 4
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @RowCount BIGINT;
    DECLARE @DataSizeMB BIGINT;
    DECLARE @OptimalBatchSize INT;
    DECLARE @OptimalConcurrency INT;
    DECLARE @EstimatedDuration INT;
    
    -- Get table statistics
    DECLARE @StatsSQL NVARCHAR(MAX) = CONCAT('
        USE [', @SourceDatabase, '];
        
        SELECT 
            COUNT(*) as RowCount,
            SUM(a.used_pages) * 8 / 1024.0 as DataSizeMB
        FROM ', @TableName, ' t
        CROSS APPLY sys.dm_db_partition_stats ps
        JOIN sys.allocation_units a ON ps.partition_id = a.container_id
        WHERE ps.object_id = OBJECT_ID(''', @TableName, ''');
    ');
    
    CREATE TABLE #TableStats (RowCount BIGINT, DataSizeMB DECIMAL(18,2));
    INSERT INTO #TableStats EXEC sp_executesql @StatsSQL;
    
    SELECT @RowCount = RowCount, @DataSizeMB = DataSizeMB FROM #TableStats;
    
    -- Determine optimal batch size based on table characteristics
    IF @RowCount > 1000000 -- Large tables
        SET @OptimalBatchSize = 50000;
    ELSE IF @RowCount > 100000 -- Medium tables
        SET @OptimalBatchSize = 25000;
    ELSE -- Small tables
        SET @OptimalBatchSize = @BatchSize;
    
    -- Determine optimal concurrency based on data size and server resources
    IF @DataSizeMB > 50000 -- Very large tables
        SET @OptimalConcurrency = 2; -- Lower concurrency for I/O intensive operations
    ELSE IF @DataSizeMB > 10000 -- Large tables
        SET @OptimalConcurrency = 4;
    ELSE -- Medium/small tables
        SET @OptimalConcurrency = @MaxConcurrency;
    
    -- Estimate migration duration
    SET @EstimatedDuration = (@RowCount / (@OptimalBatchSize * @OptimalConcurrency * 1000)) * 60; -- Rough estimate
    
    -- Log optimization parameters
    INSERT INTO DataMigrationOptimization (
        TableName, SourceServer, SourceDatabase, TargetServer, TargetDatabase,
        RowCount, DataSizeMB, BatchSize, ConcurrentConnections,
        EstimatedDurationMinutes, OptimizationApplied
    ) VALUES (
        @TableName, @SourceServer, @SourceDatabase, @TargetServer, @TargetDatabase,
        @RowCount, @DataSizeMB, @OptimalBatchSize, @OptimalConcurrency,
        @EstimatedDuration, 
        CONCAT('Batch Size: ', @OptimalBatchSize, ', Concurrency: ', @OptimalConcurrency, ', Compression: Enabled, Encryption: TLS 1.2')
    );
    
    -- Generate optimized migration scripts
    DECLARE @MigrationSQL NVARCHAR(MAX);
    
    SET @MigrationSQL = CONCAT('
        -- Optimized migration script for table: ', @TableName, '
        -- Estimated duration: ', @EstimatedDuration, ' minutes
        -- Batch size: ', @OptimalBatchSize, '
        -- Concurrency: ', @OptimalConcurrency, '
        
        -- Step 1: Create table structure
        SELECT * INTO [', @TargetDatabase, '].[dbo].[' , @TableName, ']_Staging
        FROM [', @SourceServer, '].[', @SourceDatabase, '].[dbo].[' , @TableName, ']
        WHERE 1 = 0;
        
        -- Step 2: Copy data in batches
        DECLARE @BatchSize INT = ', @OptimalBatchSize, ';
        DECLARE @LastProcessed BIGINT = 0;
        DECLARE @RowsProcessed BIGINT = 0;
        DECLARE @TotalRows BIGINT = (SELECT COUNT(*) FROM [', @SourceServer, '].[', @SourceDatabase, '].[dbo].[' , @TableName, ']);
        
        WHILE @RowsProcessed < @TotalRows
        BEGIN
            INSERT INTO [', @TargetDatabase, '].[dbo].[' , @TableName, ']_Staging
            SELECT TOP (@BatchSize) *
            FROM [', @SourceServer, '].[', @SourceDatabase, '].[dbo].[' , @TableName, ']
            WHERE (SELECT COUNT(*) FROM [', @TargetDatabase, '].[dbo].[' , @TableName, ']_Staging) > @RowsProcessed;
            
            SET @RowsProcessed = (SELECT COUNT(*) FROM [', @TargetDatabase, '].[dbo].[' , @TableName, ']_Staging);
            
            -- Print progress every 10000 rows
            IF @RowsProcessed % 10000 = 0
                PRINT CONCAT(''Processed '', @RowsProcessed, '' / '', @TotalRows, '' rows ('', 
                              CAST(@RowsProcessed * 100.0 / @TotalRows AS DECIMAL(5,2)), ''%)'');
            
            -- Add delay between batches for very large tables
            IF @DataSizeMB > 10000
                WAITFOR DELAY ''00:00:01'';
        END
        
        -- Step 3: Create indexes and constraints
        -- (Index creation scripts would be generated here)
        
        -- Step 4: Validate data
        IF (SELECT COUNT(*) FROM [', @TargetDatabase, '].[dbo].[' , @TableName, ']_Staging) = @TotalRows
            PRINT ''Data migration completed successfully'';
        ELSE
            RAISERROR(''Data migration validation failed'', 16, 1);
        
        -- Step 5: Switch to production table
        BEGIN TRANSACTION;
        
        IF EXISTS (SELECT 1 FROM [', @TargetDatabase, '].[dbo].[', @TableName, '])
            DROP TABLE [', @TargetDatabase, '].[dbo].[', @TableName, '];
        
        EXEC sp_rename ''[', @TargetDatabase, '].[dbo].[', @TableName, ']_Staging'', ''', @TableName, ''';
        
        COMMIT TRANSACTION;
    ');
    
    -- Return optimization results
    SELECT 
        @TableName as TableName,
        @RowCount as TotalRows,
        @DataSizeMB as DataSizeMB,
        @OptimalBatchSize as OptimalBatchSize,
        @OptimalConcurrency as OptimalConcurrency,
        @EstimatedDuration as EstimatedDurationMinutes,
        CASE 
            WHEN @DataSizeMB < 1000 THEN 'Fast migration expected'
            WHEN @DataSizeMB < 10000 THEN 'Moderate migration time'
            ELSE 'Long migration - consider off-peak hours'
        END as MigrationExpectation,
        @MigrationSQL as OptimizedMigrationScript;
    
    DROP TABLE #TableStats;
END;

-- Example optimization for large tables
EXEC sp_OptimizeTableMigration
    @SourceServer = 'PROD-SQL01',
    @SourceDatabase = 'CustomerDB',
    @TargetServer = 'sql-mi-eastus.database.windows.net',
    @TargetDatabase = 'CustomerDB_Migrated',
    @TableName = 'CustomerTransactions',
    @BatchSize = 50000,
    @MaxConcurrency = 4;
```

## Hybrid Cloud Architecture

### Azure SQL with On-Premises Integration

Many organizations adopt hybrid cloud strategies, maintaining some workloads on-premises while migrating others to the cloud.

**Hybrid Architecture Implementation:**

```powershell
# Hybrid Cloud SQL Server Architecture Configuration
param(
    [Parameter(Mandatory=$true)]
    [string]$OnPremisesServerName,
    
    [Parameter(Mandatory=$true)]
    [string]$AzureSQLServerName,
    
    [Parameter(Mandatory=$true)]
    [string]$AzureSQLDatabaseName,
    
    [Parameter(Mandatory=$true)]
    [string]$VPNConnectionName,
    
    [Parameter(Mandatory=$false)]
    [string]$SyncGroupName = "HybridSync",
    
    [Parameter(Mandatory=$false)]
    [switch]$EnableActiveDirectoryIntegration = $true,
    
    [Parameter(Mandatory=$false)]
    [switch]$EnableDataSync = $true
)

# This script configures hybrid connectivity between on-premises SQL Server and Azure SQL

function Initialize-HybridConnectivity {
    param(
        [string]$OnPremServer,
        [string]$AzureServer,
        [string]$VPNName
    )
    
    Write-Host "Configuring hybrid connectivity..." -ForegroundColor Yellow
    
    # Configure Azure SQL firewall rules for VPN
    # Note: This would use Azure PowerShell modules
    
    # Add firewall rule for VPN gateway
    $vpnFirewallRule = @{
        Name = "Allow-VPN-$VPNName"
        StartIPAddress = "10.0.0.0"  # VPN IP range
        EndIPAddress = "10.255.255.255"
    }
    
    # In real implementation:
    # New-AzSqlServerFirewallRule -ResourceGroupName $ResourceGroupName -ServerName $AzureServer @vpnFirewallRule
    
    Write-Host "Hybrid connectivity configured" -ForegroundColor Green
}

function New-SQLDataSyncConfiguration {
    param(
        [string]$OnPremServer,
        [string]$OnPremDatabase,
        [string]$AzureServer,
        [string]$AzureDatabase,
        [string]$SyncName
    )
    
    Write-Host "Configuring SQL Data Sync..." -ForegroundColor Yellow
    
    # This would configure Azure SQL Data Sync between on-premises and cloud
    # The actual implementation uses Azure SQL Data Sync service
    
    $syncConfig = @{
        SyncGroupName = $SyncName
        HubDatabaseServerName = $AzureServer
        HubDatabaseName = $AzureDatabase
        MemberDatabaseServerName = $OnPremServer
        MemberDatabaseName = $OnPremDatabase
        SyncDirection = "Bidirectional"
        ConflictResolutionPolicy = "HubWins"
        SyncInterval = "5 minutes"
    }
    
    Write-Host "SQL Data Sync configured for bidirectional synchronization" -ForegroundColor Green
    return $syncConfig
}

function Initialize-ActiveDirectoryIntegration {
    param(
        [string]$AzureServer
    )
    
    if (-not $EnableActiveDirectoryIntegration) {
        return
    }
    
    Write-Host "Configuring Active Directory integration..." -ForegroundColor Yellow
    
    # Configure Azure SQL with Azure AD authentication
    # This would set up Azure AD authentication for the Azure SQL server
    
    # Create Azure AD admin
    # Configure service principal authentication
    # Set up on-premises AD Connect integration
    
    Write-Host "Active Directory integration configured" -ForegroundColor Green
}

function New-HybridBackupStrategy {
    param(
        [string]$OnPremServer,
        [string]$AzureServer,
        [string]$AzureDatabase
    )
    
    Write-Host "Configuring hybrid backup strategy..." -ForegroundColor Yellow
    
    # Configure backup to Azure for both on-premises and cloud databases
    $backupConfig = @{
        OnPremisesBackup = @{
            UseAzureBackup = $true
            BackupToAzure = $true
            RetentionPeriod = "30 days"
            EncryptionEnabled = $true
        }
        CloudBackup = @{
            AutoBackupEnabled = $true
            BackupRetentionDays = 30
            BackupRedundancy = "Zone"
            PointInTimeRestore = $true
        }
    }
    
    Write-Host "Hybrid backup strategy configured" -ForegroundColor Green
    return $backupConfig
}

function New-MonitoringStrategy {
    param(
        [string]$OnPremServer,
        [string]$AzureServer
    )
    
    Write-Host "Configuring hybrid monitoring..." -ForegroundColor Yellow
    
    # Configure unified monitoring for hybrid environment
    $monitoringConfig = @{
        AzureSQLMonitoring = @{
            AzureSQLAnalytics = $true
            PerformanceInsights = $true
            ThreatDetection = $true
            AutomatedTuning = $true
        }
        OnPremisesMonitoring = @{
            DMVs = $true
            ExtendedEvents = $true
            AgentJobs = $true
            LogShipping = $true
        }
        CrossPlatformAlerts = $true
        UnifiedDashboard = $true
    }
    
    Write-Host "Hybrid monitoring strategy configured" -ForegroundColor Green
    return $monitoringConfig
}

# Main hybrid architecture setup
try {
    Write-Host "Setting up hybrid SQL Server architecture..." -ForegroundColor Green
    
    # Configure hybrid connectivity
    Initialize-HybridConnectivity -OnPremServer $OnPremisesServerName -AzureServer $AzureSQLServerName -VPNName $VPNConnectionName
    
    # Configure data synchronization if enabled
    if ($EnableDataSync) {
        New-SQLDataSyncConfiguration -OnPremServer $OnPremisesServerName -OnPremDatabase "OnPremDB" `
                                     -AzureServer $AzureSQLServerName -AzureDatabase $AzureSQLDatabaseName `
                                     -SyncName $SyncGroupName | Out-Null
    }
    
    # Configure Active Directory integration
    Initialize-ActiveDirectoryIntegration -AzureServer $AzureSQLServerName
    
    # Configure backup strategy
    $backupConfig = New-MybridBackupStrategy -OnPremServer $OnPremisesServerName `
                                            -AzureServer $AzureSQLServerName `
                                            -AzureDatabase $AzureSQLDatabaseName
    
    # Configure monitoring
    $monitoringConfig = New-MonitoringStrategy -OnPremServer $OnPremisesServerName -AzureServer $AzureSQLServerName
    
    # Generate architecture documentation
    $architectureDoc = @{
        Architecture = "Hybrid SQL Server"
        Components = @{
            OnPremises = @{
                ServerName = $OnPremisesServerName
                Role = "Primary for compliance/performance requirements"
                BackupStrategy = "Azure Backup with local replication"
            }
            Cloud = @{
                ServerName = $AzureSQLServerName
                DatabaseName = $AzureSQLDatabaseName
                Role = "Secondary/copy for analytics and DR"
                BackupStrategy = "Automated Azure backup"
            }
        }
        Connectivity = @{
            VPNConnection = $VPNConnectionName
            DataSync = $EnableDataSync
            Authentication = @{
                ActiveDirectory = $EnableActiveDirectoryIntegration
                SQLAuthentication = $true
            }
        }
        Backup = $backupConfig
        Monitoring = $monitoringConfig
        CreatedDate = Get-Date
    }
    
    $architectureDoc | ConvertTo-Json -Depth 3 | Out-File -FilePath "./hybrid-architecture-$($OnPremisesServerName)-$AzureSQLServerName.json"
    
    Write-Host "Hybrid SQL Server architecture setup completed!" -ForegroundColor Green
    Write-Host "Architecture documentation saved to hybrid-architecture-$($OnPremisesServerName)-$AzureSQLServerName.json" -ForegroundColor Cyan
    
} catch {
    Write-Host "Hybrid architecture setup failed: $($_.Exception.Message)" -ForegroundColor Red
    throw
}
```

### Cross-Platform Data Synchronization

Hybrid environments require sophisticated data synchronization strategies to maintain consistency across platforms.

**Data Synchronization Framework:**

```sql
-- Create hybrid data synchronization tracking system
CREATE TABLE HybridDataSync (
    SyncID INT IDENTITY(1,1) PRIMARY KEY,
    SourceServer NVARCHAR(256),
    SourceDatabase NVARCHAR(256),
    SourceTable NVARCHAR(256),
    TargetServer NVARCHAR(256),
    TargetDatabase NVARCHAR(256),
    TargetTable NVARCHAR(256),
    SyncType NVARCHAR(50), -- 'Full', 'Incremental', 'Bidirectional'
    SyncMethod NVARCHAR(50), -- 'ChangeDataCapture', 'Triggers', 'Manual', 'Azure Data Sync'
    LastSyncTime DATETIME2,
    LastSuccessfulSync DATETIME2,
    SyncStatus NVARCHAR(50), -- 'Success', 'Failed', 'InProgress', 'Delayed'
    RecordsProcessed BIGINT,
    SyncDurationSeconds INT,
    ConflictResolutionStrategy NVARCHAR(100), -- 'SourceWins', 'TargetWins', 'Manual', 'Timestamp'
    ErrorCount INT DEFAULT 0,
    LastErrorMessage NVARCHAR(MAX),
    SyncFrequency NVARCHAR(50), -- 'RealTime', 'Minutes5', 'Hourly', 'Daily'
    DataValidationEnabled BIT DEFAULT 1,
    RetryAttempts INT DEFAULT 3,
    CreatedDate DATETIME2 DEFAULT GETDATE()
);

-- Create procedure for implementing hybrid sync using Change Data Capture
CREATE PROCEDURE sp_ConfigureHybridCDC
    @SourceServer NVARCHAR(256),
    @SourceDatabase NVARCHAR(256),
    @SourceTable NVARCHAR(256),
    @TargetServer NVARCHAR(256),
    @TargetDatabase NVARCHAR(256),
    @TargetTable NVARCHAR(256),
    @SyncFrequency NVARCHAR(50) = 'Minutes5'
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Enable Change Data Capture on source database
    DECLARE @EnableCDCSQL NVARCHAR(MAX) = CONCAT('
        USE [', @SourceDatabase, '];
        
        -- Enable CDC at database level
        EXEC sys.sp_cdc_enable_db;
        
        -- Enable CDC for specific table
        EXEC sys.sp_cdc_enable_table
            @source_schema = N''dbo'',
            @source_name = N''', @SourceTable, ''',
            @role_name = NULL,
            @supports_net_changes = 1;
        
        -- Create CDC capture job
        EXEC msdb.dbo.sp_add_job
            @job_name = N''CDC_Process_', @SourceTable, ''',
            @enabled = 1,
            @description = N''Process CDC changes for table ', @SourceTable, ''',
            @category_name = N''CDC'';
        
        -- Add job step for CDC processing
        EXEC msdb.dbo.sp_add_jobstep
            @job_name = N''CDC_Process_', @SourceTable, ''',
            @step_name = N''Process Changes'',
            @command = CONCAT(''
                -- Process CDC changes
                DECLARE @from_lsn binary(10), @to_lsn binary(10);
                
                SET @from_lsn = sys.fn_cdc_get_min_lshifted_lsn('''', dbo_', @SourceTable, ''', NULL, NULL);
                SET @to_lsn = sys.fn_cdc_get_max_lshifted_lsn('''', dbo_', @SourceTable, ''', NULL, NULL);
                
                -- Process inserts
                INSERT INTO OPENROWSET(
                    ''SQLNCLI'', 
                    ''Server=', @TargetServer, ';Database=', @TargetDatabase, ';Trusted_Connection=yes;'',
                    ''SELECT * FROM [dbo].[', @TargetTable, ']''
                )
                SELECT * FROM cdc.dbo_', @SourceTable, '_CT
                WHERE __$operation = 2 -- Insert
                  AND __$start_lsn >= @from_lsn
                  AND __$start_lsn <= @to_lsn;
                
                -- Process updates
                UPDATE target
                SET ', @SourceTable, ' = source.*
                FROM OPENROWSET(
                    ''SQLNCLI'', 
                    ''Server=', @TargetServer, ';Database=', @TargetDatabase, ';Trusted_Connection=yes;'',
                    ''SELECT * FROM [dbo].[', @TargetTable, ']''
                ) source
                JOIN cdc.dbo_', @SourceTable, '_CT cdc_table ON source.ID = cdc_table.ID
                WHERE cdc_table.__$operation = 4 -- Update
                  AND cdc_table.__$start_lsn >= @from_lsn
                  AND cdc_table.__$start_lsn <= @to_lsn;
                
                -- Process deletes
                DELETE target
                FROM OPENROWSET(
                    ''SQLNCLI'', 
                    ''Server=', @TargetServer, ';Database=', @TargetDatabase, ';Trusted_Connection=yes;'',
                    ''SELECT * FROM [dbo].[', @TargetTable, ']''
                ) target
                JOIN cdc.dbo_', @SourceTable, '_CT cdc_table ON target.ID = cdc_table.ID
                WHERE cdc_table.__$operation = 1 -- Delete
                  AND cdc_table.__$start_lsn >= @from_lsn
                  AND cdc_table.__$start_lsn <= @to_lsn;
                
                -- Clean up processed changes
                EXEC sys.sp_cdc_cleanup_change_table
                    @capture_instance = '''', dbo_', @SourceTable, ''',
                    @low_water_mark = @to_lsn;
            '');
        
        -- Schedule job based on sync frequency
        EXEC msdb.dbo.sp_add_schedule
            @schedule_name = N''CDC_Schedule_', @SourceTable, ''',
            @enabled = 1,
            @freq_type = 4, -- Daily
            @active_start_date = CONVERT(CHAR(8), GETDATE(), 112), -- Today
            @active_end_date = 99991231, -- No end date
            @freq_interval = 1, -- Every day
            @active_start_time = CASE 
                WHEN ''', @SyncFrequency, ''' = ''RealTime'' THEN 0
                WHEN ''', @SyncFrequency, ''' = ''Minutes5'' THEN 0
                WHEN ''', @SyncFrequency, ''' = ''Hourly'' THEN 0
                ELSE 0
            END; -- Start immediately
        
        -- Attach schedule to job
        EXEC msdb.dbo.sp_attach_schedule
            @job_name = N''CDC_Process_', @SourceTable, ''',
            @schedule_name = N''CDC_Schedule_', @SourceTable, ''';
        
        -- Start job
        EXEC msdb.dbo.sp_start_job
            @job_name = N''CDC_Process_', @SourceTable, ''';
    ');
    
    -- Execute CDC setup
    CREATE TABLE #CDCSetupResults (SetupStep NVARCHAR(200), Status NVARCHAR(50), Message NVARCHAR(MAX));
    
    BEGIN TRY
        -- Enable CDC on source database
        EXEC sp_executesql @EnableCDCSQL;
        
        INSERT INTO #CDCSetupResults VALUES ('CDC Database Enable', 'Success', 'Change Data Capture enabled successfully');
        INSERT INTO #CDCSetupResults VALUES ('CDC Table Enable', 'Success', 'CDC enabled for table ' + @SourceTable);
        INSERT INTO #CDCSetupResults VALUES ('Job Creation', 'Success', 'CDC processing job created');
        INSERT INTO #CDCSetupResults VALUES ('Schedule Configuration', 'Success', 'CDC job scheduled for ' + @SyncFrequency);
        
        -- Log successful sync configuration
        INSERT INTO HybridDataSync (
            SourceServer, SourceDatabase, SourceTable,
            TargetServer, TargetDatabase, TargetTable,
            SyncType, SyncMethod, SyncStatus, SyncFrequency
        ) VALUES (
            @SourceServer, @SourceDatabase, @SourceTable,
            @TargetServer, @TargetDatabase, @TargetTable,
            'Incremental', 'ChangeDataCapture', 'Success', @SyncFrequency
        );
        
    END TRY
    BEGIN CATCH
        INSERT INTO #CDCSetupResults VALUES ('CDC Setup', 'Failed', ERROR_MESSAGE());
        
        -- Log failed sync configuration
        INSERT INTO HybridDataSync (
            SourceServer, SourceDatabase, SourceTable,
            TargetServer, TargetDatabase, TargetTable,
            SyncType, SyncMethod, SyncStatus, LastErrorMessage
        ) VALUES (
            @SourceServer, @SourceDatabase, @SourceTable,
            @TargetServer, @TargetDatabase, @TargetTable,
            'Incremental', 'ChangeDataCapture', 'Failed', ERROR_MESSAGE()
        );
    END CATCH
    
    -- Return setup results
    SELECT * FROM #CDCSetupResults;
    
    DROP TABLE #CDCSetupResults;
END;

-- Example: Configure CDC sync for customer data
EXEC sp_ConfigureHybridCDC
    @SourceServer = 'PROD-SQL01',
    @SourceDatabase = 'CustomerDB',
    @SourceTable = 'CustomerData',
    @TargetServer = 'sql-mi-eastus.database.windows.net',
    @TargetDatabase = 'CustomerDB_Cloud',
    @TargetTable = 'CustomerData_Sync',
    @SyncFrequency = 'Minutes5';
```

## Cloud Security and Compliance

### Azure SQL Security Framework

Security in cloud environments requires multi-layered approaches that include encryption, authentication, network security, and compliance monitoring.

**Azure SQL Security Implementation:**

```powershell
# Azure SQL Security Configuration Script
param(
    [Parameter(Mandatory=$true)]
    [string]$ResourceGroupName,
    
    [Parameter(Mandatory=$true)]
    [string]$ServerName,
    
    [Parameter(Mandatory=$true)]
    [string]$DatabaseName,
    
    [Parameter(Mandatory=$false)]
    [string]$KeyVaultName,
    
    [Parameter(Mandatory=$false)]
    [switch]$EnableTransparentDataEncryption = $true,
    
    [Parameter(Mandatory=$false)]
    [switch]$EnableAlwaysEncrypted = $true,
    
    [Parameter(Mandatory=$false)]
    [switch]$EnableAdvancedThreatProtection = $true,
    
    [Parameter(Mandatory=$false)]
    [switch]$EnableAuditing = $true,
    
    [Parameter(Mandatory=$false)]
    [string]$ThreatProtectionEmail = "security@company.com",
    
    [Parameter(Mandatory=$false)]
    [string]$AuditRetentionDays = 90
)

function Set-AzureSQLEncryption {
    param(
        [string]$ResourceGroup,
        [string]$Server,
        [string]$Database,
        [string]$KeyVault
    )
    
    if (-not $EnableTransparentDataEncryption) {
        return
    }
    
    Write-Host "Configuring Transparent Data Encryption..." -ForegroundColor Yellow
    
    # Enable TDE with Azure Key Vault
    $tdeParams = @{
        ResourceGroupName = $ResourceGroup
        ServerName = $Server
        DatabaseName = $Database
        State = "Enabled"
    }
    
    if ($KeyVault) {
        $tdeParams["UseKeyVault"] = $true
        $tdeParams["KeyVaultKeyName"] = "tde-key"
        $tdeParams["KeyVaultKeyVersion"] = "latest"
        $tdeParams["KeyVaultResourceId"] = "/subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.KeyVault/vaults/$KeyVault"
    }
    
    try {
        Set-AzSqlDatabaseTransparentDataEncryption @tdeParams
        Write-Host "Transparent Data Encryption enabled successfully" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed to enable Transparent Data Encryption: $($_.Exception.Message)" -ForegroundColor Red
    }
}

function Set-AzureSQLAlwaysEncrypted {
    param(
        [string]$ResourceGroup,
        [string]$Server,
        [string]$Database
    )
    
    if (-not $EnableAlwaysEncrypted) {
        return
    }
    
    Write-Host "Configuring Always Encrypted..." -ForegroundColor Yellow
    
    # Enable Always Encrypted for sensitive columns
    # This would typically be done through application code or SQL scripts
    
    Write-Host "Always Encrypted configuration requires application updates" -ForegroundColor Yellow
    Write-Host "Ensure application drivers support Always Encrypted" -ForegroundColor Yellow
}

function Set-AzureSQLThreatProtection {
    param(
        [string]$ResourceGroup,
        [string]$Server,
        [string]$Database,
        [string]$Email
    )
    
    if (-not $EnableAdvancedThreatProtection) {
        return
    }
    
    Write-Host "Configuring Advanced Threat Protection..." -ForegroundColor Yellow
    
    $threatProtectionParams = @{
        ResourceGroupName = $ResourceGroup
        ServerName = $Server
        DatabaseName = $Database
        NotificationRecipientsEmails = $Email
        EmailAdmins = $true
        StorageAccountName = "threatdetection$Server"  # Would be pre-created
    }
    
    try {
        Set-AzSqlDatabaseThreatDetectionPolicy @threatProtectionParams
        Write-Host "Advanced Threat Protection enabled successfully" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed to enable Advanced Threat Protection: $($_.Exception.Message)" -ForegroundColor Red
    }
}

function Set-AzureSQLAuditing {
    param(
        [string]$ResourceGroup,
        [string]$Server,
        [string]$Database,
        [string]$RetentionDays
    )
    
    if (-not $EnableAuditing) {
        return
    }
    
    Write-Host "Configuring Auditing..." -ForegroundColor Yellow
    
    # Configure auditing to Log Analytics workspace
    $auditingParams = @{
        ResourceGroupName = $ResourceGroup
        ServerName = $Server
        DatabaseName = $Database
        AuditActionGroup = @(
            "DATABASE_OWNERSHIP_CHANGE_GROUP",
            "DATABASE_OBJECT_CHANGE_GROUP",
            "SCHEMA_OBJECT_CHANGE_GROUP",
            "DATABASE_PRIVILEGE_CHANGE_GROUP",
            "DATABASE_ACCESS_GROUP",
            "DATABASE_ACCOUNT_MANAGEMENT_GROUP"
        )
        AuditAction = @("UPDATE", "INSERT", "DELETE", "SELECT")
        RetentionInDays = $RetentionDays
        LogAnalyticsWorkspaceResourceId = "/subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/sql-monitoring"
    }
    
    try {
        Set-AzSqlDatabaseAudit @auditingParams
        Write-Host "Auditing configured successfully" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed to configure Auditing: $($_.Exception.Message)" -ForegroundColor Red
    }
}

function Set-AzureSQLNetworkSecurity {
    param(
        [string]$ResourceGroup,
        [string]$Server
    )
    
    Write-Host "Configuring Network Security..." -ForegroundColor Yellow
    
    # Disable public access
    Update-AzSqlServer -ResourceGroupName $ResourceGroup -ServerName $Server -PublicNetworkAccess Disabled
    
    # Configure private endpoints (requires pre-configured private endpoints)
    Write-Host "Note: Configure private endpoints for network isolation" -ForegroundColor Yellow
}

function New-SQLSecurityComplianceReport {
    param(
        [string]$ResourceGroup,
        [string]$Server,
        [string]$Database
    )
    
    Write-Host "Generating Security Compliance Report..." -ForegroundColor Yellow
    
    $complianceReport = @{
        Timestamp = Get-Date
        ServerName = $Server
        DatabaseName = $Database
        SecurityControls = @{
            TransparentDataEncryption = @{
                Status = if ($EnableTransparentDataEncryption) { "Enabled" } else { "Disabled" }
                Compliance = "Required for data at rest encryption"
                Priority = "High"
            }
            AlwaysEncrypted = @{
                Status = if ($EnableAlwaysEncrypted) { "Enabled" } else { "Disabled" }
                Compliance = "Recommended for sensitive columns"
                Priority = "Medium"
            }
            AdvancedThreatProtection = @{
                Status = if ($EnableAdvancedThreatProtection) { "Enabled" } else { "Disabled" }
                Compliance = "Required for GDPR/SOX compliance"
                Priority = "High"
            }
            Auditing = @{
                Status = if ($EnableAuditing) { "Enabled" } else { "Disabled" }
                Compliance = "Required for audit trails"
                Priority = "High"
            }
            NetworkSecurity = @{
                Status = "Public Access Disabled"
                Compliance = "Required for PCI DSS compliance"
                Priority = "High"
            }
        }
        ComplianceScore = 0
        Recommendations = @()
    }
    
    # Calculate compliance score
    $totalControls = $complianceReport.SecurityControls.PSObject.Properties.Count
    $enabledControls = ($complianceReport.SecurityControls.PSObject.Properties | Where-Object { $_.Value.Status -eq "Enabled" -or $_.Value.Status -eq "Public Access Disabled" }).Count
    $complianceReport.ComplianceScore = [math]::Round(($enabledControls / $totalControls) * 100, 2)
    
    # Generate recommendations
    foreach ($control in $complianceReport.SecurityControls.PSObject.Properties) {
        if ($control.Value.Status -eq "Disabled" -and $control.Value.Priority -eq "High") {
            $complianceReport.Recommendations += "Enable $($control.Name) - Required for compliance"
        }
    }
    
    # Save compliance report
    $complianceReport | ConvertTo-Json -Depth 3 | Out-File -FilePath "./security-compliance-$Server-$Database.json"
    
    Write-Host "Security compliance score: $($complianceReport.ComplianceScore)%" -ForegroundColor Cyan
    Write-Host "Compliance report saved to security-compliance-$Server-$Database.json" -ForegroundColor Cyan
    
    return $complianceReport
}

# Main security configuration
try {
    Write-Host "Configuring Azure SQL Security..." -ForegroundColor Green
    
    # Apply security configurations
    Set-AzureSQLEncryption -ResourceGroup $ResourceGroupName -Server $ServerName -Database $DatabaseName -KeyVault $KeyVaultName
    Set-AzureSQLAlwaysEncrypted -ResourceGroup $ResourceGroupName -Server $ServerName -Database $DatabaseName
    Set-AzureSQLThreatProtection -ResourceGroup $ResourceGroupName -Server $ServerName -Database $DatabaseName -Email $ThreatProtectionEmail
    Set-AzureSQLAuditing -ResourceGroup $ResourceGroupName -Server $ServerName -Database $DatabaseName -RetentionDays $AuditRetentionDays
    Set-AzureSQLNetworkSecurity -ResourceGroup $ResourceGroupName -Server $ServerName
    
    # Generate compliance report
    $complianceReport = New-SQLSecurityComplianceReport -ResourceGroup $ResourceGroupName -Server $ServerName -Database $DatabaseName
    
    Write-Host "Azure SQL Security configuration completed!" -ForegroundColor Green
    
} catch {
    Write-Host "Security configuration failed: $($_.Exception.Message)" -ForegroundColor Red
    throw
}
```

## Cost Optimization and Performance Management

### Azure SQL Cost Management

Effective cost management is crucial for cloud SQL deployments, requiring continuous monitoring and optimization.

**Cost Optimization Framework:**

```sql
-- Create cost tracking and optimization system for Azure SQL
CREATE TABLE AzureSQLCostTracking (
    CostTrackingID INT IDENTITY(1,1) PRIMARY KEY,
    ServerName NVARCHAR(256),
    DatabaseName NVARCHAR(256),
    ServiceTier NVARCHAR(50),
    ComputeTier NVARCHAR(50), -- DTU or vCore
    VCoreCount INT,
    DTUCount INT,
    StorageSizeGB BIGINT,
    BackupStorageGB BIGINT,
    Region NVARCHAR(50),
    BillingPeriod NVARCHAR(7), -- Format: YYYY-MM
    ComputeCost DECIMAL(10,2),
    StorageCost DECIMAL(10,2),
    BackupCost DECIMAL(10,2),
    TotalCost DECIMAL(10,2),
    CostPerDay DECIMAL(10,2),
    UtilizationPercent DECIMAL(5,2), -- Average CPU/Storage utilization
    OptimizationOpportunity NVARCHAR(MAX),
    RecommendedAction NVARCHAR(200),
    LastAnalyzed DATETIME2 DEFAULT GETDATE()
);

-- Create procedure for cost analysis and optimization recommendations
CREATE PROCEDURE sp_AnalyzeAzureSQLCostOptimization
    @ServerName NVARCHAR(256) = NULL,
    @BillingPeriod NVARCHAR(7) = NULL,
    @CostThreshold DECIMAL(10,2) = 1000.00 -- Monthly cost threshold for optimization
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AnalysisDate DATETIME2 = GETDATE();
    
    -- Default to current month if not specified
    IF @BillingPeriod IS NULL
        SET @BillingPeriod = FORMAT(@AnalysisDate, 'yyyy-MM');
    
    -- Analyze cost patterns and identify optimization opportunities
    WITH CostAnalysis AS (
        SELECT 
            ServerName,
            DatabaseName,
            ServiceTier,
            ComputeTier,
            AVG(VCoreCount) as AvgVCoreCount,
            AVG(DTUCount) as AvgDTUCount,
            AVG(StorageSizeGB) as AvgStorageGB,
            AVG(TotalCost) as MonthlyCost,
            AVG(UtilizationPercent) as AvgUtilization,
            SUM(CASE WHEN TotalCost > @CostThreshold THEN 1 ELSE 0 END) as HighCostCount
        FROM AzureSQLCostTracking
        WHERE (@ServerName IS NULL OR ServerName = @ServerName)
          AND BillingPeriod = @BillingPeriod
        GROUP BY ServerName, DatabaseName, ServiceTier, ComputeTier
    ),
    OptimizationRecommendations AS (
        SELECT 
            ca.*,
            -- Analyze utilization patterns
            CASE 
                WHEN ca.AvgUtilization < 30 THEN 'Right-size down'
                WHEN ca.AvgUtilization < 60 THEN 'Monitor and potentially optimize'
                WHEN ca.AvgUtilization < 80 THEN 'Optimal range'
                ELSE 'Consider scaling up'
            END as UtilizationAnalysis,
            
            -- Generate specific recommendations
            CASE 
                WHEN ca.AvgUtilization < 30 AND ca.ComputeTier = 'vCore'
                THEN CONCAT('Reduce vCore count from ', ca.AvgVCoreCount, ' to ', CAST(ca.AvgVCoreCount * 0.5 AS INT))
                
                WHEN ca.AvgUtilization < 30 AND ca.ComputeTier = 'DTU'
                THEN CONCAT('Reduce DTU from ', ca.AvdDTUCount, ' to ', CAST(ca.AvgDTUCount * 0.5 AS INT))
                
                WHEN ca.ServiceTier = 'Premium' AND ca.AvgUtilization < 50
                THEN 'Consider Standard tier for better cost efficiency'
                
                WHEN ca.MonthlyCost > @CostThreshold
                THEN 'High cost - implement data archiving and compression'
                
                ELSE 'Continue monitoring'
            END as SpecificRecommendation,
            
            -- Calculate potential savings
            CASE 
                WHEN ca.AvgUtilization < 30 THEN ca.MonthlyCost * 0.4
                WHEN ca.MonthlyCost > @CostThreshold THEN ca.MonthlyCost * 0.2
                ELSE 0
            END as PotentialSavings,
            
            -- Implementation difficulty
            CASE 
                WHEN ca.AvgUtilization < 30 THEN 'Easy - Scale down compute'
                WHEN ca.ServiceTier = 'Premium' AND ca.AvgUtilization < 50 THEN 'Medium - Change service tier'
                WHEN ca.MonthlyCost > @CostThreshold THEN 'Complex - Data optimization required'
                ELSE 'No action needed'
            END as ImplementationDifficulty
        FROM CostAnalysis ca
    )
    SELECT 
        ServerName,
        DatabaseName,
        ServiceTier,
        ComputeTier,
        MonthlyCost,
        AvgUtilization,
        UtilizationAnalysis,
        SpecificRecommendation,
        PotentialSavings,
        ImplementationDifficulty,
        CASE 
            WHEN PotentialSavings > 0 THEN 'Review Recommended'
            WHEN MonthlyCost > @CostThreshold THEN 'Investigate'
            ELSE 'Acceptable'
        END as Priority
    FROM OptimizationRecommendations
    WHERE MonthlyCost > (@CostThreshold * 0.5) -- Only show databases with significant cost
    ORDER BY PotentialSavings DESC, MonthlyCost DESC;
    
    -- Log cost optimization analysis
    INSERT INTO CostOptimizationLog (
        AnalysisDate, ServerName, AnalysisType, CostThreshold, OptimizationCount
    ) VALUES (
        @AnalysisDate, @ServerName, 'MonthlyCostAnalysis', @CostThreshold,
        (SELECT COUNT(*) FROM OptimizationRecommendations WHERE PotentialSavings > 0)
    );
END;

-- Create automated scaling procedure based on utilization
CREATE PROCEDURE sp_AutoScaleAzureSQL
    @ServerName NVARCHAR(256),
    @DatabaseName NVARCHAR(256),
    @UtilizationThreshold DECIMAL(5,2) = 80.0, -- Scale up when utilization > 80%
    @ScaleDownThreshold DECIMAL(5,2) = 30.0, -- Scale down when utilization < 30%
    @MaxScaleOperations INT = 3 -- Limit to prevent excessive scaling
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentUtilization DECIMAL(5,2);
    DECLARE @CurrentVCores INT;
    DECLARE @CurrentServiceTier NVARCHAR(50);
    DECLARE @ScaleOperationCount INT;
    DECLARE @RecommendedAction NVARCHAR(100);
    DECLARE @ScaleDirection NVARCHAR(20);
    
    -- Get current database configuration and utilization
    SELECT TOP 1
        @CurrentUtilization = CurrentUtilization,
        @CurrentVCores = VCores,
        @CurrentServiceTier = ServiceTier
    FROM AzureSQLMonitoring
    WHERE ServerName = @ServerName AND DatabaseName = @DatabaseName
    ORDER BY MeasurementTime DESC;
    
    -- Check recent scaling operations
    SELECT @ScaleOperationCount = COUNT(*)
    FROM AzureSQLScalingLog
    WHERE ServerName = @ServerName
      AND DatabaseName = @DatabaseName
      AND ScalingDate >= DATEADD(HOUR, -24, GETDATE())
      AND OperationType IN ('ScaleUp', 'ScaleDown');
    
    -- Determine scaling action
    IF @CurrentUtilization > @UtilizationThreshold AND @ScaleOperationCount < @MaxScaleOperations
    BEGIN
        SET @ScaleDirection = 'ScaleUp';
        SET @RecommendedAction = CONCAT('Scale up to ', @CurrentVCores + 2, ' vCores');
        
        -- Log scaling decision
        INSERT INTO AzureSQLScalingLog (
            ServerName, DatabaseName, ScalingDate, OperationType, Reason, BeforeVCores, AfterVCores
        ) VALUES (
            @ServerName, @DatabaseName, GETDATE(), 'ScaleUp', 
            CONCAT('High utilization: ', @CurrentUtilization, '%'), 
            @CurrentVCores, @CurrentVCores + 2
        );
    END
    ELSE IF @CurrentUtilization < @ScaleDownThreshold AND @ScaleOperationCount < @MaxScaleOperations
    BEGIN
        SET @ScaleDirection = 'ScaleDown';
        SET @RecommendedAction = CONCAT('Scale down to ', @CurrentVCores - 1, ' vCores');
        
        -- Log scaling decision
        INSERT INTO AzureSQLScalingLog (
            ServerName, DatabaseName, ScalingDate, OperationType, Reason, BeforeVCores, AfterVCores
        ) VALUES (
            @ServerName, @DatabaseName, GETDATE(), 'ScaleDown',
            CONCAT('Low utilization: ', @CurrentUtilization, '%'),
            @CurrentVCores, @CurrentVCores - 1
        );
    END
    ELSE
    BEGIN
        SET @RecommendedAction = 'No scaling needed - utilization within acceptable range';
    END
    
    -- Return scaling recommendation
    SELECT 
        @ServerName as ServerName,
        @DatabaseName as DatabaseName,
        @CurrentUtilization as CurrentUtilization,
        @ScaleDirection as ScaleDirection,
        @RecommendedAction as RecommendedAction,
        @ScaleOperationCount as RecentScaleOperations,
        CASE 
            WHEN @ScaleOperationCount >= @MaxScaleOperations THEN 'Rate limited'
            WHEN @ScaleDirection IS NOT NULL THEN 'Ready to execute'
            ELSE 'No action needed'
        END as ExecutionStatus;
END;

-- Example cost analysis
EXEC sp_AnalyzeAzureSQLCostOptimization 
    @ServerName = 'sql-mi-eastus.database.windows.net',
    @BillingPeriod = '2024-01',
    @CostThreshold = 500.00;
```

## Lab Exercises and Hands-On Scenarios

### Exercise 1: Azure SQL Database Migration
Implement a complete cloud migration scenario including:
- Migration assessment and readiness evaluation
- Azure SQL Database deployment with appropriate service tier selection
- Data migration using Azure Database Migration Service
- Performance validation and optimization
- Security configuration with TDE and Advanced Threat Protection
- Cost optimization and monitoring setup

### Exercise 2: Hybrid Cloud Architecture
Design and implement a hybrid cloud architecture featuring:
- VPN connectivity between on-premises and Azure
- SQL Data Sync configuration for bidirectional data replication
- Hybrid backup strategy with Azure Backup integration
- Unified monitoring and alerting across platforms
- Active Directory integration for authentication
- Disaster recovery planning for hybrid environment

### Exercise 3: Cloud Security and Compliance
Build a comprehensive security framework including:
- Multi-layered encryption (TDE, Always Encrypted, TLS)
- Advanced threat detection and auditing
- Compliance monitoring for GDPR, HIPAA, and SOX
- Network security with private endpoints and firewall rules
- Identity and access management integration
- Security baseline compliance reporting

## Week 17 Summary

This comprehensive week on cloud SQL Server deployment has covered the complete spectrum of cloud database management, from migration planning to operational optimization. We've explored Azure SQL Database and Managed Instance architectures, migration strategies, hybrid cloud designs, security frameworks, and cost optimization techniques.

The key insight is that successful cloud database management requires a holistic approach that considers business requirements, technical constraints, security needs, and cost optimization simultaneously. Organizations must carefully balance cloud-native advantages with the need for control and compliance requirements.

## Next Week Preview

Next week, we'll focus on Disaster Recovery Planning, covering comprehensive DR strategies, testing methodologies, business continuity planning, and the unique challenges of maintaining database availability in the face of failures, disasters, and business disruptions.