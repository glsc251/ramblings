---
title: "AutomatedLab Networking With External Switch"
date: 2024-05-04T14:57:39-04:00
draft: true
toc: false
images:
tags: 
  - AutomatedLab
  - Hyper-V
  - PowerShell
---

A few weeks ago (March 20th) at the [@RTPSUG](https://twitter.com/RTPSug) meeting [Phil Bossman](https://twitter.com/Schlauge) presented on using AutomatedLab. I had wanted to give it a try in my home lab and 
until recently one of my lab machines was running a bare metal hypervisor that will remain nameless. It was time to change nameless. I loaded Proxmox and even tried XCP-NG. 
After watching Phils presentation I decided to load Hyper-V 2019 and give AutomatedLab a spin. A lot of the decision to change from Proxmox/XCP-NG had to do with automation. 

The one thing you notice with Hyper-V 2019 is that you are running Core not Desktop. Given that, running with an internal switch will prove difficult. I started with the AutomatedLab docuemtation and sample scripts 
to see how to accomplish this. For me, the examples were not totally helpful. I tried searching for the information I needed and found a number of sources that I will list at the end. Between those and the documentation I 
was able to put together a solution that met my needs. 

The first exapmle is in my home lab. 

```PowerShell
# pshlab.ps1

# Lab name can be anything
$labName = 'PowerHouse'

# I am specifying a particular path for the VMs given my hardware setup.
New-LabDefinition -Name $LabName -DefaultVirtualizationEngine HyperV -VmPath D:\automatedlab-vms

# Going to have SQL in this lab
Add-LabIsoImageDefinition -Name SQLServer2022 -Path $labSources\ISOs\SQLServer2022-x64-ENU.iso

# Adding the network definition with the name of the existing virtual switch and its connected adapter and the CIDR.
Add-LabVirtualNetworkDefinition -Name 'vSwitch' -HyperVProperties @{ SwitchType = 'External'; AdapterName = 'Ethernet' } -AddressSpace 10.10.10.0/24

# The single domain controller running Windows Server 2022 Core. Note this also has the Certificate Authority role for the domain.
$addLabMachine_DC1 = @{
    Name = 'dc1'
    Memory = 4GB
    OperatingSystem = 'Windows Server 2022 Standard Evaluation'
    Network = 'vSwitch'
    Roles = 'RootDC', 'CARoot'
    DomainName = 'Company.pri'
    IpAddress = '10.10.10.16'
    gateway = '10.10.10.1'
    VmGeneration = 2
}

Add-LabMachineDefinition @addLabMachine_DC1


# S3 is a Winddows Server 2022 with Desktop Experience that will have all of the domain administration tools on it.
$addLabMachine_S3 = @{
    Name = 's3'
    Memory = 8GB
    OperatingSystem = 'Windows Server 2022 Standard Evaluation (Desktop Experience)'
    DomainName = 'Company.pri'
    IpAddress = '10.10.10.15'
    gateway = '10.10.10.1'
    DNSServer1 = '10.10.10.16'
    IsDomainJoined = $true
    VmGeneration = 2
}

Add-LabMachineDefinition @addLabMachine_S3


# I don't want the stock SQL role that AutomatedLab has as everything is installed on C:.
# Defining disks for the SQL server.
Add-LabDiskDefinition -Name SQL_AppDrive -DiskSizeInGb 60 -Label Data -DriveLetter E -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_DataDrive -DiskSizeInGb 60 -Label Data -DriveLetter D -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_LogDrive -DiskSizeInGb 20 -Label Data -DriveLetter L -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_TempDBDrive -DiskSizeInGb 20 -Label TempDB -DriveLetter T -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_BackupDrive -DiskSizeInGb 50 -Label Backup -DriveLetter B -AllocationUnitSize 64kb

# Again, Winddows Server 2022 with Desktop Experience with SQL to be installed. 
$addLabMachine_SQL1 = @{
    Name = 'SQL1'
    Memory = 8GB
    OperatingSystem = 'Windows Server 2022 Standard Evaluation (Desktop Experience)'
    DomainName = 'Company.pri'
    IpAddress = '10.10.10.12'
    gateway = '10.10.10.1'
    DNSServer1 = '10.10.10.16'
    IsDomainJoined = $true
    Disk = 'SQL_AppDrive', 'SQL_DataDrive', 'SQL_LogDrive', 'SQL_TempDBDrive', 'SQL_BackupDrive'
    VmGeneration = 2
}

Add-LabMachineDefinition @addLabMachine_SQL1

# Spin it all up
Install-Lab -Verbose

#This is the default I took from one of the examples. You still have to enable PKI in the Domain Controller and Default Domain GPOs. At this time I have not automated that if it is even possible.
Enable-LabCertificateAutoenrollment -Computer -User -CodeSigning

# Installing the latest PowerShell on the servers.
Install-LabSoftwarePackage -Path $labSources\SoftwarePackages\PowerShell-7.4.2-win-x64.msi -ComputerName dc1, s3, SQL1 -CommandLine '/quiet ADD_EXPLORER_CONTEXT_MENU_OPENPOWERSHELL=1 ADD_FILE_CONTEXT_MENU_RUNPOWERSHELL=1 ENABLE_PSREMOTING=1 REGISTER_MANIFEST=1 USE_MU=1 ENABLE_MU=1 ADD_PATH=1'

# S3 needs some features installed.
Invoke-LabCommand -Activity 'Install RSAT AD Tools' -ComputerName s3 -ScriptBlock { Add-WindowsFeature RSAT-AD-Tools -IncludeAllSubFeature }
Invoke-LabCommand -Activity 'Install Certificate Management' -ComputerName s3 -ScriptBlock { Add-WindowsFeature RSAT-ADCS-Mgmt }
Invoke-LabCommand -Activity 'Install Group Policy Management' -ComputerName s3 -ScriptBlock { Add-WindowsFeature GPMC }

# Create temp folder on DC1
$invokeLabCommand_Temp = @{
    Activity = 'Create Temp folder if not found'
    ComputerName = (get-labvm | Where Name -match "dc*" | select -ExpandProperty Name)
    ScriptBlock = { If (-not (Test-Path -Path C:\Temp)) {New-Item -Path C:\ -Name Temp -ItemType Directory} }
}

Invoke-LabCommand @invokeLabCommand_Temp

# This is just me being me. I don't like passwords in scripts. I need to add the code to remove these files from the temp folder.
Copy-LabFileItem -Path $Labsources\SoftwarePackages\encryptdata.zip  -ComputerName (get-labvm | Where Name -match "dc*" | select -ExpandProperty Name) -DestinationFolderPath C:\temp
Copy-LabFileItem -Path $Labsources\SoftwarePackages\encpw.txt  -ComputerName (get-labvm | Where Name -match "dc*" | select -ExpandProperty Name) -DestinationFolderPath C:\temp
Copy-LabFileItem -Path $Labsources\SoftwarePackages\privatekey.txt  -ComputerName (get-labvm | Where Name -match "dc*" | select -ExpandProperty Name) -DestinationFolderPath C:\temp

# I followed part of the code from Adrian Hall for two sections of code below. See the links below for his article.
Invoke-LabCommand -ActivityName 'Create groups and users' -ComputerName (get-labvm | Where Name -match "dc*" | select -ExpandProperty Name) -ScriptBlock {
    If (Test-Path -Path C:\temp\encryptdata.zip) { Expand-Archive -Path C:\temp\encryptdata.zip -DestinationPath "C:\Program Files\WindowsPowerShell\Modules" -Force }
    If (Test-Path -Path "C:\Program Files\WindowsPowerShell\Modules\EncryptData") {
        $SecurePassword = (ConvertTo-SecureString -String (UnProtect-Data -Data (Get-Content C:\temp\encpw.txt) -PrivateKey (Get-Content C:\Temp\privatekey.txt)) -AsPlainText -Force)
        $sqlsvc = New-ADUser -Name svc_sqlserver -AccountPassword $SecurePassword -Path "CN=Users,DC=Company,DC=Pri" -Enabled $true -PassThru
        New-ADGroup -Name 'SQLAdmins' -SamAccountName SQLAdmins -Path "CN=Users,DC=Company,DC=Pri" -GroupScope Domain -GroupCategory Security -DisplayName 'SQL Admins'
        Get-ADGroup -Identity 'SQLAdmins' | Add-ADGroupMember -Members @($sqlsvc,'domain admins') -PassThru
    }
}

$SQLMachines = Get-LabVM | Where Name -like "SQL*"
foreach ($Machine in $SQLMachines)
{
    # Install .NET Framework 3.5
    Install-LabWindowsFeature -ComputerName $Machine.Name -FeatureName Net-Framework-Core
 
    # Mount the DVD Drive with the ISO 
    Add-VMDvdDrive -VMName $Machine.Name -Path (Get-LabIsoImageDefinition | Where Name -Match "SQLServer2022" | Select -ExpandProperty Path)

    $invokeLabCommand_Temp = @{
        Activity = 'Create Temp folder if not found'
        ComputerName = (get-labvm | Where Name -match "dc*" | select -ExpandProperty Name)
        ScriptBlock = { If (-not (Test-Path -Path C:\Temp)) {New-Item -Path C:\ -Name Temp -ItemType Directory} }
    }

    Invoke-LabCommand @invokeLabCommand_Temp
    
    Copy-LabFileItem -Path $Labsources\SoftwarePackages\encryptdata.zip  -ComputerName $Machine.Name -DestinationFolderPath C:\temp
    Copy-LabFileItem -Path $Labsources\SoftwarePackages\encpw.txt  -ComputerName $Machine.Name -DestinationFolderPath C:\temp
    Copy-LabFileItem -Path $Labsources\SoftwarePackages\privatekey.txt  -ComputerName $Machine.Name -DestinationFolderPath C:\temp
    Copy-LabFileItem -Path D:\LabSources\SoftwarePackages\SSMS-Setup-ENU.exe -DestinationFolderPath C:\Temp -ComputerName $Machine.Name

       # Install SQL Server
       Invoke-LabCommand -ActivityName 'InstallSQLServer' -ComputerName $Machine.Name -ScriptBlock {
        If (Test-Path -Path C:\temp\encryptdata.zip) { Expand-Archive -Path C:\temp\encryptdata.zip -DestinationPath "C:\Program Files\WindowsPowerShell\Modules" -Force }
        New-Item -ItemType Directory -Path 'E:\Program Files\Microsoft SQL Server\MSSQL'
        New-Item -ItemType Directory -Path 'D:\Program Files\Microsoft SQL Server\'
        New-Item -ItemType Directory -Path 'L:\Program Files\Microsoft SQL Server\MSSQL\Data'
        New-Item -ItemType Directory -Path 'B:\Program Files\Microsoft SQL Server\MSSQL\Backup'
        New-Item -ItemType Directory -Path 'T:\Program Files\Microsoft SQL Server\MSSQL\Data'
        New-Item -ItemType Directory -Path 'E:\Program Files (x86)\Microsoft SQL Server\'

       If (Test-Path -Path "C:\Program Files\WindowsPowerShell\Modules\EncryptData") {
        Set-Location -Path (Get-WmiObject -Class Win32_CDRomDrive).Drive
        .\Setup.exe /Q /Action=Install /ENU /IAcceptSQLServerLicenseTerms /InstanceName='MSSQLSERVER' /ErrorReporting=0 /InstallSQLDataDir='D:\Program Files\MSSQL\Data' /SQLUserDBDir='D:\Program Files\MSSQL\Data' SQLUserDBLogDir='L:\Program Files\MSSQL\Log' /SQLTempDBDir='T:\Program Files\MSSQL\Data\TempDB' /SQLTempDBLogDir='T:\Program Files\MSSQL\Log\TempDB' /SQLBackupDir='B:\Program Files\MSSQL\Backup' /Features='SQLENGINE,Tools' /InstanceID='MSSQLSERVER' /UpdateEnabled='False' /InstallSharedDir='E:\Program Files\Microsoft SQL Server' /InstallSharedWOWDir='E:\Program Files (x86)\Microsoft SQL Server' /InstanceDir='E:\Program Files\Microsoft SQL Server' /SQLCollation='SQL_Latin1_General_CP1_CI_AS' /AgtSvcStartupType=Disabled /BrowserSvcStartupType=Disabled /SQLSvcAccount=Company\svc_sqlserver /SQLSvcPassword=(UnProtect-Data -Data (Get-Content C:\temp\encpw.txt) -PrivateKey (Get-Content C:\Temp\privatekey.txt)) /SQLSvcStartupType=Automatic /SQLSysAdminAccounts=Company\SQLAdmins /SQMReporting=0 /FilestreamLevel=0 /TCPEnabled=1
        C:\temp\SSMS-Setup-ENU.exe /install /quiet /norestart smsinstallroot='E:\Program Files (x86)\Microsoft SQL Server\Server Management Studio 20'
        Set-Location -Path 'C:\Temp'
        }
    }

}

Show-LabDeploymentSummary -Detailed

```
### Links
Adrian Hall [Adding SQL Services to an AutomatedLab Configuration](https://shellmonger.wordpress.com/2014/12/31/adding-sql-services-to-an-automatedlab-configuration/)

