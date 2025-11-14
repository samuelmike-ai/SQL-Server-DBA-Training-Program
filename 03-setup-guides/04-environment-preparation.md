# SQL Server Environment Preparation Guide

## Table of Contents
1. [Virtual Machine Planning and Setup](#vm-planning)
2. [Operating System Preparation](#os-preparation)
3. [Network Configuration](#network-config)
4. [Security Framework](#security-framework)
5. [Storage and Disk Management](#storage-management)
6. [High Availability Environment](#high-availability)
7. [Development vs Production Environments](#env-types)
8. [Monitoring and Alerting Infrastructure](#monitoring)
9. [Disaster Recovery Setup](#disaster-recovery)
10. [Environment Validation and Testing](#validation)

## Virtual Machine Planning and Setup {#vm-planning}

### Step 1: Infrastructure Requirements Assessment

**Virtual Machine Sizing Guidelines:**
```
Development Environment (Minimum):
- vCPUs: 2-4 cores
- RAM: 8-16 GB
- Storage: 100-200 GB (SSD preferred)
- Network: 1 Gbps

Development Environment (Recommended):
- vCPUs: 4-8 cores
- RAM: 16-32 GB
- Storage: 200-500 GB (NVMe SSD)
- Network: 1 Gbps

Production Environment (Minimum):
- vCPUs: 8-16 cores
- RAM: 32-64 GB
- Storage: 500 GB - 2 TB (Enterprise SSD)
- Network: 10 Gbps

Production Environment (Recommended):
- vCPUs: 16-32 cores
- RAM: 64-128 GB
- Storage: 1-5 TB (Enterprise NVMe)
- Network: 10-25 Gbps
```

**Capacity Planning Worksheet:**
```sql
-- Create infrastructure assessment template
CREATE TABLE #InfrastructureAssessment (
    AssessmentID INT IDENTITY(1,1),
    EnvironmentType NVARCHAR(50), -- Development, Test, Staging, Production
    PurposeDescription NVARCHAR(200),
    ExpectedUsers INT,
    ExpectedTransactionsPerHour BIGINT,
    DataSizeGB DECIMAL(10,2),
    GrowthRatePercent DECIMAL(5,2),
    AvailabilityRequirement NVARCHAR(50), -- 99.5%, 99.9%, 99.99%
    RecoveryTimeObjectiveHours INT,
    RecoveryPointObjectiveMinutes INT,
    ComplianceRequirements NVARCHAR(200),
    BudgetTier NVARCHAR(50), -- Basic, Standard, Premium, Enterprise
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    CreatedBy NVARCHAR(100) DEFAULT SUSER_SNAME()
);

-- Sample assessment entries
INSERT INTO #InfrastructureAssessment 
(EnvironmentType, PurposeDescription, ExpectedUsers, ExpectedTransactionsPerHour, 
 DataSizeGB, GrowthRatePercent, AvailabilityRequirement, RecoveryTimeObjectiveHours, 
 RecoveryPointObjectiveMinutes, ComplianceRequirements, BudgetTier)
VALUES 
('Development', 'Development and testing', 5, 1000, 10, 50, '99.5%', 8, 60, 'None', 'Basic'),
('Staging', 'Pre-production testing', 10, 5000, 50, 30, '99.9%', 4, 30, 'SOX', 'Standard'),
('Production', 'Live business operations', 500, 100000, 500, 25, '99.99%', 1, 15, 'SOX, GDPR, HIPAA', 'Premium');

-- Generate recommendations
SELECT 
    EnvironmentType,
    PurposeDescription,
    ExpectedUsers,
    ExpectedTransactionsPerHour,
    DataSizeGB,
    GrowthRatePercent,
    CASE 
        WHEN ExpectedUsers <= 10 AND ExpectedTransactionsPerHour <= 10000 THEN '4 vCPUs, 16 GB RAM'
        WHEN ExpectedUsers <= 100 AND ExpectedTransactionsPerHour <= 100000 THEN '8 vCPUs, 32 GB RAM'
        WHEN ExpectedUsers <= 500 AND ExpectedTransactionsPerHour <= 1000000 THEN '16 vCPUs, 64 GB RAM'
        ELSE '32+ vCPUs, 128+ GB RAM'
    END AS RecommendedConfiguration,
    CASE 
        WHEN AvailabilityRequirement = '99.5%' THEN 'Single VM with backups'
        WHEN AvailabilityRequirement = '99.9%' THEN 'VM with failover clustering'
        ELSE 'Availability Group with multiple replicas'
    END AS RecommendedAvailability,
    BudgetTier
FROM #InfrastructureAssessment;
```

### Step 2: Hypervisor Selection and Setup

**VMware vSphere Configuration:**
```powershell
# Create VM specification for SQL Server
$VMConfig = @{
    Name = "SQLServer2019"
    MemoryGB = 32
    NumCpu = 8
    DiskGB = 500
    DiskType = "Thin"
    Network = "Production Network"
    PortGroup = "VLAN_100_Production"
    Datastore = "ESX01-Production-DS"
    GuestOS = "windows2019_64Guest"
}

# VM creation script for vSphere
$VM = New-VM -Name $VMConfig.Name -MemoryGB $VMConfig.MemoryGB -NumCpu $VMConfig.NumCpu 
$VM = Set-VM -VM $VMConfig.Name -DiskGB $VMConfig.DiskGB -DiskType $VMConfig.DiskType
$VM = Set-VMNetworkAdapter -VM $VMConfig.Name -NetworkName $VMConfig.Network -PortGroup $VMConfig.PortGroup
$VM = Set-VMDatastore -VM $VMConfig.Name -Datastore $VMConfig.Datastore

# Configure VM hardware settings
$VMView = Get-VM $VMConfig.Name | Get-View
$VMConfigSpec = New-Object VMware.Vim.VirtualMachineConfigSpec

# Enable hardware acceleration
$VMConfigSpec.tools = New-Object VMware.Vim.ToolsConfigInfo
$VMConfigSpec.tools.ToolsUpgradePolicy = "Manual"

# Set VMware Tools settings
$VMConfigSpec.ToolsInfo.ToolsLatest = $true
$VMConfigSpec.ToolsInfo.VmToolSupport = $true

$VMView.ReconfigVM($VMConfigSpec)
```

**Microsoft Hyper-V Configuration:**
```powershell
# Create VM using Hyper-V PowerShell
$VMName = "SQLServer2022"
$MemoryStartupBytes = 32GB
$MemoryMinimumBytes = 16GB
$MemoryMaximumBytes = 64GB
$vSwitchName = "Production_Switch"
$vSwitchPortProfile = "Production_PortProfile"

# Create virtual machine
New-VM -Name $VMName -MemoryStartupBytes $MemoryStartupBytes `
    -BootDevice VHD `
    -Generation 2 `
    -SwitchName $vSwitchName

# Configure dynamic memory
Set-VM -Name $VMName -DynamicMemoryEnabled $true `
    -MemoryMinimumBytes $MemoryMinimumBytes `
    -MemoryMaximumBytes $MemoryMaximumBytes

# Configure CPU settings
Set-VM -Name $VMName -ProcessorCount 8 `
    -StaticMemory $false

# Configure checkpoint settings (production should have disabled)
Set-VM -Name $VMName -CheckpointType ProductionOnly

# Enable guest services
Enable-VMIntegrationService -Name "Guest Service Interface" -VMName $VMName

# Create and attach VHDX
$VMPath = "C:\Hyper-V\$VMName\Virtual Hard Disks"
New-Item -Path $VMPath -ItemType Directory

New-VHD -Path "$VMPath\$VMName.vhdx" -SizeBytes 500GB

Add-VMHardDiskDrive -VMName $VMName -Path "$VMPath\$VMName.vhdx"
```

### Step 3: Cloud Infrastructure Deployment

**Azure Virtual Machine Deployment:**
```powershell
# Install Azure PowerShell module
Install-Module -Name Az -Force

# Connect to Azure
Connect-AzAccount

# Create resource group
$ResourceGroupName = "rg-sql-server-prod"
$Location = "East US"
New-AzResourceGroup -Name $ResourceGroupName -Location $Location

# Create storage account for VM
$StorageAccountName = "stsqlserverprod$(Get-Random -Minimum 100 -Maximum 999)"
New-AzStorageAccount -ResourceGroupName $ResourceGroupName `
    -AccountName $StorageAccountName `
    -Location $Location `
    -SkuName Standard_LRS

# Define VM configuration
$VMConfig = @{
    ResourceGroupName = $ResourceGroupName
    VMName = "VM-SQLServer2022"
    Location = $Location
    Size = "Standard_D16s_v3" # 16 vCPUs, 64 GB RAM
    OSDiskSizeGB = 1024
    OSDiskType = "Premium_LRS"
    SubnetName = "snet-production"
    VNetName = "vnet-production"
}

# Create virtual network subnet
$Subnet = New-AzVirtualNetworkSubnetConfig -Name $VMConfig.SubnetName `
    -AddressPrefix 10.0.1.0/24

$VNet = New-AzVirtualNetwork -ResourceGroupName $ResourceGroupName `
    -Name $VMConfig.VNetName `
    -Location $Location `
    -AddressPrefix 10.0.0.0/16 `
    -Subnet $Subnet

# Create public IP address
$PublicIP = New-AzPublicIpAddress -ResourceGroupName $ResourceGroupName `
    -Name "pip-$($VMConfig.VMName)" `
    -Location $Location `
    -AllocationMethod Static `
    -Sku Standard

# Create network security group
$NSG = New-AzNetworkSecurityGroup -ResourceGroupName $ResourceGroupName `
    -Location $Location `
    -Name "nsg-$($VMConfig.VMName)"

# Allow SQL Server port
$NSGRule = New-AzNetworkSecurityRuleConfig -Name "Allow-SQL-Server" `
    -Protocol Tcp `
    -Direction Inbound `
    -Priority 1000 `
    -SourceAddressPrefix * `
    -SourcePortRange * `
    -DestinationAddressPrefix * `
    -DestinationPortRange 1433 `
    -Access Allow

$NSG.SecurityRules.Add($NSGRule)
Set-AzNetworkSecurityGroup -NetworkSecurityGroup $NSG

# Create network interface
$NIC = New-AzNetworkInterface -ResourceGroupName $ResourceGroupName `
    -Name "nic-$($VMConfig.VMName)" `
    -Location $Location `
    -SubnetId $VNet.Subnets[0].Id `
    -PublicIpAddressId $PublicIP.Id `
    -NetworkSecurityGroupId $NSG.Id

# Get Windows Server 2022 image
$Publisher = "MicrosoftWindowsServer"
$Offer = "WindowsServer"
$Sku = "2022-Datacenter"

$VMImage = Get-AzVMImage -Location $Location -PublisherName $Publisher -Offer $Offer -Sku $Sku | Sort-Object Version -Descending | Select-Object -First 1

# Create VM configuration
$VM = New-AzVMConfig -VMName $VMConfig.VMName -VMSize $VMConfig.Size

# Set VM properties
$VM = Set-AzVMOperatingSystem -VM $VM -Windows -ComputerName $VMConfig.VMName `
    -Credential $Credential -ProvisionVMAgent -EnableAutoUpdate

# Add VM image
$VM = Add-AzVMNetworkInterface -VM $VM -Id $NIC.Id

# Set OS disk configuration
$OSDiskName = "osdisk-$($VMConfig.VMName)"
$OSDiskUri = "https://$StorageAccountName.blob.core.windows.net/vhds/$OSDiskName.vhd"

$VM = Set-AzVMOSDisk -VM $VM -Name $OSDiskName -VhdUri $OSDiskUri `
    -Caching ReadWrite -StorageAccountType $VMConfig.OSDiskType

# Create the VM
New-AzVM -ResourceGroupName $ResourceGroupName -Location $Location -VM $VM
```

## Operating System Preparation {#os-preparation}

### Step 1: Windows Server Optimization

**PowerShell Script for Server Optimization:**
```powershell
# SQL Server Windows Server Optimization Script
# Run as Administrator

# Disable unnecessary Windows features
Disable-WindowsOptionalFeature -Online -FeatureName IIS-WebServerRole
Disable-WindowsOptionalFeature -Online -FeatureName IIS-WebServer
Disable-WindowsOptionalFeature -Online -FeatureName IIS-CommonHttpFeatures
Disable-WindowsOptionalFeature -Online -FeatureName IIS-HttpErrors
Disable-WindowsOptionalFeature -Online -FeatureName IIS-HttpRedirect
Disable-WindowsOptionalFeature -Online -FeatureName IIS-ApplicationDevelopment

# Configure performance settings for SQL Server
# Set high performance power plan
powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c

# Disable Windows Updates during installation
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name "NoAutoUpdate" -Type DWord -Value 1

# Configure network settings
# Disable TCP chimney offload if causing issues
netsh int ip set global icmpredirects=disabled
netsh int tcp set global rss=enabled
netsh int tcp set global netdma=enabled
netsh int tcp set global ecncapability=disabled
netsh int tcp set global timestamps=disabled

# Configure memory management
# Set large page support
Enable-LargeSystemCache

# Disable Prefetch and Superfetch (can interfere with database I/O)
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management\PrefetchParameters" /v EnablePrefetcher /t REG_DWORD /d 0 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management\PrefetchParameters" /v EnableSuperfetch /t REG_DWORD /d 0 /f

# Configure file system
# Enable NTFS compression for OS drive (not for data drives)
fsutil behavior set disablecompression 0

# Optimize disk I/O scheduler
# For SSDs, use deadline scheduler
fsutil behavior set DisableDeleteNotify 1

# Configure time synchronization
w32tm /config /syncfromflags:MANUAL /manualpeerlist:"0.pool.ntp.org,1.pool.ntp.org,2.pool.ntp.org" /reliable:YES
w32tm /resync /force

# Install .NET Framework
# For SQL Server 2022
Add-WindowsFeature -Name "NET-Framework-Features" -IncludeAllSubFeature -IncludeManagementTools

# Install Windows PowerShell 5.0 if not present
# Already included in Windows Server 2019/2022

# Configure Windows Firewall
# Allow SQL Server port
netsh advfirewall firewall add rule name="SQL Server" dir=in action=allow protocol=TCP localport=1433
netsh advfirewall firewall add rule name="SQL Server Analysis Services" dir=in action=allow protocol=TCP localport=2383
netsh advfirewall firewall add rule name="SQL Server Browser" dir=in action=allow protocol=UDP localport=1434

# Disable Windows Defender real-time protection for SQL Server folders
Add-MpPreference -ExclusionPath "C:\Program Files\Microsoft SQL Server\"
Add-MpPreference -ExclusionPath "D:\SQLData\"  # Adjust path as needed
Add-MpPreference -ExclusionPath "C:\SQLLogs\"  # Adjust path as needed

# Configure Visual Effects for optimal performance
$VisualFX = Get-WmiObject -Class Win32_PerfRawData_PerfOS_System
Set-ItemProperty -Path "HKCU:\Control Panel\Desktop" -Name "VisualFXSetting" -Type DWord -Value 3

Write-Host "Windows Server optimization completed. Please restart the server." -ForegroundColor Green
```

### Step 2: Storage Configuration

**Disk Partitioning Strategy:**
```
Recommended disk layout for SQL Server:

System Drive (C:):
- Size: 60-100 GB
- File System: NTFS
- Allocation Unit Size: 4096 bytes
- Purpose: Windows OS, SQL Server binaries

Data Drive (D:):
- Size: Variable (depends on data volume)
- File System: NTFS
- Allocation Unit Size: 8192 bytes (for large files)
- Purpose: Database data files (.mdf, .ndf)
- Performance: SSD/NVMe recommended

Log Drive (E:):
- Size: 20-30% of data drive size
- File System: NTFS
- Allocation Unit Size: 8192 bytes
- Purpose: Transaction log files (.ldf)
- Performance: Separate physical disk

Backup Drive (F:):
- Size: 150% of data drive size
- File System: NTFS
- Allocation Unit Size: 4092 bytes
- Purpose: Backup files
- Performance: Can be slower disk

TempDB Drive (T:):
- Size: 25% of data drive size
- File System: NTFS
- Allocation Unit Size: 8192 bytes
- Purpose: TempDB files
- Performance: High-performance storage
```

**Disk Configuration Script:**
```powershell
# Create and configure disks for SQL Server
function Initialize-SqlServerDisks {
    param(
        [string[]]$DiskPaths = @("D:", "E:", "F:", "T:"),
        [int[]]$DiskSizes = @(100, 50, 150, 25) # GB
    )
    
    $DriveLetter = 0
    for ($i = 0; $i -lt $DiskPaths.Count; $i++) {
        $DriveLetter = [int]([char]$DiskPaths[$i][0])
        $DriveLetter += 1
        
        # Get physical disk
        $Disk = Get-Disk | Where-Object { $_.Number -eq $i -and $_.PartitionStyle -eq "RAW" }
        
        if ($Disk) {
            # Initialize disk
            $Disk | Initialize-Disk -PartitionStyle GPT -PassThru
            
            # Create partition
            $Partition = $Disk | New-Partition -DriveLetter $DiskPaths[$i][0] -UseMaximumSize
            
            # Format disk
            $Partition | Format-Volume -FileSystem NTFS -NewFileSystemLabel "SQL$($DiskPaths[$i][0])Data"
            
            # Set allocation unit size
            $Volume = Get-Volume -DriveLetter $DiskPaths[$i][0]
            $Volume.FileSystemLabel = "SQL$($DiskPaths[$i][0])Data"
        }
    }
}

# Configure storage performance for databases
function Set-DatabaseStorageSettings {
    param([string]$DriveLetter = "D:")
    
    # Disable indexing on database drives
    Get-WmiObject -Class Win32_Volume -Filter "DriveLetter='$DriveLetter'" | 
    Set-WmiInstance -Arguments @{IndexingEnabled=$false}
    
    # Set disk timeout
    $regPath = "HKLM\SYSTEM\CurrentControlSet\Services\Disk"
    Set-ItemProperty -Path $regPath -Name "TimeOutValue" -Type DWord -Value 60
    
    # Optimize disk performance
    fsutil behavior set DisableLastAccess 1
    fsutil behavior set DisableDeleteNotify 0
}
```

### Step 3: Network Configuration

**Network Optimization Settings:**
```powershell
# SQL Server Network Configuration
# Run as Administrator

# Configure network adapter settings
function Set-SqlServerNetworkConfig {
    $Adapters = Get-NetAdapter | Where-Object { $_.Status -eq "Up" -and $_.PhysicalMediaType -ne "Wireless" }
    
    foreach ($Adapter in $Adapters) {
        Write-Host "Configuring adapter: $($Adapter.Name)"
        
        # Enable Jumbo Frames (if supported)
        $Adapter | Set-NetAdapterAdvancedProperty -DisplayName "Jumbo Packet" -DisplayValue "9014 Bytes" -NoRestart
        
        # Set receive and send buffer sizes
        $Adapter | Set-NetAdapterAdvancedProperty -DisplayName "Receive Buffers" -DisplayValue "4096" -NoRestart
        $Adapter | Set-NetAdapterAdvancedProperty -DisplayName "Transmit Buffers" -DisplayValue "4096" -NoRestart
        
        # Enable flow control
        $Adapter | Set-NetAdapterAdvancedProperty -DisplayName "Flow Control" -DisplayValue "Enabled" -NoRestart
        
        # Set interrupt moderation
        $Adapter | Set-NetAdapterAdvancedProperty -DisplayName "Interrupt Moderation" -DisplayValue "Moderate" -NoRestart
        
        # Disable power saving
        $Adapter | Set-NetAdapterPowerManagement -WakeOnMagicPacket Enabled -DeviceSleepOnDisconnect Disabled
    }
}

# Configure TCP/IP settings for SQL Server
function Set-TcpIpConfig {
    $regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters"
    
    # Set TCP window scaling
    Set-ItemProperty -Path $regPath -Name "Tcp1323Opts" -Type DWord -Value 3
    
    # Set default TCP window size
    Set-ItemProperty -Path $regPath -Name "DefaultTcpWindowSize" -Type DWord -Value 65536
    
    # Set TCP window scaling threshold
    Set-ItemProperty -Path $regPath -Name "DefaultTcpWindowScalingThreshold" -Type DWord -Value 32768
    
    # Disable TCP chimney offload
    Set-ItemProperty -Path $regPath -Path $regPath -Name "EnableTCPChimney" -Type DWord -Value 0
    Set-ItemProperty -Path $regPath -Name "EnableRSS" -Type DWord -Value 1
    Set-ItemProperty -Path $regPath -Name "EnableTCPA" -Type DWord -Value 1
    
    # Optimize TCP keepalive settings
    Set-ItemProperty -Path $regPath -Name "TCPKeepAliveTime" -Type DWord -Value 7200000  # 2 hours
    Set-ItemProperty -Path $regPath -Name "TCPKeepAliveInterval" -Type DWord -Value 60000   # 1 minute
    Set-ItemProperty -Path $regPath -Name "TCPDelAckTicks" -Type DWord -Value 2            # 200ms
}

# Configure DNS settings
function Set-DnsConfig {
    # Primary DNS: Enterprise DNS server
    $PrimaryDns = "192.168.1.10"
    $SecondaryDns = "192.168.1.11"
    
    # Set DNS for all active network adapters
    Get-NetAdapter | Where-Object { $_.Status -eq "Up" } | ForEach-Object {
        Set-DnsClientServerAddress -InterfaceAlias $_.Name -ServerAddresses @($PrimaryDns, $SecondaryDns)
    }
}

# Execute network configuration
Set-SqlServerNetworkConfig
Set-TcpIpConfig
Set-DnsConfig

Write-Host "Network configuration completed for SQL Server." -ForegroundColor Green
```

## Network Configuration {#network-config}

### Step 1: Network Security Zones

**Network Architecture Design:**
```
┌─────────────────────────────────────────────────────────────┐
│                      FIREWALL                               │
├─────────────────────────────────────────────────────────────┤
│  DMZ Zone (10.0.1.0/24)                                    │
│  ├─ Web Servers                                              │
│  └─ Application Servers                                      │
├─────────────────────────────────────────────────────────────┤
│  Application Zone (10.0.2.0/24)                            │
│  ├─ Middleware Servers                                       │
│  └─ Business Application Servers                             │
├─────────────────────────────────────────────────────────────┤
│  Database Zone (10.0.3.0/24)                                │
│  ├─ Primary SQL Server (10.0.3.10)                          │
│  ├─ Secondary SQL Server (10.0.3.11)                        │
│  ├─ Reporting Server (10.0.3.12)                            │
│  └─ Analysis Services (10.0.3.13)                           │
├─────────────────────────────────────────────────────────────┤
│  Management Zone (10.0.4.0/24)                              │
│  ├─ Admin Workstations                                       │
│  ├─ Monitoring Servers                                       │
│  └─ Backup Servers                                           │
└─────────────────────────────────────────────────────────────┘
```

**Network Security Rules:**
```powershell
# Create network security group rules for Azure/Azure Stack
$NSGRule1 = New-AzNetworkSecurityRuleConfig -Name "SQL-Server-Inbound" `
    -Protocol TCP -Direction Inbound -Priority 1000 `
    -SourceAddressPrefix * -SourcePortRange * `
    -DestinationAddressPrefix 10.0.3.0/24 -DestinationPortRange 1433 `
    -Access Allow

$NSGRule2 = New-AzNetworkSecurityRuleConfig -Name "SQL-Backup-Port" `
    -Protocol TCP -Direction Inbound -Priority 1001 `
    -SourceAddressPrefix 10.0.4.0/24 -SourcePortRange * `
    -DestinationAddressPrefix 10.0.3.0/24 -DestinationPortRange 9999 `
    -Access Allow

$NSGRule3 = New-AzNetworkSecurityRuleConfig -Name "Management-RDP" `
    -Protocol TCP -Direction Inbound -Priority 1002 `
    -SourceAddressPrefix 10.0.4.0/24 -SourcePortRange * `
    -DestinationAddressPrefix 10.0.3.0/24 -DestinationPortRange 3389 `
    -Access Allow

# Block all other inbound traffic
$NSGRuleDeny = New-AzNetworkSecurityRuleConfig -Name "Deny-All-Inbound" `
    -Protocol * -Direction Inbound -Priority 4000 `
    -SourceAddressPrefix * -SourcePortRange * `
    -DestinationAddressPrefix * -DestinationPortRange * `
    -Access Deny
```

### Step 2: Load Balancing Configuration

**Azure Load Balancer Setup:**
```powershell
# Create Azure Load Balancer for SQL Server Always On
$ResourceGroupName = "rg-sql-server-prod"
$Location = "East US"
$LbName = "lb-sql-server"

# Create public IP
$PublicIP = New-AzPublicIpAddress -ResourceGroupName $ResourceGroupName `
    -Name "pip-$LbName" -Location $Location `
    -AllocationMethod Static -Sku Standard

# Create load balancer
$FrontEndIp = New-AzLoadBalancerFrontendIpConfig -Name "frontend" -PublicIpAddress $PublicIP
$BackendPool = New-AzLoadBalancerBackendAddressPoolConfig -Name "backend"

# Configure health probe
$HealthProbe = New-AzLoadBalancerProbeConfig -Name "sql-probe" `
    -Protocol Tcp -Port 1433 -IntervalInSeconds 15 -ProbeCount 2

# Create load balancing rule
$LbRule = New-AzLoadBalancerRuleConfig -Name "sql-rule" `
    -FrontendIpConfiguration $FrontEndIp -BackendAddressPool $BackendPool `
    -Probe $HealthProbe -Protocol Tcp -FrontendPort 1433 -BackendPort 1433

# Create load balancer
$LoadBalancer = New-AzLoadBalancer -ResourceGroupName $ResourceGroupName `
    -Name $LbName -Location $Location -Sku Standard `
    -FrontendIpConfiguration $FrontEndIp -BackendAddressPool $BackendPool `
    -LoadBalancingRule $LbRule -Probe $HealthProbe
```

### Step 3: VPN and Remote Access

**Site-to-Site VPN Configuration:**
```powershell
# Azure VPN Gateway configuration
$GatewayName = "vpn-gateway-sql"
$VpnType = "RouteBased"
$VpnSku = "VpnGw3"
$GatewaySubnet = New-AzVirtualNetworkSubnetConfig -Name "GatewaySubnet" `
    -AddressPrefix 10.0.5.0/27

# Create virtual network for VPN Gateway
$VNet = New-AzVirtualNetwork -ResourceGroupName $ResourceGroupName `
    -Name "vnet-sql-vpn" -Location $Location `
    -AddressPrefix 10.0.0.0/16 -Subnet $GatewaySubnet

# Create VPN Gateway
$Gateway = New-AzVirtualNetworkGateway -ResourceGroupName $ResourceGroupName `
    -Name $GatewayName -Location $Location `
    -VirtualNetwork $VNet -GatewayType Vpn -VpnType $VpnType `
    -PublicIpAddressId $PublicIP.Id -Sku $VpnSku

# On-premises VPN device configuration
# Update with your on-premises VPN device settings
$VpnConnection = New-AzVirtualNetworkGatewayConnection -ResourceGroupName $ResourceGroupName `
    -Name "sql-vpn-connection" -Location $Location `
    -VirtualNetworkGateway1 $Gateway -LocalNetworkGateway2 $LocalGateway `
    -ConnectionType IPsec -SharedKey "YourSharedSecretKey" -EnableBgp $false
```

## Security Framework {#security-framework}

### Step 1: Active Directory Integration

**Domain Controller Setup for SQL Server:**
```powershell
# Install and configure Active Directory Domain Services
Import-Module ADDSDeployment

$DomainName = "corp.company.com"
$NetBIOSName = "CORP"
$SafeModePassword = ConvertTo-SecureString "SecurePassword123!" -AsPlainText -Force
$DomainNetBIOSName = $NetBIOSName

# Install AD DS
Install-ADDSDomainController -DomainName $DomainName `
    -SiteName "Default-First-Site-Name" `
    -SkipAutoDnsDNSRegistration `
    -SkipAutoADAccount `
    -SafeModeAdministratorPassword $SafeModePassword `
    -NoRebootOnCompletion

# Install AD DS and promote to domain controller
Install-ADDSDomain -DomainName $DomainName `
    -SiteName "Default-First-Site-Name" `
    -SkipAutoDnsDNSRegistration `
    -SkipAutoADAccount `
    -SafeModeAdministratorPassword $SafeModePassword `
    -NoRebootOnCompletion -Force

# Configure SQL Server service accounts
$SqlServiceAccountName = "svc-sqlserver"
$SqlAgentAccountName = "svc-sqlagent"

# Create SQL Server service accounts
New-ADUser -Name $SqlServiceAccountName -UserPrincipalName "$SqlServiceAccountName@$DomainName" `
    -AccountPassword $SafeModePassword -Enabled $true -PasswordLastSet $false

New-ADUser -Name $SqlAgentAccountName -UserPrincipalName "$SqlAgentAccountName@$DomainName" `
    -AccountPassword $SafeModePassword -Enabled $true -PasswordLastSet $false

# Add service accounts to appropriate groups
Add-ADGroupMember -Identity "Domain Admins" -Members $SqlServiceAccountName
Add-ADGroupMember -Identity "Backup Operators" -Members $SqlAgentAccountName
```

**Group Policy Configuration for SQL Server:**
```powershell
# Configure Group Policy for SQL Server security
$PolicyPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
New-ItemProperty -Path $PolicyPath -Name "EnableScriptBlockLogging" -Value 1 -PropertyType DWord -Force

# Disable PowerShell v2 for security
Disable-WindowsOptionalFeature -Online -FeatureName "MicrosoftWindowsPowerShellv2"

# Configure audit policies
auditpol /set /category:"Logon/Logoff" /success:enable /failure:enable
auditpol /set /category:"Object Access" /success:enable /failure:enable
auditpol /set /category:"Privilege Use" /success:enable /failure:enable
auditpol /set /category:"Policy Change" /success:enable /failure:enable
auditpol /set /category:"System Events" /success:enable /failure:enable

# Configure account lockout policy
net accounts /minpwdlen:12 /maxpwage:90 /minpwage:0 /uniquepw:24 /lockoutduration:30 /lockoutthreshold:5 /lockoutwindow:30

# Configure password policy for service accounts
$GPOPath = "HKLM:\SYSTEM\CurrentControlSet\Services\NetLogon\Parameters"
New-ItemProperty -Path $GPOPath -Name "RequireStrongNP2" -Value 1 -PropertyType DWord -Force
```

### Step 2: Certificate Management

**SQL Server Certificate Setup:**
```powershell
# Create and install SSL certificate for SQL Server
function New-SqlServerCertificate {
    param(
        [string]$CertificateName,
        [string]$ServerName,
        [string]$Subject = "CN=$ServerName",
        [string]$KeySize = "4096",
        [int]$ValidDays = 365
    )
    
    # Create certificate request
    $CertRequest = @"
[Version]
Signature = "$Windows NT"

[Strings]
szOID_2_5_29_17 = 2.5.29.17   # Subject Alternative Name
szOID_2_5_29_15 = 2.5.29.15   # Key Usage

[RequestAttributes]
SAN="DNS=$ServerName&DNS=$($ServerName).$((Get-ADDomain).DNSRoot)"
KeyUsage=CRITICAL,DigitalSignature
"@
    
    # Save certificate request
    $RequestFile = "C:\Temp\$CertificateName.csr"
    $CertRequest | Out-File -FilePath $RequestFile -Encoding ASCII
    
    # Generate self-signed certificate for testing
    $Cert = New-SelfSignedCertificate -DnsName $ServerName, "$ServerName.$((Get-ADDomain).DNSRoot)" `
        -CertStoreLocation "Cert:\LocalMachine\My" `
        -KeyExportPolicy Exportable `
        -KeySpec Signature `
        -KeyLength $KeySize `
        -KeyAlgorithm RSA `
        -HashAlgorithm SHA256
    
    # Export certificate
    $CertFile = "C:\Temp\$CertificateName.cer"
    $PfxFile = "C:\Temp\$CertificateName.pfx"
    
    Export-Certificate -Cert $Cert -FilePath $CertFile
    Export-PfxCertificate -Cert $Cert -FilePath $PfxFile -Password (ConvertTo-SecureString "Password123!" -AsPlainText -Force)
    
    # Install certificate to SQL Server certificate store
    $Store = New-Object System.Security.Cryptography.X509Certificates.X509Store("My", "LocalMachine")
    $Store.Open("ReadWrite")
    $Store.Add($Cert)
    $Store.Close()
    
    Write-Host "Certificate created: $CertFile"
    Write-Host "Certificate installed successfully"
    
    return $Cert
}

# Generate certificate for SQL Server
$ServerName = (Get-ADComputer -Filter {Name -like "*sql*"}).Name | Select-Object -First 1
$Cert = New-SqlServerCertificate -CertificateName "SQL-Server-Cert" -ServerName $ServerName
```

### Step 3: Security Monitoring

**Security Event Configuration:**
```powershell
# Configure Windows Event Logging for SQL Server
function Set-SqlServerSecurityLogging {
    # Enable advanced audit policies
    auditpol /set /category:"Account Logon" /success:enable /failure:enable
    auditpol /set /category:"Account Management" /success:enable /failure:enable
    auditpol /set /category:"Detailed Tracking" /success:enable /failure:enable
    auditpol /set /category:"DS Access" /success:enable /failure:enable
    auditpol /set /category:"DS Change" /success:enable /failure:enable
    
    # Configure SQL Server specific security logging
    $LogPath = "HKLM:\SYSTEM\CurrentControlSet\Control\小天信"
    
    # Configure Event Log retention
    wevtutil sl security /ms:104857600  # 100MB
    wevtutil sl security /rt:true       # Real-time
    
    # Configure SQL Server Error Log
    $SQLLogPath = "HKLM:\SOFTWARE\Microsoft\MSSQLServer\MSSQLServer"
    New-ItemProperty -Path $SQLLogPath -Name "AuditLevel" -Value 3 -PropertyType DWord -Force  # Success and Failure
    New-ItemProperty -Path $SQLLogPath -Name "LoginTimeout" -Value 30 -PropertyType DWord -Force
    New-ItemProperty -Path $SQLLogPath -Name "RemoteDacEnabled" -Value 0 -PropertyType DWord -Force
    
    # Enable C2 audit tracing (enhanced security)
    EXEC sp_configure 'c2 audit mode', 1
    RECONFIGURE
}

# Configure intrusion detection
function Set-IntrusionDetection {
    # Install and configure Windows Defender Advanced Threat Protection
    Install-Module -Name AzureADPreview -Force
    
    # Configure Windows Defender
    Set-MpPreference -DisableRealtimeMonitoring $false
    Set-MpPreference -DisableBehaviorMonitoring $false
    Set-MpPreference -DisableBlockAtFirstSeen $true
    Set-MpPreference -DisableIOAVProtection $false
    Set-MpPreference -DisableScriptScanning $false
    Set-MpPreference -CheckForSignaturesBeforeRunningScan $true
    
    # Configure network protection
    Set-MpPreference -EnableNetworkProtection Enabled
    Set-MpPreference -PUAProtection Enabled
}

# Configure database security
function Set-DatabaseSecurity {
    param([string]$DatabaseName = "master")
    
    # Enable TDE (Transparent Data Encryption)
    $TdeCert = "TDE_Certificate_$DatabaseName"
    
    # Create certificate for TDE
    USE master;
    CREATE CERTIFICATE $TdeCert WITH SUBJECT = 'TDE Certificate for $DatabaseName';
    
    # Enable TDE
    USE $DatabaseName;
    CREATE DATABASE ENCRYPTION KEY WITH ALGORITHM AES_256 ENCRYPTION BY SERVER CERTIFICATE $TdeCert;
    ALTER DATABASE $DatabaseName SET ENCRYPTION ON;
    
    # Configure database security options
    ALTER DATABASE $DatabaseName SET RECOVERY FULL;
    ALTER DATABASE $DatabaseName SET PAGE_VERIFY CHECKSUM;
    ALTER DATABASE $DatabaseName SET AUTO_CREATE_STATISTICS ON;
    ALTER DATABASE $DatabaseName SET AUTO_UPDATE_STATISTICS ON;
    ALTER DATABASE $DatabaseName SET AUTO_UPDATE_STATISTICS_ASYNC ON;
}

# Execute security configuration
Set-SqlServerSecurityLogging
Set-IntrusionDetection
```

This comprehensive environment preparation guide covers virtual machine setup, operating system optimization, network configuration, and security framework implementation. Each section provides detailed scripts and configurations that can be adapted to your specific infrastructure requirements.

Remember to test all configurations in a development environment before applying to production systems, and ensure proper documentation of all security settings and procedures for compliance requirements.