\# Enterprise Help Desk Lab



\## Overview



This project simulates a small enterprise Windows help desk environment designed to practice common Tier 1 and Tier 2 support tasks.



The lab was built on top of my existing Proxmox infrastructure and includes a Windows Server domain controller, domain-joined Windows 11 client systems, Active Directory, DNS, Group Policy, shared folders, permissions, remote administration, and realistic troubleshooting scenarios.



The purpose of the project was to gain practical experience with the types of tasks performed by Help Desk Technicians, Desktop Support Technicians, IT Support Specialists, and Junior Systems Administrators.



\---



\# Lab Goals



The lab was designed to practice:



\* Windows Server administration

\* Active Directory Domain Services

\* User and group management

\* Organizational Units

\* Windows domain joins

\* Password resets

\* Account unlocks

\* Group Policy

\* NTFS permissions

\* Shared-folder permissions

\* DNS troubleshooting

\* Windows network troubleshooting

\* Printer troubleshooting

\* PowerShell administration

\* Remote Desktop support

\* Ticket documentation

\* Incident prioritization

\* Escalation procedures



\---



\# Environment



\## Virtualization



The environment runs inside my existing Proxmox infrastructure.



Example systems:



| Hostname | Operating System | Purpose                |

| -------- | ---------------- | ---------------------- |

| DC01     | Windows Server   | Active Directory / DNS |

| CLIENT01 | Windows 11       | Domain workstation     |

| CLIENT02 | Windows 11       | Domain workstation     |



Example domain:



```text

corp.lab

```



Example IP plan:



```text

DC01       192.168.1.210

CLIENT01   DHCP

CLIENT02   DHCP

Gateway    192.168.1.1

DNS        192.168.1.210

```



\---



\# 1. Windows Server Deployment



A Windows Server virtual machine was created in Proxmox and configured as the domain controller.



After installing Windows Server, I renamed the system to:



```text

DC01

```



\## Rename Server



PowerShell:



```powershell

Rename-Computer -NewName "DC01" -Restart

```



After the reboot, I verified the hostname:



```powershell

hostname

```



Expected output:



```text

DC01

```



\---



\# 2. Configure Static IP Address



A domain controller should use a static IP address so client systems can consistently locate DNS and Active Directory services.



First I identified the network adapter:



```powershell

Get-NetAdapter

```



Then reviewed the current configuration:



```powershell

Get-NetIPConfiguration

```



Example static configuration:



```powershell

New-NetIPAddress `

&#x20; -InterfaceAlias "Ethernet" `

&#x20; -IPAddress 192.168.1.210 `

&#x20; -PrefixLength 24 `

&#x20; -DefaultGateway 192.168.1.1

```



Configure DNS:



```powershell

Set-DnsClientServerAddress `

&#x20; -InterfaceAlias "Ethernet" `

&#x20; -ServerAddresses 192.168.1.210

```



Verify:



```powershell

ipconfig /all

```



Connectivity tests:



```powershell

ping 192.168.1.1

ping 8.8.8.8

```



\---



\# 3. Install Active Directory Domain Services



The Active Directory Domain Services role was installed using PowerShell.



Check available roles:



```powershell

Get-WindowsFeature

```



Install Active Directory Domain Services:



```powershell

Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

```



Verify:



```powershell

Get-WindowsFeature AD-Domain-Services

```



\---



\# 4. Create the Active Directory Forest



A new Active Directory forest was created for:



```text

corp.lab

```



PowerShell:



```powershell

Install-ADDSForest `

&#x20; -DomainName "corp.lab" `

&#x20; -DomainNetbiosName "CORP" `

&#x20; -InstallDNS

```



The server prompted for a Directory Services Restore Mode password.



After configuration, the server restarted automatically.



\---



\# 5. Verify Active Directory



After rebooting, I confirmed that Active Directory was available.



```powershell

Get-ADDomain

