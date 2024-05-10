---
title: "AutomatedLab Networking With External Switch"
date: 2024-05-10T16:17:00-00:00
draft: false
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

The one thing you notice with Hyper-V 2019 is that you are running Core not Desktop. Given that, running with an internal switch will prove difficult. I started with the AutomatedLab documentation and sample scripts 
to see how to accomplish this. I also looked for additional information and found a number of sources that I will list at the end. Between those sources and the documentation I 
was able to put together a solution that met my needs. 

This example is from my home lab. There is no splatting or backticks. I will splat the code later and test it and update the blog/gist. 
The full source code can be found here. [PSHLab](https://gist.github.com/glsc251/6269ce9c263a0409bf4e9c3209acb3de)

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
Add-LabMachineDefinition -Name dc1 -Memory 4GB -OperatingSystem 'Windows Server 2022 Standard Evaluation' -Network 'vSwitch' -Roles RootDC, CARoot -DomainName Company.pri -IpAddress 10.10.10.16 -gateway 10.10.10.1 -VmGeneration 2

# S3 is a Winddows Server 2022 with Desktop Experience that will have all of the domain administration tools on it.
Add-LabMachineDefinition -Name s3 -Memory 8GB -OperatingSystem 'Windows Server 2022 Standard Evaluation (Desktop Experience)' -DomainName Company.pri -IpAddress 10.10.10.15 -gateway 10.10.10.1 -DNSServer1 10.10.10.16 -IsDomainJoined:$true -VmGeneration 2

# I don't want the stock SQL role that AutomatedLab has as everything is installed on C:.
# Defining disks for the SQL server.
Add-LabDiskDefinition -Name SQL_AppDrive -DiskSizeInGb 60 -Label Data -DriveLetter E -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_DataDrive -DiskSizeInGb 60 -Label Data -DriveLetter D -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_LogDrive -DiskSizeInGb 20 -Label Data -DriveLetter L -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_TempDBDrive -DiskSizeInGb 20 -Label TempDB -DriveLetter T -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_BackupDrive -DiskSizeInGb 50 -Label Backup -DriveLetter B -AllocationUnitSize 64kb

# Again, Winddows Server 2022 with Desktop Experience with SQL to be installed. 
Add-LabMachineDefinition -Name SQL1 -Memory 8GB -OperatingSystem 'Windows Server 2022 Standard Evaluation (Desktop Experience)' -DomainName Company.pri -IpAddress 10.10.10.12 -gateway 10.10.10.1 -DNSServer1 10.10.10.16 -IsDomainJoined:$true -Disk SQL_AppDrive,SQL_DataDrive,SQL_LogDrive,SQL_TempDBDrive,SQL_BackupDrive -VmGeneration 2

# Spin it all up
Install-Lab -Verbose

# For brevity I left out some of the code. You can see it at the link mentioned in the article.

Show-LabDeploymentSummary -Detailed

```

I also put together a script to build a SQL lab at work with a DC, a workstation, and 4 SQL servers (two 2019 and two 2022). I left the SQL install up to the DBAs as this  
is a lab for them to work through Learn PowerShell in a month of lunches and Learn DBATools in a month of lunches.  
The networking here is also on an external switch but also includes VLAN ids. The full source code is here [SQLLab](https://gist.github.com/glsc251/bf8f467f0f200cd3589c4ee675af5714)

```PowerShell
# SQLLab.ps1
$labName = 'SQLLab'

New-LabDefinition -Name $LabName -DefaultVirtualizationEngine HyperV -VmPath C:\ClusterStor\Vol2\Labs
Add-LabVirtualNetworkDefinition -Name 'vSwitch' -HyperVProperties @{ SwitchType = 'External'; AdapterName = 'Ethernet' } -AddressSpace 10.10.8.0/24

$nic_LabDC1 = @()
$nic_LabDC1 += New-LabNetworkAdapterDefinition -VirtualSwitch vSwitch -Ipv4Address 10.10.11.128/24 -Ipv4gateway 10.10.11.254 -AccessVLANID 64
$nic_LabS3 = @()
$nic_LabS3 += New-LabNetworkAdapterDefinition -VirtualSwitch vSwitch -Ipv4Address 10.10.11.129/24 -Ipv4gateway 10.10.11.254 -AccessVLANID 64 -Ipv4DNSServers 10.10.11.128
$nic_LabSQL1 = @()
$nic_LabSQL1 += New-LabNetworkAdapterDefinition -VirtualSwitch vSwitch -Ipv4Address 10.10.11.130/24 -Ipv4gateway 10.10.11.254 -AccessVLANID 64 -Ipv4DNSServers 10.10.11.128

Add-LabIsoImageDefinition -Name SQLServer2019 -Path $labSources\ISOs\SQLServer2019-x64-ENU.iso
Add-LabIsoImageDefinition -Name SQLServer2022 -Path $labSources\ISOs\SQLServer2022-x64-ENU.iso

Add-LabMachineDefinition -Name labdc1 -Memory 4GB -OperatingSystem 'Windows Server 2022 Standard Evaluation' -Roles RootDC, CARoot -DomainName Company.pri -NetworkAdapter $nic_LabDC1  -VmGeneration 2
Add-LabMachineDefinition -Name labs3 -Memory 16GB -OperatingSystem 'Windows Server 2022 Standard Evaluation (Desktop Experience)' -DomainName Company.pri -NetworkAdapter $nic_LabS3 -IsDomainJoined:$true -VmGeneration 2

Add-LabDiskDefinition -Name SQL_AppDrive -DiskSizeInGb 40 -Label App -DriveLetter E -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_DataDrive -DiskSizeInGb 50 -Label Data -DriveLetter D -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_LogDrive -DiskSizeInGb 20 -Label Logs -DriveLetter L -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_TempDBDrive -DiskSizeInGb 20 -Label TempDB -DriveLetter T -AllocationUnitSize 64kb
Add-LabDiskDefinition -Name SQL_BackupDrive -DiskSizeInGb 50 -Label Backup -DriveLetter B -AllocationUnitSize 64kb

Add-LabMachineDefinition -Name labSQL1 -Memory 16GB -OperatingSystem 'Windows Server 2022 Standard Evaluation (Desktop Experience)' -DomainName Company.pri -NetworkAdapter $nic_LabSQL1 -IsDomainJoined:$true -Disk SQL_AppDrive,SQL_DataDrive,SQL_LogDrive,SQL_TempDBDrive,SQL_BackupDrive -VmGeneration 2


Install-Lab -Verbose

Show-LabDeploymentSummary -Detailed

```

In closing, it is a very useful tool to be able to rapidly deploy a lab and redeploy it over and over. No licensing was hurt as I used the evaluation software for all components.  
You should note that deploying this network configuration at a work environment will likely get you a visit from your security team. I took the steps to change all the passwords to more secure ones. I can update the XML files to remove the lab when the time comes.


### Links

General issues and networking  
[HyperV Network Definitions unable to use existing virtual swith connected to Wi-Fi adapter](https://github.com/AutomatedLab/AutomatedLab/issues/387)  
[Docmentation on New-LabNetworkAdapter Definition](https://automatedlab.org/en/latest/AutomatedLabDefinition/en-us/New-LabNetworkAdapterDefinition/)  
[Defining your network](https://automatedlab.org/en/latest/Wiki/Basic/networksandaddresses/#Simple%20Network%20Definition)  
[Invoke-LabCommand Documentation with good example showing splatting](https://automatedlab.org/en/latest/AutomatedLabCore/en-us/Invoke-LabCommand/)  
[Raandree](https://github.com/raandree) example of network adapter definition in this [Issue 606](https://github.com/AutomatedLab/AutomatedLab/issues/606) was helpful.  
  
SQL  
Adrian Hall [Adding SQL Services to an AutomatedLab Configuration](https://shellmonger.wordpress.com/2014/12/31/adding-sql-services-to-an-automatedlab-configuration/)  
[Installing SQL server from the command line](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/install-sql-server-from-the-command-prompt?view=sql-server-ver16&redirectedfrom=MSDN)  
[This DSC example helped me understand some of the SQL server installer command line switches](https://github.com/dsccommunity/SqlServerDsc/issues/1254)  
[I had to turn off SQL Updates during the install due to a this error code found in this link. /UpdateEnabled='False' ](https://learn.microsoft.com/en-us/answers/questions/1501232/sqlserver2019-or-server2022-fails-with-error-proce)  

