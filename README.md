# ADDS
Active Directory lab for `contoso.lab'

If you want an **Active Directory lab for `contoso.lab`** and automated domain join scripts for **SQL Server 2022** and **Hyper-V Server/Windows Server 2022 hosts**, here's a complete example.

## Lab Design

| Server | Name               | IP        | Role              |
| ------ | ------------------ | --------- | ----------------- |
| DC01   | dc01.contoso.lab   | 10.0.0.10 | Domain Controller |
| SQL01  | sql01.contoso.lab  | 10.0.0.20 | SQL Server 2022   |
| HV01   | hv01.contoso.lab   | 10.0.0.30 | Hyper-V Host      |
| MGMT01 | mgmt01.contoso.lab | 10.0.0.40 | Management Server |

Domain:

```text
contoso.lab
```

Administrator:

```text
CONTOSO\Administrator
```

---

# 1. Create Active Directory Forest

Run on DC01:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Import-Module ADDSDeployment

Install-ADDSForest `
-DomainName "contoso.lab" `
-DomainNetbiosName "CONTOSO" `
-InstallDNS `
-Force
```

---

# 2. Create OU Structure

```powershell
Import-Module ActiveDirectory

New-ADOrganizationalUnit -Name "Servers" -Path "DC=contoso,DC=lab"

New-ADOrganizationalUnit -Name "SQL" `
-Path "OU=Servers,DC=contoso,DC=lab"

New-ADOrganizationalUnit -Name "HyperV" `
-Path "OU=Servers,DC=contoso,DC=lab"

New-ADOrganizationalUnit -Name "ServiceAccounts" `
-Path "DC=contoso,DC=lab"
```

---

# 3. Create Service Accounts

```powershell
New-ADUser `
-Name "svc_sql" `
-SamAccountName "svc_sql" `
-AccountPassword (ConvertTo-SecureString "Passw0rd01!" -AsPlainText -Force) `
-Enabled $true

New-ADUser `
-Name "svc_hyperv" `
-SamAccountName "svc_hyperv" `
-AccountPassword (ConvertTo-SecureString "Passw0rd01!" -AsPlainText -Force) `
-Enabled $true
```

---

# 4. Domain Join Script for SQL01

Save as:

```text
Join-SQL01.ps1
```

```powershell
$Domain="contoso.lab"

$User="CONTOSO\Administrator"

$Password=ConvertTo-SecureString "Passw0rd01!" -AsPlainText -Force

$Credential=New-Object System.Management.Automation.PSCredential($User,$Password)

Rename-Computer -NewName "SQL01" -Force

Add-Computer `
-DomainName $Domain `
-Credential $Credential `
-Restart `
-Force
```

---

# 5. Domain Join Script for HV01

Save as:

```text
Join-HV01.ps1
```

```powershell
$Domain="contoso.lab"

$User="CONTOSO\Administrator"

$Password=ConvertTo-SecureString "Passw0rd01!" -AsPlainText -Force

$Credential=New-Object System.Management.Automation.PSCredential($User,$Password)

Rename-Computer -NewName "HV01" -Force

Add-Computer `
-DomainName $Domain `
-Credential $Credential `
-Restart `
-Force
```

---

# 6. Install Hyper-V Role

Run on HV01:

```powershell
Install-WindowsFeature `
-Hyper-V `
-IncludeManagementTools `
-Restart
```

Create virtual switch:

```powershell
New-VMSwitch `
-Name "ExternalSwitch" `
-NetAdapterName "Ethernet" `
-AllowManagementOS $true
```

---

# 7. Install SQL Server 2022

Silent installation:

```powershell
setup.exe /Q `
/ACTION=Install `
/FEATURES=SQLENGINE `
/INSTANCENAME=MSSQLSERVER `
/SQLSVCACCOUNT="CONTOSO\svc_sql" `
/SQLSVCPASSWORD="Passw0rd01!" `
/AGTSVCACCOUNT="CONTOSO\svc_sql" `
/AGTSVCPASSWORD="Passw0rd01!" `
/SQLSYSADMINACCOUNTS="CONTOSO\Domain Admins" `
/IACCEPTSQLSERVERLICENSETERMS
```

---

# 8. Create Hyper-V Cluster (Optional)

```powershell
Install-WindowsFeature Failover-Clustering -IncludeManagementTools
```

Validate:

```powershell
Test-Cluster `
-Node HV01,HV02
```

Create:

```powershell
New-Cluster `
-Name HVC01 `
-Node HV01,HV02 `
-StaticAddress 10.0.0.50
```

---

# 9. One-Shot Automated Build Script

```powershell
# Configure DNS

Set-DnsClientServerAddress `
-InterfaceAlias Ethernet `
-ServerAddresses 10.0.0.10

# Join Domain

.\Join-SQL01.ps1

# Install SQL

.\Install-SQL2022.ps1
```

For a production-style lab, I would also add:

* Group Policy baseline
* PKI / Certificate Services
* Windows Admin Center
* Microsoft Entra Connect
* SQL Always On Availability Groups
* Hyper-V Cluster with Shared Storage
* Microsoft Defender for Servers
* Azure Arc onboarding
* Monitoring with Prometheus + Grafana
* Microsoft Sentinel integration

This would give you a complete enterprise `contoso.lab` environment similar to what you would find in a real infrastructure architecture lab.