```



Verify forest information:



```powershell

Get-ADForest

```



Verify domain controller:



```powershell

Get-ADDomainController

```



DNS was also tested:



```powershell

nslookup corp.lab

```



\---



\# 6. Create Organizational Units



Organizational Units were created to separate users and computers by department.



Example structure:



```text

corp.lab



├── IT

│   ├── Users

│   └── Computers

│

├── Finance

│   ├── Users

│   └── Computers

│

├── HR

│   ├── Users

│   └── Computers

│

└── Operations

&#x20;   ├── Users

&#x20;   └── Computers

```



PowerShell example:



```powershell

New-ADOrganizationalUnit -Name "IT" -Path "DC=corp,DC=lab"



New-ADOrganizationalUnit -Name "Finance" -Path "DC=corp,DC=lab"



New-ADOrganizationalUnit -Name "HR" -Path "DC=corp,DC=lab"



New-ADOrganizationalUnit -Name "Operations" -Path "DC=corp,DC=lab"

```



Create child OUs:



```powershell

New-ADOrganizationalUnit `

&#x20; -Name "Users" `

&#x20; -Path "OU=IT,DC=corp,DC=lab"



New-ADOrganizationalUnit `

&#x20; -Name "Computers" `

&#x20; -Path "OU=IT,DC=corp,DC=lab"

```



The same structure was created for Finance, HR, and Operations.



List OUs:



```powershell

Get-ADOrganizationalUnit -Filter \*

```



\---



\# 7. Create Domain Users



Example users were created for each department.



Example:



```text

Alice Smith      Finance

Bob Johnson      IT

Carol Martinez   HR

David Lee        Operations

```



Create a password variable:



```powershell

$password = ConvertTo-SecureString "TempP@ss123!" -AsPlainText -Force

```



Create Alice:



```powershell

New-ADUser `

&#x20; -Name "Alice Smith" `

&#x20; -GivenName "Alice" `

&#x20; -Surname "Smith" `

&#x20; -SamAccountName "asmith" `

&#x20; -UserPrincipalName "asmith@corp.lab" `

&#x20; -Path "OU=Users,OU=Finance,DC=corp,DC=lab" `

&#x20; -AccountPassword $password `

&#x20; -Enabled $true `

&#x20; -ChangePasswordAtLogon $true

```



Create Bob:



```powershell

New-ADUser `

&#x20; -Name "Bob Johnson" `

&#x20; -GivenName "Bob" `

&#x20; -Surname "Johnson" `

&#x20; -SamAccountName "bjohnson" `

&#x20; -UserPrincipalName "bjohnson@corp.lab" `

&#x20; -Path "OU=Users,OU=IT,DC=corp,DC=lab" `

&#x20; -AccountPassword $password `

&#x20; -Enabled $true `

&#x20; -ChangePasswordAtLogon $true

```



Verify users:



```powershell

Get-ADUser -Filter \*

```



\---



\# 8. Create Security Groups



Security groups were created to control access to resources.



Examples:



```text

Finance-Users

HR-Users

IT-Users

Operations-Users

Shared-Drive-Access

Remote-Desktop-Users

```



Create group:



```powershell

New-ADGroup `

&#x20; -Name "Finance-Users" `

&#x20; -GroupScope Global `

&#x20; -GroupCategory Security `

&#x20; -Path "OU=Finance,DC=corp,DC=lab"

```



Add Alice:



```powershell

Add-ADGroupMember `

&#x20; -Identity "Finance-Users" `

&#x20; -Members "asmith"

```



Verify:



```powershell

Get-ADGroupMember "Finance-Users"

```



\---



\# 9. Configure Windows 11 Clients



Two Windows 11 virtual machines were created in Proxmox.



The clients were named:



```text

CLIENT01

CLIENT02

```



Rename:



```powershell

Rename-Computer -NewName "CLIENT01" -Restart

```



\---



\# 10. Configure Client DNS



Before joining the domain, each Windows client was configured to use the domain controller for DNS.



Check adapter:



```powershell

Get-NetAdapter

```



Set DNS:



```powershell

Set-DnsClientServerAddress `

&#x20; -InterfaceAlias "Ethernet" `

&#x20; -ServerAddresses 192.168.1.210

```



Verify:



```powershell

ipconfig /all

```



Test domain DNS:



```powershell

nslookup corp.lab

```



Test the domain controller:



```powershell

ping 192.168.1.210

```



\---



\# 11. Join Windows Clients to the Domain



The Windows 11 systems were joined to:



```text

corp.lab

```



PowerShell:



```powershell

Add-Computer `

&#x20; -DomainName "corp.lab" `

&#x20; -Credential CORP\\Administrator `

&#x20; -Restart

```



After restart, I verified domain membership:



```powershell

systeminfo | findstr /B /C:"Domain"

```



Another option:



```powershell

(Get-CimInstance Win32\_ComputerSystem).Domain

```



Expected result:



```text

corp.lab

```



\---



\# 12. Move Computer Objects into Organizational Units



The domain-joined machines were moved into the appropriate computer OUs.



Find computers:



```powershell

Get-ADComputer -Filter \*

```



Move CLIENT01:



```powershell

Get-ADComputer "CLIENT01" |

Move-ADObject `

&#x20; -TargetPath "OU=Computers,OU=Finance,DC=corp,DC=lab"

```



Verify:



```powershell

Get-ADComputer "CLIENT01" -Properties DistinguishedName

```



\---



\# 13. Group Policy Configuration



Group Policy was used to simulate centralized workstation management.



Group Policy Management was available after installing Active Directory management tools.



Example policies included:



\* Password policy

\* Account lockout

\* Desktop settings

\* Control Panel restrictions

\* Mapped network drives

\* Security settings



\---



\# 14. Create a Group Policy Object



Create a GPO:



```powershell

New-GPO -Name "Finance Workstation Policy"

```



Link it to Finance:



```powershell

New-GPLink `

&#x20; -Name "Finance Workstation Policy" `

&#x20; -Target "OU=Finance,DC=corp,DC=lab"

```



List GPOs:



```powershell

Get-GPO -All

```



\---



\# 15. Force Group Policy Update



On the Windows client:



```powershell

gpupdate /force

```



Review applied policies:



```powershell

gpresult /r

```



For a more detailed HTML report:



```powershell

gpresult /h C:\\gpresult.html

```



Open:



```powershell

Start-Process C:\\gpresult.html

```



\---



\# 16. Configure Account Lockout Policy



Account lockout policies were configured to simulate enterprise security controls.



Examples:



```text

Account lockout threshold: 5 attempts

Lockout duration: 15 minutes

Reset counter: 15 minutes

```



The policy was configured through:



```text

Group Policy Management

&#x20;   ↓

Default Domain Policy

&#x20;   ↓

Computer Configuration

&#x20;   ↓

Policies

&#x20;   ↓

Windows Settings

&#x20;   ↓

Security Settings

&#x20;   ↓

Account Policies

&#x20;   ↓

Account Lockout Policy

```



Policy verification:



```powershell

net accounts

```



\---



\# 17. Create Shared Folders



Department shares were created on DC01 for lab purposes.



Create directories:



```powershell

New-Item -ItemType Directory -Path "C:\\Shares\\Finance"

New-Item -ItemType Directory -Path "C:\\Shares\\HR"

New-Item -ItemType Directory -Path "C:\\Shares\\IT"

New-Item -ItemType Directory -Path "C:\\Shares\\Shared"

```



Create SMB share:



```powershell

New-SmbShare `

&#x20; -Name "Finance" `

&#x20; -Path "C:\\Shares\\Finance" `

&#x20; -FullAccess "CORP\\Domain Admins" `

&#x20; -ChangeAccess "CORP\\Finance-Users"

```



Verify:



```powershell

Get-SmbShare

```



\---



\# 18. Configure NTFS Permissions



NTFS permissions were assigned based on Active Directory security groups.



Inspect existing permissions:



```powershell

icacls "C:\\Shares\\Finance"

```



Grant Finance users Modify access:



```powershell

icacls "C:\\Shares\\Finance" `

&#x20; /grant "CORP\\Finance-Users:(OI)(CI)M"

```



Verify:



```powershell

icacls "C:\\Shares\\Finance"

```



\---



\# 19. Map a Network Drive



On a Finance workstation:



```powershell

net use F: \\\\DC01\\Finance

```



Verify:



```powershell

net use

```



Remove if needed:



```powershell

net use F: /delete

```



\---



\# 20. Remote Desktop Administration



Remote Desktop was enabled to practice remote administration.



Enable Remote Desktop:



```powershell

Set-ItemProperty `

&#x20; -Path "HKLM:\\System\\CurrentControlSet\\Control\\Terminal Server" `

&#x20; -Name "fDenyTSConnections" `

&#x20; -Value 0

```



Enable firewall rule:



```powershell

Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

```



Verify:



```powershell

Get-NetFirewallRule -DisplayGroup "Remote Desktop"

```



Connect from another Windows machine:



```powershell

mstsc

```



\---



\# 21. Common Active Directory Help Desk Tasks



\## Reset User Password



```powershell

$newPassword = ConvertTo-SecureString "NewTempP@ss123!" -AsPlainText -Force



Set-ADAccountPassword `

&#x20; -Identity "asmith" `

&#x20; -Reset `

&#x20; -NewPassword $newPassword

```



Require password change:



```powershell

Set-ADUser `

&#x20; -Identity "asmith" `

&#x20; -ChangePasswordAtLogon $true

```



\---



\# 22. Unlock User Account



Check locked accounts:



```powershell

Search-ADAccount -LockedOut

```



Unlock:



```powershell

Unlock-ADAccount -Identity "asmith"

```



Verify:



```powershell

Get-ADUser "asmith" -Properties LockedOut

```



\---



\# 23. Disable User Account



This simulates an employee leaving the organization.



```powershell

Disable-ADAccount -Identity "asmith"

```



Verify:



```powershell

Get-ADUser "asmith" -Properties Enabled

```



Re-enable:



```powershell

Enable-ADAccount -Identity "asmith"

```



\---



\# 24. Review Group Membership



```powershell

Get-ADPrincipalGroupMembership "asmith"

```



Filter names:



```powershell

Get-ADPrincipalGroupMembership "asmith" |

Select-Object Name

```



\---



\# 25. Windows Endpoint Troubleshooting Commands



\## IP Configuration



```powershell

ipconfig

```



Detailed:



```powershell

ipconfig /all

```



Release DHCP lease:



```powershell

ipconfig /release

```



Renew:



```powershell

ipconfig /renew

```



Flush DNS:



```powershell

ipconfig /flushdns

```



\---



\# 26. Test Network Connectivity



Loopback:



```powershell

ping 127.0.0.1

```



Gateway:



```powershell

ping 192.168.1.1

```



Domain controller:



```powershell

ping 192.168.1.210

```



DNS name:



```powershell

ping DC01

```



Trace route:



```powershell

tracert 192.168.1.210

```



\---



\# 27. DNS Troubleshooting



Query domain:



```powershell

nslookup corp.lab

```



Query domain controller:



```powershell

nslookup DC01.corp.lab

```



PowerShell alternative:



```powershell

Resolve-DnsName corp.lab

```



Review DNS servers:



```powershell

Get-DnsClientServerAddress

```



\---



\# 28. Network Adapter Troubleshooting



List adapters:



```powershell

Get-NetAdapter

```



Detailed configuration:



```powershell

Get-NetIPConfiguration

```



Disable adapter:



```powershell

Disable-NetAdapter -Name "Ethernet" -Confirm:$false

```



Re-enable:



```powershell

Enable-NetAdapter -Name "Ethernet"

```



\---



\# 29. Windows Service Troubleshooting



List services:



```powershell

Get-Service

```



Check one service:



```powershell

Get-Service Spooler

```



Restart:



```powershell

Restart-Service Spooler

```



Check status:



```powershell

Get-Service Spooler

```



\---



\# 30. Printer Troubleshooting



Example printer troubleshooting workflow:



```text

Check printer power

&#x20;       ↓

Check network/USB connection

&#x20;       ↓

Check default printer

&#x20;       ↓

Check print queue

&#x20;       ↓

Check driver

&#x20;       ↓

Check Print Spooler

&#x20;       ↓

Restart spooler

&#x20;       ↓

Test print

```



PowerShell:



```powershell

Get-Printer

```



Check queue:



```powershell

Get-PrintJob -PrinterName "Office Printer"

```



Restart spooler:



```powershell

Restart-Service Spooler

```



\---



\# 31. Process Troubleshooting



List processes:



```powershell

Get-Process

```



Sort by CPU:



```powershell

Get-Process |

Sort-Object CPU -Descending |

Select-Object -First 10

```



Stop application:



```powershell

Stop-Process -Name notepad

```



Force:



```powershell

Stop-Process -Name notepad -Force

```



\---



\# 32. Disk Troubleshooting



Check disks:



```powershell

Get-Disk

```



Volumes:



```powershell

Get-Volume

```



Filesystem drives:



```powershell

Get-PSDrive -PSProvider FileSystem

```



Classic command:



```powershell

Get-CimInstance Win32\_LogicalDisk |

Select-Object DeviceID, Size, FreeSpace

```



\---



\# 33. Event Viewer Troubleshooting



Recent system events:



```powershell

Get-WinEvent -LogName System -MaxEvents 20

```



Recent application events:



```powershell

Get-WinEvent -LogName Application -MaxEvents 20

```



Errors only:



```powershell

Get-WinEvent -LogName System |

Where-Object {$\_.LevelDisplayName -eq "Error"} |

Select-Object -First 20

```



\---



\# 34. Help Desk Scenario — Locked Account



\## User Report



```text

"I can't sign into my computer."

```



\## Investigation



Check account:



```powershell

Get-ADUser "asmith" -Properties LockedOut

```



Or:



```powershell

Search-ADAccount -LockedOut

```



\## Resolution



```powershell

Unlock-ADAccount -Identity "asmith"

```



If a password reset is required:



```powershell

$password = ConvertTo-SecureString "TempP@ss456!" -AsPlainText -Force



Set-ADAccountPassword `

&#x20; -Identity "asmith" `

&#x20; -Reset `

&#x20; -NewPassword $password

```



Require password change:



```powershell

Set-ADUser "asmith" -ChangePasswordAtLogon $true

```



\## Validation



User successfully signs into:



```text

CORP\\asmith

```



\---



\# 35. Help Desk Scenario — Shared Drive Access



\## User Report



```text

"I can't open the Finance folder."

```



\## Investigation



Test server:



```powershell

ping DC01

```



Test share:



```powershell

Test-Path "\\\\DC01\\Finance"

```



Review user groups:



```powershell

Get-ADPrincipalGroupMembership "asmith" |

Select-Object Name

```



Review share:



```powershell

Get-SmbShareAccess -Name "Finance"

```



Review NTFS permissions:



```powershell

icacls "C:\\Shares\\Finance"

```



\## Root Cause



User was missing:



```text

Finance-Users

```



\## Fix



```powershell

Add-ADGroupMember `

&#x20; -Identity "Finance-Users" `

&#x20; -Members "asmith"

```



Verify:



```powershell

Get-ADGroupMember "Finance-Users"

```



\---



\# 36. Help Desk Scenario — Domain Login Failure



\## User Report



```text

"I can't log into the domain."

```



\## Troubleshooting



Check network:



```powershell

ipconfig /all

```



Test DC:



```powershell

ping 192.168.1.210

```



Check DNS:



```powershell

nslookup corp.lab

```



Check configured DNS:



```powershell

Get-DnsClientServerAddress

```



\## Root Cause



The client was configured to use the wrong DNS server.



\## Fix



```powershell

Set-DnsClientServerAddress `

&#x20; -InterfaceAlias "Ethernet" `

&#x20; -ServerAddresses 192.168.1.210

```



Flush DNS:



```powershell

ipconfig /flushdns

```



Test:



```powershell

nslookup corp.lab

```



\---



\# 37. Help Desk Scenario — Group Policy Not Applying



\## User Report



Expected workstation settings were missing.



Check applied policy:



```powershell

gpresult /r

```



Force refresh:



```powershell

gpupdate /force

```



Generate report:



```powershell

gpresult /h C:\\gp.html

```



Verify computer location in AD:



```powershell

Get-ADComputer "CLIENT01" -Properties DistinguishedName

```



If required, move workstation:



```powershell

Get-ADComputer "CLIENT01" |

Move-ADObject `

&#x20; -TargetPath "OU=Computers,OU=Finance,DC=corp,DC=lab"

```



Refresh:



```powershell

gpupdate /force

```



\---



\# 38. Help Desk Scenario — No Internet



\## User Report



```text

"The internet isn't working."

```



\## Troubleshooting Order



Check address:



```powershell

ipconfig /all

```



Test TCP/IP:



```powershell

ping 127.0.0.1

```



Test gateway:



```powershell

ping 192.168.1.1

```



Test public IP:



```powershell

ping 8.8.8.8

```



Test DNS:



```powershell

nslookup google.com

```



Trace path:



```powershell

tracert 8.8.8.8

```



Review routes:



```powershell

route print

```



PowerShell:



```powershell

Get-NetRoute

```



\---



\# 39. Help Desk Scenario — Printer Not Working



Check printers:



```powershell

Get-Printer

```



Check spooler:



```powershell

Get-Service Spooler

```



Restart:



```powershell

Restart-Service Spooler

```



Verify:



```powershell

Get-Service Spooler

```



Review Windows events if the problem continues:



```powershell

Get-WinEvent -LogName System -MaxEvents 50

```



\---



\# 40. Help Desk Scenario — Application Won't Launch



Check running process:



```powershell

Get-Process

```



Check application events:



```powershell

Get-WinEvent -LogName Application -MaxEvents 30

```



Restart application process if required:



```powershell

Stop-Process -Name "ApplicationName" -Force

```



Check system resources:



```powershell

Get-Process |

Sort-Object CPU -Descending |

Select-Object -First 10

```



Check available disk space:



```powershell

Get-Volume

```



\---



\# 41. Ticketing Workflow



Each issue followed a basic incident-management workflow.



```text

Ticket created

&#x20;     ↓

Review user report

&#x20;     ↓

Determine impact

&#x20;     ↓

Assign priority

&#x20;     ↓

Gather evidence

&#x20;     ↓

Troubleshoot

&#x20;     ↓

Identify root cause

&#x20;     ↓

Apply fix

&#x20;     ↓

Validate with user

&#x20;     ↓

Document resolution

&#x20;     ↓

Close ticket

```



\---



\# 42. Example Incident Ticket



```text

Ticket ID: INC-001



User:

Alice Smith



Department:

Finance



Device:

CLIENT01



Priority:

P3



Issue:

User cannot access Finance shared folder.



Impact:

User cannot access files required for daily work.



Troubleshooting:

\- Confirmed CLIENT01 network connectivity

\- Confirmed DC01 was reachable

\- Confirmed \\\\DC01\\Finance was online

\- Reviewed SMB share permissions

\- Reviewed user security-group membership

\- Found user was not a member of Finance-Users



Root Cause:

Missing Active Directory security-group membership.



Resolution:

Added user to Finance-Users group.



Validation:

User successfully accessed the Finance share.



Status:

Resolved

```



\---



\# 43. Incident Priority Model



Example priority system:



| Priority | Description                                   |

| -------- | --------------------------------------------- |

| P1       | Critical outage affecting many users          |

| P2       | Major department or service issue             |

| P3       | Individual user unable to perform normal work |

| P4       | Low-impact request or informational issue     |



\---



\# 44. Escalation Workflow



Issues that could not be resolved within Tier 1 scope were escalated with sufficient evidence.



Example escalation note:



```text

Affected User:

Alice Smith



Device:

CLIENT01



Issue:

Intermittent authentication failure.



Impact:

User unable to consistently access domain resources.



Troubleshooting Completed:

\- Verified IP address

\- Verified gateway

\- Verified DNS server

\- Confirmed DC01 reachable

\- Confirmed account not locked

\- Reviewed local Event Viewer

\- Reproduced issue



Current State:

Issue persists intermittently.



Escalation:

Request Tier 2 review of authentication and domain-controller logs.

```



\---



\# 45. Support Commands Reference



\## Active Directory



```powershell

Get-ADUser

Get-ADGroup

Get-ADGroupMember

Get-ADPrincipalGroupMembership

Search-ADAccount

Unlock-ADAccount

Set-ADAccountPassword

Enable-ADAccount

Disable-ADAccount

Get-ADComputer

Get-ADOrganizationalUnit

```



\## Network



```powershell

ipconfig /all

ping

tracert

nslookup

route print

Get-NetAdapter

Get-NetIPConfiguration

Get-NetRoute

Get-DnsClientServerAddress

Resolve-DnsName

```



\## Windows



```powershell

Get-Service

Restart-Service

Get-Process

Stop-Process

Get-WinEvent

Get-Disk

Get-Volume

Get-Printer

Get-PrintJob

```



\## Group Policy



```powershell

gpupdate /force

gpresult /r

gpresult /h C:\\gpresult.html

Get-GPO -All

```



\---



\# 46. Skills Demonstrated



\## Active Directory



\* Installing Active Directory Domain Services

\* Creating an AD forest

\* Domain administration

\* Creating users

\* Creating security groups

\* Creating Organizational Units

\* Password resets

\* Account unlocks

\* Group membership management

\* Computer-object management



\## Windows



\* Windows Server

\* Windows 11

\* PowerShell

\* Windows Services

\* Event Viewer

\* Printer troubleshooting

\* Process troubleshooting

\* Remote Desktop

\* Domain joins



\## Networking



\* TCP/IP

\* DNS

\* DHCP concepts

\* IP addressing

\* Gateway troubleshooting

\* Name resolution

\* Route validation

\* Connectivity testing



\## Enterprise Administration



\* Group Policy

\* SMB shares

\* NTFS permissions

\* Group-based access

\* Least privilege

\* Centralized authentication



\## Help Desk Operations



\* Incident triage

\* Password resets

\* User-access troubleshooting

\* Ticket documentation

\* Troubleshooting methodology

\* Priority assignment

\* Escalation

\* Resolution validation

\* User communication



\---



\# 47. Troubleshooting Methodology



The lab reinforced a structured troubleshooting process.



```text

1\. Identify the issue



2\. Determine scope and impact



3\. Gather symptoms



4\. Check simple causes first



5\. Determine failure domain



&#x20;  Endpoint

&#x20;     ↓

&#x20;  Network

&#x20;     ↓

&#x20;  DNS

&#x20;     ↓

&#x20;  Active Directory

&#x20;     ↓

&#x20;  Permissions

&#x20;     ↓

&#x20;  Application



6\. Make the smallest safe change



7\. Test the result



8\. Confirm user functionality



9\. Document the issue



10\. Escalate if necessary

```



\---



\# 48. Security Practices



The environment incorporated basic enterprise security practices.



These included:



\* Separate administrator and user accounts

\* Centralized authentication

\* Password policies

\* Account lockout policies

\* Least privilege

\* Group-based access control

\* Controlled shared-folder permissions

\* Administrative-change documentation

\* Restricted privileged access



\---



\# 49. Project Outcome



This project provided hands-on experience supporting a small Windows enterprise environment rather than working only with standalone computers.



The completed lab covered:



```text

Windows Server

&#x20;     ↓

Active Directory

&#x20;     ↓

DNS

&#x20;     ↓

Users / Groups / OUs

&#x20;     ↓

Windows 11 Domain Clients

&#x20;     ↓

Group Policy

&#x20;     ↓

File Shares / Permissions

&#x20;     ↓

Endpoint Troubleshooting

&#x20;     ↓

Incident Tickets

&#x20;     ↓

Resolution / Escalation

```



The project also reinforced that common help desk problems often involve several systems at the same time.



For example:



```text

User cannot access folder

&#x20;       ↓

Is workstation connected?

&#x20;       ↓

Can DNS resolve server?

&#x20;       ↓

Can server be reached?

&#x20;       ↓

Is user authenticated?

&#x20;       ↓

Correct AD group?

&#x20;       ↓

Correct share permission?

&#x20;       ↓

Correct NTFS permission?

&#x20;       ↓

Validate access

```



\---



\# Resume-Relevant Experience



This lab provides practical experience applicable to:



\* Help Desk Technician

\* Service Desk Analyst

\* Desktop Support Technician

\* IT Support Specialist

\* Junior Systems Administrator

\* IT Operations Technician

\* NOC Technician



Example resume bullet:



> Built and administered a Windows enterprise support lab using Windows Server, Active Directory, DNS, Group Policy, domain-joined Windows 11 endpoints, SMB shares, NTFS permissions, PowerShell, and remote administration; resolved simulated incidents involving account lockouts, password resets, DNS failures, network connectivity, permissions, printers, applications, and Group Policy.



Another possible bullet:



> Practiced end-to-end help desk incident workflows including user intake, impact assessment, troubleshooting, Active Directory administration, endpoint diagnostics, resolution validation, ticket documentation, and escalation.



\---



\# Future Improvements



Potential additions include:



\* Microsoft Entra ID

\* Microsoft 365 administration

\* Intune

\* Windows Autopilot

\* BitLocker administration

\* MFA support

\* Software deployment

\* WSUS

\* Windows Server DHCP

\* ServiceNow

\* GLPI

\* osTicket

\* Automated user provisioning with PowerShell

\* Hybrid Active Directory / Entra integration



\---



\# Project Status



```text

Windows Server VM                Complete

Static Server Networking         Complete

Active Directory Domain          Complete

DNS                              Complete

Organizational Units             Complete

Users                            Complete

Security Groups                  Complete

Domain-Joined Windows Clients    Complete

Group Policy                     Complete

SMB Shares                       Complete

NTFS Permissions                 Complete

Remote Administration            Complete

PowerShell Administration        Complete

Troubleshooting Scenarios        Complete

Incident Documentation           Complete

Escalation Workflow              Complete

```



\---



\# Conclusion



The Enterprise Help Desk Lab was designed to recreate the core responsibilities of an entry-level enterprise IT support environment.



The project went beyond simply creating Active Directory users by combining Windows Server, domain authentication, DNS, Group Policy, endpoint administration, permissions, networking, PowerShell, troubleshooting, incident documentation, and escalation into a single support environment.



The lab provides a repeatable platform for practicing enterprise support scenarios and serves as a practical demonstration of the skills required for Help Desk, Desktop Support, IT Operations, and Junior Systems Administration roles.



