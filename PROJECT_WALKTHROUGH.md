# SMALL ENTERPRISE INFRASTRUCTURE LAB — MASTER REVISION FILE

**Purpose of this file:**  
This is the complete revision/reference file for the enterprise infrastructure lab built in VMware. It is written so that after a month or more, you can reopen it and remember what was built, why each part was built, what each command did, what problems occurred, and how they were solved.

**Project type:** Small Enterprise Infrastructure / Junior System Engineer Lab  
**Core platforms:** Windows Server 2019, Windows 10, Kali Linux used as the Linux server component  
**Domain:** `corp.local`  
**Domain NetBIOS name:** `CORP`  
**Domain Controller / DNS Server:** `DC01`  
**Linux server hostname:** `LINUX01`

---

# 0. BIG PICTURE — WHAT WE BUILT

The goal was to simulate a small company network containing:

- A Windows Server 2019 Domain Controller
- Active Directory Domain Services (AD DS)
- DNS
- Departmental Organizational Units (OUs)
- Domain users
- Department security groups
- A Windows 10 domain workstation
- Shared department folders
- NTFS and Share Permissions
- Department isolation
- Password and account lockout policies
- User security restriction GPOs
- Workstation firewall GPO
- Automatic mapped network drives through GPO
- A centralized shared printer
- A Linux server on the same enterprise network
- Linux static IP and DNS integration
- SSH remote administration
- Apache internal intranet
- Linux firewall
- Linux integration with Windows Active Directory using Samba + Winbind
- A basic Linux user/group permission exercise
- Final validation of the full environment

Simplified topology:

```text
                         VMware Virtual Network
                         192.168.207.0/24
                                |
          -------------------------------------------------
          |                       |                       |
        DC01                 Windows 10                LINUX01
   Windows Server 2019       Domain Client            Kali Linux
   192.168.207.10            ~192.168.207.128         192.168.207.20
          |
   AD DS + DNS
   File Server
   Print Server
   Group Policy
          |
       corp.local
          |
   -------------------------------
   |        |        |            |
   HR    Finance     IT      Management
```

The VMware network gateway observed in the lab was:

```text
192.168.207.2
```

---

# 1. VIRTUAL MACHINE FOUNDATION

We used VMware Workstation for the lab.

The main machines were:

```text
Windows Server 2019  -> Enterprise server / DC
Windows 10 x64       -> Domain workstation/client
Kali Linux           -> Linux server component
```

The Linux VM was originally discussed as Debian, but the available installed machine was actually Kali Linux. Because storage space was limited, Kali was retained and used for normal Linux server/admin tasks.

Kali is Debian-based, so it can provide services such as SSH, Apache, Samba, Kerberos, Winbind, and UFW. We intentionally treated it as a Linux server in this project rather than turning the project into a penetration-testing lab.

## Important VM persistence concept

Shutting down a VM does **not** normally erase the configuration.

Your changes remain on the virtual disk after shutdown. You would normally lose the configuration only if you:

- delete/reinstall the VM,
- revert to an earlier snapshot,
- replace the virtual disk,
- or intentionally reset the machine.

This matters because the lab can safely be powered off and resumed later.

---

# 2. NETWORKING BASICS WE USED

Before Active Directory could work correctly, the server and client had to be able to communicate.

Important networking concepts:

## IP address

An IP address identifies a device on the network.

Example:

```text
DC01 = 192.168.207.10
LINUX01 = 192.168.207.20
```

## Subnet mask

The lab subnet mask was:

```text
255.255.255.0
```

This is the same as:

```text
/24
```

The network was therefore:

```text
192.168.207.0/24
```

Devices with addresses such as `.10`, `.20`, and `.128` are on the same local subnet.

## Default gateway

The gateway observed in VMware was:

```text
192.168.207.2
```

The gateway is the route used to reach networks outside the local subnet, including the internet.

## Static vs dynamic IP

A DHCP/dynamic IP can change.

A server should normally have a static/fixed IP because clients need to reliably find important services.

For example, if DC01 provided DNS at `192.168.207.10` but its IP later changed, domain clients could lose DNS and Active Directory connectivity.

That is why the server was given a static IP.

---

# 3. DC01 NETWORK CONFIGURATION

Windows Server 2019 was configured as:

```text
Hostname:         DC01
IPv4 address:     192.168.207.10
Subnet mask:      255.255.255.0
Default gateway:  192.168.207.2
DNS server:       192.168.207.10
Domain:           corp.local
```

`ipconfig /all` later confirmed:

```text
DHCP Enabled: No
IPv4 Address: 192.168.207.10
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.207.2
DNS Servers: ::1 and 192.168.207.10
DNS Suffix Search List: corp.local
```

The IPv6 loopback `::1` appearing in the DNS list is the local machine referring to itself over IPv6. The important IPv4 DNS address for the lab was `192.168.207.10`.

## Commands used

### `ipconfig`

```cmd
ipconfig
```

**Purpose:** Shows basic Windows IP configuration such as IPv4 address, subnet mask, and default gateway.

### `ipconfig /all`

```cmd
ipconfig /all
```

**Purpose:** Shows detailed adapter information including:

- hostname
- DHCP status
- MAC address
- DNS servers
- DNS suffix
- IPv4/IPv6 addresses
- default gateway

We used it later to confirm DC01 was using itself as DNS.

### `ping`

Example:

```cmd
ping 192.168.207.2
```

**Purpose:** Tests basic IP connectivity to a target.

We also tested internet reachability earlier using a public address such as:

```cmd
ping 8.8.8.8
```

If IP ping works but a hostname does not resolve, the problem is usually DNS rather than raw connectivity.

---

# 4. INSTALLING ACTIVE DIRECTORY DOMAIN SERVICES

On Windows Server 2019 we installed the **Active Directory Domain Services (AD DS)** role through Server Manager.

General path:

```text
Server Manager
-> Manage
-> Add Roles and Features
-> Role-based or feature-based installation
-> Select DC01
-> Active Directory Domain Services
-> Add Features
-> Install
```

## Why AD DS?

AD DS provides centralized identity and domain management.

Instead of creating unrelated local users on every computer, Active Directory allows a company administrator to centrally manage:

- users
- computers
- groups
- authentication
- Organizational Units
- Group Policy targeting
- domain resources

A useful mental model:

```text
AD DS = central identity/directory manager for the company
```

---

# 5. PROMOTING THE SERVER TO A DOMAIN CONTROLLER

After AD DS was installed, the server was promoted to a Domain Controller.

The domain created was:

```text
corp.local
```

The NetBIOS/domain prefix became:

```text
CORP
```

After promotion and reboot, the server became the Domain Controller:

```text
DC01.corp.local
```

## Early issue encountered

During the early build there was a point where the expected “Promote this server to a domain controller” notification was not visible.

The final successful state was that AD DS was installed and DC01 was promoted successfully.

The exact click-by-click resolution of that very early notification issue was not preserved in the later project transcript, so this revision file does not invent a fake resolution. The important verified result is that promotion completed and `corp.local` became operational.

---

# 6. DNS — WHY IT WAS CRITICAL

Active Directory depends heavily on DNS.

Domain clients need DNS to locate services such as:

```text
dc01.corp.local
corp.local
Kerberos
LDAP
domain controller service records
```

That is why domain clients should use the AD DNS server instead of a random public DNS server.

DC01 provided DNS at:

```text
192.168.207.10
```

## Commands used

### `nslookup corp.local`

```cmd
nslookup corp.local
```

**Purpose:** Asks DNS to resolve the domain name.

Expected result:

```text
Name: corp.local
Address: 192.168.207.10
```

### `nslookup dc01.corp.local`

```cmd
nslookup dc01.corp.local
```

**Purpose:** Confirms the fully qualified domain name of the DC resolves.

Final verified result:

```text
dc01.corp.local -> 192.168.207.10
```

### `dcdiag /test:dns`

```cmd
dcdiag /test:dns
```

**Purpose:** Runs Domain Controller DNS diagnostic tests.

Final validation showed:

```text
DC01 passed test DNS
corp.local passed test DNS
```

This was one of the final checks proving AD DNS health.

---

# 7. WINDOWS 10 DOMAIN CLIENT

The Windows 10 VM was configured on the same VMware network.

An observed Windows 10 address during the project was:

```text
192.168.207.128
```

It could communicate with:

```text
DC01 = 192.168.207.10
```

The client used DC01 for DNS so that `corp.local` could be resolved.

The Windows 10 machine was successfully joined to:

```text
corp.local
```

After joining, users could log in with domain accounts using the format:

```text
CORP\username
```

Examples:

```text
CORP\hr-user
CORP\Finance-user
CORP\IT-User
CORP\Manager
CORP\test-user
```

Actual account naming changed slightly during early experimentation, but the final department accounts were recognized by the domain and used successfully.

## Early domain-join issue

At one point the Windows client did not allow the Domain option to be selected in the system membership dialog.

The final verified state was:

```text
Windows 10 successfully joined corp.local
Domain authentication works
```

Because the exact early GUI fix was not retained in the available conversation record, this file does not fabricate one.

## `whoami`

```cmd
whoami
```

**Purpose:** Shows the exact security identity of the currently logged-in Windows user.

This was extremely useful because there were similarly named users.

Example result:

```text
corp\hr-user
```

This told us which AD account Windows was actually using.

---

# 8. ACTIVE DIRECTORY ORGANIZATIONAL UNITS

We created departmental OUs such as:

```text
corp.local
|
|-- HR
|-- Finance
|-- IT
|-- Management
|-- Workstations
```

## Purpose of an OU

An OU is an Active Directory organizational container.

It helps administrators:

- organize users and computers
- target GPOs
- delegate administration
- keep the directory structured

## VERY IMPORTANT CONCEPT

Being inside an OU does **not** automatically give access to a similarly named file folder.

Example:

```text
HR OU
  -> HR-User
```

does NOT automatically mean:

```text
HR-User can access C:\CompanyData\HR
```

Folder access comes from Share/NTFS permissions.

Remember:

```text
OU          = organization and GPO targeting
Permissions = file/folder access
GPO         = Windows/user/computer settings
```

This was one of the concepts that initially caused confusion, so it is important to keep these three systems separate.

---

# 9. DOMAIN USERS

Department users were created for:

- HR
- Finance
- IT
- Management

A separate user was later created for lockout testing:

```text
test-user
```

There were some older/test identities during the build, including similarly named objects such as:

```text
HRUser
HR-User
FinanceUser
Finance-user
```

This caused confusion at times.

We used:

```cmd
whoami
```

to confirm the active identity before troubleshooting permissions.

## Final lesson

In production, consistent naming matters.

A good naming convention avoids duplicate/confusing accounts.

---

# 10. COMPANY FILE SERVER STRUCTURE

A folder was created on DC01:

```text
C:\CompanyData
```

Inside it:

```text
C:\CompanyData
|
|-- HR
|-- Finance
|-- IT
|-- Management
```

The parent folder was shared over the network.

Network path:

```text
\\DC01\CompanyData
```

Department paths:

```text
\\DC01\CompanyData\HR
\\DC01\CompanyData\Finance
\\DC01\CompanyData\IT
\\DC01\CompanyData\Management
```

## Why use a centralized file share?

A company file server allows:

- centralized department data
- access control
- easier backup
- consistent network paths
- users to access data from domain workstations
- permissions to be managed centrally

---

# 11. SHARE PERMISSIONS VS NTFS PERMISSIONS

This became one of the most important lessons in the project.

## Share permissions

Share permissions apply when a folder is accessed over the network.

Example:

```text
\\DC01\CompanyData
```

Mental model:

```text
Share permission = network gate
```

Typical share permissions:

```text
Read
Change
Full Control
```

## NTFS permissions

NTFS permissions apply to the actual files and folders on the Windows filesystem.

Mental model:

```text
NTFS permission = what you can do inside the room
```

Common NTFS permissions:

```text
Read
Read & Execute
List Folder Contents
Write
Modify
Full Control
```

`Modify` normally includes:

- read
- create
- edit
- save
- delete

but not full administrative ownership/permission-changing abilities.

## Effective network access

When accessing a Windows share over the network, both Share and NTFS permissions matter.

A useful rule for this lab:

```text
Final network access is limited by whichever layer is more restrictive.
```

Example:

```text
Share = Read
NTFS  = Modify
```

Result over the network:

```text
User can read but cannot create/edit/delete
```

---

# 12. FIRST MAJOR FILE PERMISSION PROBLEM

The HR user had an NTFS entry showing:

```text
HR-User -> Modify
```

but on the Windows 10 client, the HR user could not create a text file.

We inspected HR folder Advanced Security and saw entries including:

```text
HRUser / HR-User
SYSTEM
Administrators
CORP\Users
CREATOR OWNER
```

There was no obvious explicit Deny entry.

## Effective Access issue

When trying to evaluate Effective Access from the remote client, Windows displayed:

```text
You do not have permission to evaluate effective access rights for the remote source.
Contact the administrator for the target server.
```

### Why?

Effective Access calculation against a remote resource may require privileges on the target server.

### Resolution

We checked Effective Access locally on DC01 as an administrator.

But the root problem turned out not to be an explicit NTFS Deny.

## Real cause

The Share Permission on CompanyData was:

```text
Everyone:
Read = Allow
Change = Not allowed
Full Control = Not allowed
```

So even though HR had NTFS Modify permission, the network share only allowed Read.

## Fix

Share permissions were changed to:

```text
Everyone:
Read   = Allow
Change = Allow
```

Full Control was not necessary for normal users.

After that, the HR user could:

```text
create a .txt file
edit/save it
delete it
```

## Lesson

```text
NTFS Modify + Share Read = effectively Read over network
NTFS Modify + Share Change = Modify actions can work
```

---

# 13. SECOND MAJOR FILE PERMISSION PROBLEM — DEPARTMENT ISOLATION

After Share permissions were changed to allow Change, the HR user could also create/delete files inside:

```text
Finance
Management
```

This was incorrect.

Expected:

```text
HR user -> HR only
Finance user -> Finance only
IT user -> IT only
Management user -> Management only
```

## Cause

Department folders were inheriting broad permissions from the parent.

For example, an inherited permission such as:

```text
CORP\Users
```

could give a domain user access to folders outside their department.

## Inheritance

Inheritance means:

```text
Child folders automatically receive permission entries from their parent.
```

Mental model:

```text
Inheritance = permission copying from parent to child
```

It does NOT mean “blocking other departments.”

## Fix

On the departmental folders we used:

```text
Properties
-> Security
-> Advanced
-> Disable inheritance
```

Then unnecessary inherited/broad entries such as `Users` were removed while keeping required administrative entries.

Typical clean department folder entries were:

```text
SYSTEM         -> Full Control
Administrators -> Full Control
Department user/group -> Modify
```

After correcting inheritance and permissions:

```text
HR user         -> HR allowed, others denied
Finance user    -> Finance allowed, others denied
IT user         -> IT allowed, others denied
Management user -> Management allowed, others denied
```

## Critical revision rule

```text
Share        = network gate
NTFS         = file/folder rights
Inheritance  = parent NTFS permissions flowing to child objects
OU           = AD organization/GPO targeting
```

Do not mix these concepts.

---

# 14. DEPARTMENT ACCESS TESTING

Each department account was tested from Windows 10.

A correct department test included:

1. Log in using the department domain account.
2. Open:
   `\\DC01\CompanyData`
3. Open the user's own department.
4. Create a text file.
5. Edit it.
6. Save it.
7. Delete it.
8. Try opening other department folders.

Expected result:

```text
Own folder       -> Modify allowed
Other departments -> Access Denied
```

Management and IT were explicitly tested this way, and all department isolation was eventually verified.

Network Discovery being empty did not mean the share was broken.

A direct UNC path was more reliable:

```text
\\DC01\CompanyData
```

This bypasses the visual “Network” browsing view.

---

# 15. SECURITY GROUP CLEANUP — MORE ENTERPRISE-LIKE DESIGN

Initially, permissions were assigned directly to individual department users.

That works technically but becomes difficult to manage at scale.

We improved the design by creating Global Security Groups:

```text
GG_HR_Users
GG_Finance_Users
GG_IT_Users
GG_Management_Users
```

Configuration:

```text
Group Scope: Global
Group Type: Security
```

Users were added to the matching group.

Example:

```text
HR-User
   |
   v
GG_HR_Users
   |
   v
C:\CompanyData\HR -> Modify
```

## Why groups are better

Without groups:

```text
Folder -> Alice
Folder -> Bob
Folder -> Charlie
```

With groups:

```text
Users -> Department Security Group -> Folder Permission
```

When a new HR employee arrives, the admin only adds them to `GG_HR_Users`.

The folder ACL does not need to be edited for every employee.

## Migration process we used

For each department:

1. Create the new security group.
2. Add the department user to it.
3. Add the group to the folder with Modify.
4. Test create/edit/delete while the old direct user entry still exists.
5. Remove the direct individual permission.
6. Test again.
7. If it still works, group-based access is confirmed.

This was performed for:

```text
HR
Finance
IT
Management
```

## Cleanup note

An older `HRUser` group was deleted after confirming the new `GG_HR_Users` model worked.

An older Finance-related security group was visible during the project. Before deleting any remaining old object in the future, first verify it is not used by a folder ACL or GPO. Do not delete objects only because the name looks old.

---

# 16. GROUP POLICY BASICS

Group Policy lets domain administrators centrally configure Windows users and computers.

Two major sections:

## User Configuration

Targets settings for the user.

Examples from our project:

```text
Block Control Panel
Block Settings
Block Command Prompt
Map network drives
Deploy printer per-user
```

## Computer Configuration

Targets the computer.

Example from our project:

```text
Windows Defender Firewall settings
```

Simple rule:

```text
User Configuration     -> rules for the person/account
Computer Configuration -> rules for the machine
```

## GPO targeting with OUs

If a GPO is linked to the HR OU, users inside HR can receive that user-side policy.

If a Computer Configuration GPO is linked to the Workstations OU, computer objects inside Workstations can receive it.

A GPO linked to an OU does not automatically affect unrelated sibling OUs.

---

# 17. PASSWORD + ACCOUNT LOCKOUT GPO

We created:

```text
CORP - Password & Account Lockout Policy
```

It was linked at the domain level.

Configured password settings included:

```text
Minimum password length: 8 characters
Password complexity: Enabled
Minimum password age: 30 days
Maximum password age: 90 days
```

Note: the 30-day minimum password age was intentionally retained in the lab when asked whether it could be left as-is. It was not separately tested during final validation.

## Account lockout settings

```text
Account lockout threshold: 5 failed attempts
Account lockout duration: 15 minutes
Reset account lockout counter after: 15 minutes
Allow Administrator account lockout: Enabled
```

## Why account lockout?

It reduces repeated password-guessing attempts against accounts.

## Safety note

We intentionally did not test the lockout using the real Administrator account.

We created:

```text
test-user
```

for testing.

## Test performed

Five incorrect passwords were entered.

Windows showed:

```text
The referenced account is currently locked out and may not be logged on to.
```

## Unlock procedure

On DC01:

```text
Active Directory Users and Computers
-> locate test-user
-> Properties
-> Account
-> Unlock account
-> Apply
```

Then the correct password worked again.

This was tested successfully both during setup and final validation.

---

# 18. GROUP POLICY COMMANDS

## `gpupdate /force`

```cmd
gpupdate /force
```

**Purpose:** Forces Windows to refresh/reapply user and computer Group Policy immediately rather than waiting for the normal background refresh interval.

Typical successful output:

```text
Computer Policy update has completed successfully.
User Policy update has completed successfully.
```

Important: it does not mean “open the GPO.” It tells Windows to process the latest policies.

## `gpresult /r`

```cmd
gpresult /r
```

**Purpose:** Shows Resultant Set of Policy summary and which GPOs are applied.

We used it to verify GPOs such as:

```text
Default Domain Policy
Company Security Policy
CORP - Password & Account Lockout Policy
CORP - Workstation Security Policy
```

depending on user/computer context.

## `gpresult /h`

```cmd
gpresult /h C:\GPReport.html
```

**Purpose:** Generates a detailed HTML report of Group Policy results.

This is useful for troubleshooting winning GPOs, applied settings, and filtering.

## `gpresult /scope user /v`

```cmd
gpresult /scope user /v
```

**Purpose:** Shows verbose user-side Group Policy results.

During printer troubleshooting, this was run while logged in as:

```text
CORP\Administrator
```

and the output showed no applied user-side GPOs for that specific session.

This reminded us that troubleshooting must be performed in the correct user context.

## `rsop.msc`

```text
rsop.msc
```

**Purpose:** Opens Resultant Set of Policy graphical console.

We considered it while investigating printer policy state.

---

# 19. SECURITY RESTRICTIONS GPO

We created:

```text
CORP - Security Restrictions Policy
```

User-side restrictions configured:

## Block Control Panel and Settings

Path:

```text
User Configuration
-> Policies
-> Administrative Templates
-> Control Panel
-> Prohibit access to Control Panel and PC settings
```

Set to:

```text
Enabled
```

## Block Command Prompt

Path:

```text
User Configuration
-> Policies
-> Administrative Templates
-> System
-> Prevent access to the command prompt
```

Set to:

```text
Enabled
```

## Block removable storage

Path:

```text
User Configuration
-> Policies
-> Administrative Templates
-> System
-> Removable Storage Access
-> All Removable Storage classes: Deny all access
```

Set to:

```text
Enabled
```

## Testing

Normal users were confirmed unable to open:

```text
Command Prompt
Control Panel
Windows Settings
```

The removable-storage/USB restriction was configured but **not physically tested** because a USB device was not available.

## Troubleshooting consequence

Because CMD/Control Panel/Settings were intentionally blocked for normal users, later testing had to respect that.

For example, we could not simply tell a Finance or HR user to open an elevated CMD for every check.

This became important during printer troubleshooting.

---

# 20. WORKSTATIONS OU

The Windows 10 computer object was originally in the default:

```text
Computers
```

container.

We moved it into:

```text
Workstations
```

OU.

Why?

Because we wanted computer-specific GPOs to target workstation computers cleanly.

Concept:

```text
Computer object in Workstations OU
            |
            v
Computer Configuration GPO linked to Workstations
            |
            v
Workstation receives policy
```

---

# 21. WINDOWS WORKSTATION FIREWALL GPO

We created:

```text
CORP - Workstation Security Policy
```

and linked it to:

```text
Workstations OU
```

Inside:

```text
Computer Configuration
-> Policies
-> Windows Settings
-> Security Settings
-> Windows Defender Firewall with Advanced Security
```

For Domain, Private, and Public profiles we configured:

```text
Firewall state: On
Inbound connections: Block
Outbound connections: Allow
```

## Why?

A secure baseline is:

- block unsolicited inbound traffic unless a rule allows it
- permit normal outbound traffic

## Verification command

```text
wf.msc
```

**Purpose:** Opens Windows Defender Firewall with Advanced Security.

When opened as an administrator, the final validation showed:

```text
Domain Profile is Active
Windows Defender Firewall is on
Inbound connections that do not match a rule are blocked
Outbound connections that do not match a rule are allowed
```

It also displayed:

```text
For your security, some settings are controlled by Group Policy
```

which proved policy control.

## Normal-user behavior

When Finance user ran `wf.msc`, Windows displayed an access/permission error because the user was not an Administrator/Network Operator.

That did **not** mean the firewall GPO was broken.

We rechecked it using an administrative session.

---

# 22. AUTOMATIC MAPPED NETWORK DRIVES THROUGH GPO

We created separate user GPOs linked to department OUs.

## HR

GPO:

```text
CORP - HR Drive Mapping
```

Drive:

```text
H:
\\DC01\CompanyData\HR
Label: HR Drive
```

## Finance

GPO:

```text
CORP - Finance Drive Mapping
```

Drive:

```text
F:
\\DC01\CompanyData\Finance
Label: Finance Drive
```

## IT

GPO:

```text
CORP - IT Drive Mapping
```

Drive:

```text
I:
\\DC01\CompanyData\IT
Label: IT Drive
```

## Management

GPO:

```text
CORP - Management Drive Mapping
```

Drive:

```text
M:
\\DC01\CompanyData\Management
Label: Management Drive
```

## GPO path

```text
User Configuration
-> Preferences
-> Windows Settings
-> Drive Maps
```

Action used:

```text
Create
```

## Why mapped drives?

Instead of telling every employee to manually browse:

```text
\\DC01\CompanyData\Finance
```

the company can automatically provide:

```text
Finance Drive (F:)
```

when the Finance user logs in.

This improves usability and central management.

## Testing

Each department drive was visibly confirmed under:

```text
File Explorer -> This PC -> Network locations
```

Examples observed:

```text
HR Drive (H:)
Finance Drive (F:)
IT Drive (I:)
Management Drive (M:)
```

Final validation used Finance and confirmed:

```text
Finance domain login works
F: drive appears and opens
Other department folders are denied
```

---

# 23. PRINT SERVER ROLE

We installed:

```text
Print and Document Services
-> Print Server
```

through Server Manager.

Then opened:

```text
Server Manager
-> Tools
-> Print Management
```

Server tree:

```text
Print Servers
-> DC01
-> Printers
```

## Why centralized printing?

A print server allows administrators to:

- centrally manage printers
- share printers
- control drivers
- deploy printers to users/computers
- avoid configuring every workstation manually

---

# 24. PRINTER PROBLEM 1 — MICROSOFT PRINT TO PDF

The first dummy printer was created using:

```text
Microsoft Print to PDF
```

with a print-to-file style port.

We named it:

```text
CORP-Office-Printer
```

When we tried to share it, Windows displayed:

```text
Sharing is not supported for this type of printer.
```

## Cause

Microsoft Print to PDF is a local virtual print-to-PDF device, not an appropriate shared network printer for this lab.

## Resolution

We created a new dummy printer using:

```text
Generic
-> Generic / Text Only
```

and a normal local printer port such as:

```text
LPT1:
```

The correct printer was named:

```text
CORP-Office-Printer-Shared
```

and shared with the same share name.

Network path:

```text
\\DC01\CORP-Office-Printer-Shared
```

The old incorrect `CORP-Office-Printer` was deleted to avoid confusion.

---

# 25. PRINTER DEPLOYMENT TROUBLESHOOTING

Initially we used Print Management's:

```text
Deploy with Group Policy
```

method.

A GPO was created:

```text
CORP - Office Printer Deployment
```

The client later showed this during:

```cmd
gpupdate /force
```

Error/warning:

```text
Windows failed to apply the Deployed Printer Connections settings.
```

## Manual network test

On Windows 10 we opened:

```text
\\DC01
```

and saw:

```text
CompanyData
netlogon
sysvol
CORP-Office-Printer-Shared
```

Double-clicking the printer successfully opened the print queue.

This proved:

```text
Network path works
Printer share is reachable
Driver/printer connection can work
```

## Better deployment method

We removed the old/legacy deployed-printer configuration and used Group Policy Preferences instead.

Path:

```text
User Configuration
-> Preferences
-> Control Panel Settings
-> Printers
```

Created:

```text
New -> Shared Printer
Action: Update
Share path: \\DC01\CORP-Office-Printer-Shared
```

Why `Update`?

`Update` can create the connection when missing and maintain/update it afterward.

## Event Viewer troubleshooting

When trying to inspect Group Policy logs, Event Viewer initially displayed:

```text
Access is denied (5)
```

for the GroupPolicy Operational log.

### Resolution

Event Viewer was reopened using:

```text
Run as administrator
```

Then the log could be read:

```text
Applications and Services Logs
-> Microsoft
-> Windows
-> GroupPolicy
-> Operational
```

Eventually:

```cmd
gpupdate /force
```

completed without the old Deployed Printer Connections warning.

## Printer verification

Because normal users had Settings and Control Panel blocked, we used:

```text
Windows + R
shell:PrintersFolder
```

This opened the Printers folder.

It showed:

```text
CORP-Office-Printer-Shared on DC01
```

Final validation confirmed the printer was still present.

## Accuracy note

During troubleshooting, the printer was also manually opened from `\\DC01`. Therefore, if you want a perfect “clean-room” proof later, remove the printer from a disposable test-user profile and verify that the GPP deployment recreates it automatically at next policy/logon. The project already proves the shared printer connection is present and the previous GPO warning was resolved.

---

# 26. LINUX SERVER DECISION

The Linux VM turned out to be Kali Linux, not a separate Debian VM.

Because there was not enough storage to add another VM, Kali was retained.

We intentionally used it for normal server/admin tasks:

- static IP
- DNS
- SSH
- Apache
- firewall
- Samba/Winbind AD integration
- Linux users/groups/permissions

Hostname later became:

```text
LINUX01
```

---

# 27. CHECKING THE ORIGINAL LINUX NETWORK

Commands:

```bash
ip addr
ip route
```

Initial output showed:

```text
Interface: eth0
IP: 192.168.207.129/24
Address type: dynamic/DHCP
Gateway: 192.168.207.2
```

## `ip addr`

**Purpose:** Shows interfaces, IP addresses, and link state.

## `ip route`

**Purpose:** Shows Linux routing table, including the default gateway.

Example:

```text
default via 192.168.207.2 dev eth0
```

---

# 28. SELECTING A STATIC IP FOR LINUX01

We wanted Linux to behave like a server, so it needed a stable IP.

Proposed address:

```text
192.168.207.20
```

Before assigning it, DC01 tested:

```powershell
ping 192.168.207.20
```

DC01 responded:

```text
Destination host unreachable
```

This strongly indicated that `.20` was not currently in use.

Final Linux network plan:

```text
IP:      192.168.207.20/24
Gateway: 192.168.207.2
DNS:     192.168.207.10
```

---

# 29. CONFIGURING STATIC IP WITH NETWORKMANAGER

First:

```bash
nmcli connection show
```

Result included:

```text
Wired connection 1
TYPE: ethernet
DEVICE: eth0
```

## `nmcli`

`nmcli` is the command-line client for NetworkManager.

We used it because the user preferred terminal-based network configuration.

Commands:

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.addresses 192.168.207.20/24
```

**Purpose:** Sets the static IPv4 address and prefix.

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.gateway 192.168.207.2
```

**Purpose:** Sets the default IPv4 gateway.

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.dns 192.168.207.10
```

**Purpose:** Makes DC01 the Linux DNS server.

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.method manual
```

**Purpose:** Switches the connection from DHCP/automatic addressing to manual/static IPv4.

Then:

```bash
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

**Purpose:** Restarts the NetworkManager connection so the new settings take effect.

Verification:

```bash
ip addr
ip route
```

Final result:

```text
eth0 = 192.168.207.20/24
default via 192.168.207.2
```

---

# 30. LINUX DNS INTEGRATION WITH DC01

We checked:

```bash
cat /etc/resolv.conf
```

Result:

```text
# Generated by NetworkManager
nameserver 192.168.207.10
```

## `cat /etc/resolv.conf`

**Purpose:** Shows the DNS resolver configuration that Linux is currently using.

Then:

```bash
ping -c 4 192.168.207.10
```

**Purpose:** Tests direct connectivity to DC01 by IP.

Then:

```bash
ping -c 4 dc01.corp.local
```

**Purpose:** Tests both DNS name resolution and network reachability.

Result:

```text
dc01.corp.local -> 192.168.207.10
0% packet loss
```

This proved Linux was successfully using Windows DNS.

---

# 31. LINUX HOSTNAME

Kali initially used hostname:

```text
kali
```

We changed it to:

```text
LINUX01
```

Command:

```bash
sudo hostnamectl set-hostname LINUX01
```

## `hostnamectl`

**Purpose:** Reads or changes Linux hostname information.

Verification:

```bash
hostnamectl
```

We also updated:

```text
/etc/hosts
```

so the local hostname mapping referred to `LINUX01`.

Then:

```bash
sudo reboot
```

After reboot the prompt displayed:

```text
kali@LINUX01
```

confirming the hostname change.

---

# 32. SSH SERVER

We installed OpenSSH Server:

```bash
sudo apt update
sudo apt install openssh-server -y
```

## `sudo apt update`

**Purpose:** Downloads the current package indexes from configured repositories.

It does not install upgrades by itself.

## `sudo apt install openssh-server -y`

**Purpose:** Installs the SSH server package.

`-y` automatically answers “yes” to the normal package confirmation prompt.

Then:

```bash
sudo systemctl enable ssh
```

**Purpose:** Configures SSH to start automatically during boot.

```bash
sudo systemctl start ssh
```

**Purpose:** Starts the SSH service now.

```bash
sudo systemctl status ssh
```

or later:

```bash
sudo systemctl status ssh --no-pager
```

**Purpose:** Displays the SSH service state.

Verified:

```text
Active: active (running)
Server listening on 0.0.0.0 port 22
Server listening on :: port 22
```

---

# 33. SSH FROM WINDOWS TO LINUX

From Windows 10 PowerShell:

```powershell
ssh kali@192.168.207.20
```

## Purpose

Creates a remote encrypted command-line session from Windows 10 to LINUX01.

On first connection, SSH asked whether to trust the host key:

```text
Are you sure you want to continue connecting?
```

We typed:

```text
yes
```

Then entered the Kali user's password.

Successful result:

```text
kali@LINUX01
```

This proved:

```text
Windows 10 -> TCP 22 -> SSH -> LINUX01
```

Remote Linux administration was working.

---

# 34. APACHE INTERNAL WEB SERVER

Installed:

```bash
sudo apt install apache2 -y
```

Enabled and started:

```bash
sudo systemctl enable apache2
sudo systemctl start apache2
```

Verified:

```bash
sudo systemctl status apache2
```

or:

```bash
sudo systemctl status apache2 --no-pager
```

Result:

```text
Active: active (running)
```

## Why Apache?

It gave LINUX01 an actual enterprise service role: an internal web/intranet server.

## Windows test

Windows 10 opened:

```text
http://192.168.207.20
```

and initially saw the default:

```text
Apache2 Debian Default Page
It works!
```

This proved HTTP connectivity from Windows to Linux.

---

# 35. CUSTOM INTRANET PAGE

Apache's default site file was:

```text
/var/www/html/index.html
```

Editing through `nano` was difficult because clipboard paste was not working well.

We decided to use a GUI editor.

## Package installation problem

Trying to install a GUI editor initially failed because:

```text
Temporary failure resolving 'http.kali.org'
```

This was a DNS forwarding problem, not an editor problem.

---

# 36. DNS FORWARDER PROBLEM AND FIX

LINUX01 used DC01 (`192.168.207.10`) as DNS.

That worked for:

```text
dc01.corp.local
corp.local
```

but Kali could not resolve public repository names such as:

```text
http.kali.org
```

## Cause

DC01 DNS could answer internal AD DNS requests but did not have a working path/forwarder for the public DNS query in this lab setup.

## Fix on DC01

Opened:

```text
Server Manager
-> Tools
-> DNS
-> DC01 Properties
-> Forwarders
```

Added public forwarders:

```text
8.8.8.8
1.1.1.1
```

After this:

```bash
sudo apt update
```

worked on Kali.

## Why DNS forwarders matter

Clients can continue using the domain DNS server for everything.

DC01 answers internal names itself and forwards unknown public internet names to external DNS resolvers.

This preserves correct AD DNS usage while still allowing internet name resolution.

---

# 37. GEANY AND THE CUSTOM HTML PAGE

Installed:

```bash
sudo apt install geany -y
```

Opened Apache index file as root:

```bash
sudo geany /var/www/html/index.html
```

The default Apache HTML was replaced with:

```html
<!DOCTYPE html>
<html>
<head>
    <title>CORP Internal Portal</title>
</head>
<body>
    <h1>CORP Internal Portal</h1>
    <p>Welcome to the internal company web server.</p>

    <h2>Departments</h2>
    <ul>
        <li>HR</li>
        <li>Finance</li>
        <li>IT</li>
        <li>Management</li>
    </ul>

    <p>Server: LINUX01</p>
    <p>Domain: corp.local</p>
</body>
</html>
```

After saving, Windows refreshed:

```text
http://192.168.207.20
```

and successfully displayed:

```text
CORP Internal Portal
Departments:
HR
Finance
IT
Management
Server: LINUX01
Domain: corp.local
```

This proved the custom Linux intranet was operational.

---

# 38. LINUX ACTIVE DIRECTORY INTEGRATION — FIRST PLAN: SSSD/REALMD

We wanted to make the Linux integration stronger by allowing Linux to recognize Windows AD identities.

The preferred modern approach was initially:

```text
realmd + SSSD
```

Attempted install:

```bash
sudo apt install realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin packagekit -y
```

But Kali reported errors such as:

```text
Unable to locate package sssd
Unable to locate package sssd-tools
Package libnss-sss has no installation candidate
Package libpam-sss has no installation candidate
Unable to locate package samba-common-bin
```

## Troubleshooting package state

Commands:

```bash
apt policy sssd realmd adcli
```

This showed:

```text
realmd installed
adcli available
sssd unable to locate
```

Then:

```bash
cat /etc/apt/sources.list
```

returned:

```text
No such file or directory
```

On this Kali version, the repository configuration was instead in:

```bash
cat /etc/apt/sources.list.d/kali.sources
```

It correctly showed:

```text
Types: deb
URIs: http://http.kali.org/kali/
Suites: kali-rolling
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/kali-archive-keyring.gpg
```

We also tried refreshing package lists:

```bash
sudo apt clean
sudo rm -rf /var/lib/apt/lists/*
sudo apt update
```

Then:

```bash
apt policy sssd
```

still showed:

```text
Unable to locate package sssd
```

A final retry after package update/upgrade still did not expose SSSD.

## Decision

We did **not** mix Debian repositories into Kali or install random `.deb` packages.

That could destabilize the distribution.

Instead, we used:

```text
Samba + Winbind
```

as the domain-member integration method.

---

# 39. SAMBA + WINBIND AD INTEGRATION

Installed:

```bash
sudo apt install samba winbind libnss-winbind libpam-winbind krb5-user -y
```

This installed:

- Samba
- Winbind
- NSS Winbind module
- PAM Winbind module
- Kerberos user tools

Winbind initially did not start automatically because domain-member configuration was not complete yet. That was expected.

---

# 40. KERBEROS TEST

We tested AD Kerberos authentication before the domain join.

Command:

```bash
kinit Administrator@CORP.LOCAL
```

## `kinit`

**Purpose:** Requests a Kerberos Ticket Granting Ticket (TGT) for the specified principal.

The AD Administrator password was entered.

Then:

```bash
klist
```

## `klist`

**Purpose:** Displays Kerberos tickets currently stored in the user's credential cache.

Successful output included:

```text
Default principal: Administrator@CORP.LOCAL
krbtgt/CORP.LOCAL@CORP.LOCAL
```

This proved:

```text
LINUX01 can authenticate to CORP.LOCAL Kerberos
```

---

# 41. SAMBA DOMAIN MEMBER CONFIGURATION

Before editing:

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

## Purpose

Creates a backup of the original Samba configuration before modification.

Because terminal paste was difficult, Geany was used:

```bash
sudo geany /etc/samba/smb.conf
```

Final minimal configuration:

```ini
[global]
   workgroup = CORP
   realm = CORP.LOCAL
   security = ADS

   kerberos method = secrets and keytab

   winbind use default domain = yes
   winbind offline logon = yes
   winbind enum users = yes
   winbind enum groups = yes

   idmap config * : backend = tdb
   idmap config * : range = 3000-7999

   idmap config CORP : backend = rid
   idmap config CORP : range = 10000-999999

   template shell = /bin/bash
   template homedir = /home/%U
```

## Important settings explained

### `workgroup = CORP`

Sets the NetBIOS domain/workgroup name.

### `realm = CORP.LOCAL`

Sets the Kerberos realm.

Kerberos realm is conventionally uppercase.

### `security = ADS`

Tells Samba to operate as an Active Directory domain member using ADS security.

### `kerberos method = secrets and keytab`

Uses Samba's machine secrets and keytab for Kerberos-related operations.

### `winbind use default domain = yes`

Allows users to be referenced without always typing `CORP\` in some Winbind contexts.

### `winbind enum users = yes`

Allows enumeration/listing of domain users.

### `winbind enum groups = yes`

Allows enumeration/listing of domain groups.

### ID mapping

Linux requires numeric UID/GID values.

The `idmap config` entries map AD identities into Linux numeric ID ranges.

### `template shell = /bin/bash`

Provides Bash as the default shell value for mapped domain users.

### `template homedir = /home/%U`

Defines a default home-directory path format for domain users.

---

# 42. VALIDATING SAMBA CONFIGURATION

Command:

```bash
testparm
```

## Purpose

Checks `/etc/samba/smb.conf` for syntax/configuration errors.

Result:

```text
Loaded services file OK.
Server role: ROLE_DOMAIN_MEMBER
```

Warnings about `/run/samba` not existing appeared before the services had fully created their runtime directories. They did not block the domain member configuration.

`testparm` also displayed an advisory Kerberos/keytab suggestion. The configuration still loaded successfully and domain join later worked.

---

# 43. JOINING LINUX01 TO CORP.LOCAL

Command:

```bash
sudo net ads join -U Administrator
```

## Purpose

Joins the Samba Linux machine to the Active Directory domain using an authorized AD account.

Successful output:

```text
Using short domain name -- CORP
Joined 'LINUX01' to dns domain 'corp.local'
```

This created/used the machine trust relationship between LINUX01 and Active Directory.

Then:

```bash
sudo systemctl enable --now winbind
```

## Purpose

- `enable` = start Winbind automatically at boot
- `--now` = also start it immediately

---

# 44. WINBIND TROUBLESHOOTING

Initial command:

```bash
wbinfo -t
```

returned:

```text
checking the trust secret for domain CORP via RPC calls failed
WBC_ERR_WINBIND_NOT_AVAILABLE
Could not check secret
```

This looked serious, so we did not immediately rejoin the domain.

## Service status

```bash
sudo systemctl status winbind --no-pager
```

showed:

```text
Active: active (running)
winbindd: ready to serve connections
```

So the service itself was running.

## Runtime socket checks

```bash
ls -l /run/samba
```

showed Samba runtime files and directories.

```bash
ls -l /run/samba/winbindd
```

showed:

```text
pipe
```

The local Winbind socket existed.

## Logs

```bash
sudo journalctl -u winbind -n 50 --no-pager
```

**Purpose:** Shows the most recent 50 Winbind service log entries without opening an interactive pager.

No major start failure was visible.

## Domain join verification

```bash
sudo net ads testjoin
```

Result:

```text
Join is OK
```

This was critical: the AD machine trust itself was valid.

---

# 45. NSS / LINUX IDENTITY LOOKUP

We checked:

```bash
grep -E '^(passwd|group):' /etc/nsswitch.conf
```

Result:

```text
passwd: files systemd winbind
group:  files systemd winbind
```

## Purpose

`/etc/nsswitch.conf` tells Linux where to look up identities.

Including `winbind` means Linux can ask Winbind for AD users/groups in addition to local `/etc/passwd` and `/etc/group` information.

`getent passwd` and `getent group` initially mainly showed local identities during troubleshooting.

---

# 46. WINBIND FINAL FIX/VERIFICATION

We restarted Winbind:

```bash
sudo systemctl restart winbind
```

Then:

```bash
wbinfo -p
```

Result:

```text
Ping to winbindd succeeded
```

## `wbinfo -p`

**Purpose:** Tests whether the `wbinfo` client can communicate with the local Winbind daemon.

Plain:

```bash
wbinfo -t
```

still failed in this environment.

Then we ran:

```bash
sudo wbinfo -t
```

Result:

```text
checking the trust secret for domain CORP via RPC calls succeeded
```

So the trust-secret check required elevated privileges in this setup.

We also ran:

```bash
sudo wbinfo --ping-dc
```

Result showed a successful NETLOGON connection to:

```text
DC01.corp.local
```

Then:

```bash
wbinfo --online-status
```

showed:

```text
CORP : active connection
```

## Domain user enumeration

```bash
wbinfo -u
```

listed AD users such as:

```text
it-user
financeuser1
hr-user
manager01
test-user
```

## Domain group enumeration

```bash
wbinfo -g
```

listed AD groups including:

```text
domain users
domain admins
it-admins
gg_hr_users
gg_finance_users
gg_it_users
gg_management_users
```

## Linux lookup of an AD user

```bash
getent passwd HR-User
```

returned an entry similar to:

```text
hr-user:x:11113:10513::/home/hr-user:/bin/bash
```

This proved:

```text
Linux NSS -> Winbind -> Active Directory
```

was functioning.

Final validation again confirmed:

```bash
sudo net ads testjoin
```

returned:

```text
Join is OK
```

---

# 47. BASIC LINUX FIREWALL — UFW

We installed:

```bash
sudo apt install ufw -y
```

Then allowed only the services required by this project:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
```

Port meanings:

```text
22/tcp -> SSH
80/tcp -> HTTP/Apache
```

Then:

```bash
sudo ufw enable
```

UFW reported:

```text
Firewall is active and enabled on system startup
```

Verification:

```bash
sudo ufw status verbose
```

Final state:

```text
Status: active
Default: deny (incoming), allow (outgoing), disabled (routed)

22/tcp      ALLOW IN Anywhere
80/tcp      ALLOW IN Anywhere
22/tcp(v6)  ALLOW IN Anywhere (v6)
80/tcp(v6)  ALLOW IN Anywhere (v6)
```

## Why this firewall policy?

Default incoming deny means services are not automatically reachable from other machines.

We explicitly opened only:

```text
SSH
HTTP
```

because those were the Linux services we needed.

---

# 48. LINUX LOCAL USER/GROUP PERMISSION EXERCISE

To demonstrate Linux group-based file permissions, we created:

```text
Group: webadmins
User:  webuser
Folder: /srv/intranet-files
```

Commands:

```bash
sudo groupadd webadmins
```

**Purpose:** Creates a local Linux group.

```bash
sudo useradd -m webuser
```

**Purpose:** Creates user `webuser`.

`-m` creates the user's home directory.

```bash
sudo usermod -aG webadmins webuser
```

**Purpose:** Adds `webuser` to supplementary group `webadmins`.

Important flags:

```text
-a = append; do not remove existing supplementary groups
-G = supplementary group list
```

Using `-G` without `-a` can accidentally replace a user's supplementary group membership.

Then:

```bash
sudo mkdir -p /srv/intranet-files
```

**Purpose:** Creates the protected directory.

`-p` allows parent creation if required and avoids an error if the directory path structure already exists.

Then:

```bash
sudo chown root:webadmins /srv/intranet-files
```

**Purpose:** Sets:

```text
Owner = root
Group = webadmins
```

Then:

```bash
sudo chmod 770 /srv/intranet-files
```

## Understanding `770`

Permission digits:

```text
7 = read + write + execute
7 = read + write + execute
0 = no permissions
```

So:

```text
Owner:  rwx
Group:  rwx
Others: ---
```

Verification:

```bash
ls -ld /srv/intranet-files
```

Result:

```text
drwxrwx--- root webadmins /srv/intranet-files
```

There was a typo once:

```bash
ls -ld/srv/intranet-files
```

which returned an invalid option error.

The correct command requires a space:

```bash
ls -ld /srv/intranet-files
```

---

# 49. LINUX PERMISSION TEST

We tested whether `webuser` could create a file:

```bash
sudo -u webuser touch /srv/intranet-files/test.txt
```

## Purpose

`sudo -u webuser` executes the command as `webuser`.

`touch` creates an empty file if it does not exist.

The command succeeded.

Then the current `kali` user tried:

```bash
ls -l /srv/intranet-files
```

Result:

```text
Permission denied
```

This was actually expected because the directory permission was:

```text
770
```

and `kali` was not a member of `webadmins`.

Then:

```bash
sudo -u webuser ls -l /srv/intranet-files
```

successfully displayed:

```text
test.txt
```

This proved:

```text
webuser -> member of webadmins -> directory access works
kali user -> not in webadmins -> access denied
```

Note: the created file itself showed owner/group as `webuser webuser`. We did not configure the setgid bit on the directory, so new files were not forced to inherit `webadmins` as their group. That was not required for this simple exercise.

---

# 50. FINAL VALIDATION — WINDOWS

We performed a final end-to-end check.

## AD + DNS

On DC01:

```cmd
dcdiag /test:dns
```

Result:

```text
DC01 passed test DNS
corp.local passed test DNS
```

Then:

```cmd
nslookup dc01.corp.local
```

Result:

```text
192.168.207.10
```

Then:

```cmd
nslookup corp.local
```

Result:

```text
192.168.207.10
```

Status:

```text
PASS
```

## Domain login + permissions + mapped drive

Final test used Finance user.

Confirmed:

```text
Domain login works
Finance Drive (F:) opens
Other department folders are denied
```

Status:

```text
PASS
```

## Security restriction GPO

Finance user tested:

```text
Command Prompt -> blocked
Control Panel -> blocked
Windows Settings -> blocked
```

Status:

```text
PASS
```

## Account lockout

`test-user`:

```text
5 wrong passwords -> locked
ADUC unlock -> successful
Correct login after unlock -> successful
```

Status:

```text
PASS
```

## Workstation firewall

Opened `wf.msc` with administrator privileges.

Confirmed:

```text
Domain Profile Active
Firewall On
Inbound unmatched traffic Blocked
Outbound unmatched traffic Allowed
Controlled by Group Policy
```

Status:

```text
PASS
```

## Printer

Used:

```text
shell:PrintersFolder
```

Confirmed:

```text
CORP-Office-Printer-Shared on DC01
```

Status:

```text
PASS
```

---

# 51. FINAL VALIDATION — LINUX

## Static IP

```bash
ip addr
```

Confirmed:

```text
eth0 = 192.168.207.20/24
```

## DNS/connectivity

```bash
ping -c 4 dc01.corp.local
```

Confirmed:

```text
dc01.corp.local -> 192.168.207.10
0% packet loss
```

## SSH

```bash
sudo systemctl status ssh --no-pager
```

Confirmed:

```text
active (running)
```

## Apache

```bash
sudo systemctl status apache2 --no-pager
```

Confirmed:

```text
active (running)
```

## UFW

```bash
sudo ufw status verbose
```

Confirmed:

```text
active
22/tcp allowed
80/tcp allowed
default incoming deny
```

## AD trust

```bash
sudo net ads testjoin
```

Confirmed:

```text
Join is OK
```

## AD identity lookup

```bash
getent passwd HR-User
```

returned the HR AD account.

## Linux local permissions

```bash
sudo -u webuser ls -l /srv/intranet-files
```

showed:

```text
test.txt
```

Status:

```text
LINUX FINAL VALIDATION PASS
```

---

# 52. MAIN PROBLEMS WE FACED AND HOW WE SOLVED THEM

## Problem A — Windows Server / VM boot setup difficulties

There were earlier VMware/Windows Server boot issues during the very beginning of the learning process. The final VM was eventually installed and persistent.

Exact individual UEFI/ISO recovery steps are not included here because those belonged to the pre-project installation troubleshooting and the final exact sequence was not preserved in the current project record.

## Problem B — Domain join option unavailable

The Windows 10 client initially had difficulty with the Domain membership option.

Final result:

```text
Windows 10 joined corp.local successfully
```

Exact early GUI fix is not fabricated here because it was not preserved reliably.

## Problem C — HR had Modify but could not create files

Cause:

```text
Share permission only allowed Read
```

Fix:

```text
Everyone -> Read + Change
```

Lesson:

```text
Share and NTFS permissions both affect network access.
```

## Problem D — HR could access Finance/Management

Cause:

```text
Inherited/broad NTFS permissions from parent CompanyData
```

Fix:

```text
Disable inheritance on department folders
Remove unnecessary broad Users permission
Keep department-specific permission
```

Lesson:

```text
Inheritance can accidentally spread access.
```

## Problem E — Duplicate/similarly named users/groups

Examples included:

```text
HRUser
HR-User
FinanceUser
Finance-user
```

Fix/approach:

```cmd
whoami
```

plus careful checking of ADUC object type and group membership.

Later, structured `GG_*` security groups simplified management.

## Problem F — Network view empty

Windows “Network” view was sometimes empty.

Instead of assuming the share was broken, we used the direct UNC path:

```text
\\DC01\CompanyData
```

This worked.

Lesson:

```text
Network Discovery UI is not the same thing as SMB connectivity.
```

## Problem G — Remote Effective Access could not be evaluated

Message:

```text
You do not have permission to evaluate effective access rights for the remote source.
```

Fix:

```text
Perform Effective Access check locally on DC01 as Administrator.
```

## Problem H — Microsoft Print to PDF could not be shared

Message:

```text
Sharing is not supported for this type of printer.
```

Fix:

```text
Use Generic / Text Only dummy printer instead.
```

## Problem I — Printer name conflict

The original test printer already used:

```text
CORP-Office-Printer
```

New printer creation reported a name conflict.

Fix:

```text
CORP-Office-Printer-Shared
```

Then the old incorrect printer was deleted to avoid confusion.

## Problem J — Deployed Printer Connections warning

Message during `gpupdate /force`:

```text
Windows failed to apply the Deployed Printer Connections settings.
```

Resolution:

- remove old legacy printer deployment
- switch to Group Policy Preferences shared printer
- use `Action: Update`
- verify network printer manually
- inspect Group Policy logs
- final `gpupdate /force` completed without the warning

## Problem K — Event Viewer GroupPolicy log Access Denied

Message:

```text
Access is denied (5)
```

Fix:

```text
Run Event Viewer as administrator
```

Then GroupPolicy Operational logs were readable.

## Problem L — Kali package downloads failed

Message:

```text
Temporary failure resolving 'http.kali.org'
```

Cause:

```text
DC01 DNS was resolving internal names but public DNS forwarding was not working.
```

Fix:

```text
DC01 DNS Forwarders -> 8.8.8.8 and 1.1.1.1
```

Then `apt update` succeeded.

## Problem M — SSSD packages unavailable

Errors:

```text
Unable to locate package sssd
Unable to locate package sssd-tools
```

Troubleshooting confirmed the Kali repository definition was correct.

We deliberately did not mix foreign Debian repositories.

Resolution:

```text
Use Samba + Winbind instead of SSSD.
```

## Problem N — `wbinfo -t` failed

Plain:

```bash
wbinfo -t
```

returned:

```text
WBC_ERR_WINBIND_NOT_AVAILABLE
```

But:

```bash
sudo net ads testjoin
```

said:

```text
Join is OK
```

Further checks showed:

```text
Winbind active
Winbind socket exists
NSS configured with winbind
wbinfo -p works
```

Final trust check:

```bash
sudo wbinfo -t
```

succeeded.

Also:

```bash
sudo wbinfo --ping-dc
wbinfo --online-status
```

confirmed active DC/domain connectivity.

## Problem O — Linux directory appeared inaccessible

`webuser` successfully created the test file, but:

```bash
ls -l /srv/intranet-files
```

as the normal `kali` user returned:

```text
Permission denied
```

This was not a failure.

The directory was deliberately:

```text
770 root:webadmins
```

and `kali` was outside the group.

Using:

```bash
sudo -u webuser ls -l /srv/intranet-files
```

proved the intended group-authorized user had access.

---

# 53. COMMAND QUICK REFERENCE — WINDOWS

```cmd
ipconfig
```
Shows basic Windows IP configuration.

```cmd
ipconfig /all
```
Shows detailed network/DNS/DHCP information.

```cmd
ping <IP-or-hostname>
```
Tests IP connectivity.

```cmd
nslookup corp.local
```
Tests DNS resolution for the domain.

```cmd
nslookup dc01.corp.local
```
Tests DNS resolution for DC01.

```cmd
dcdiag /test:dns
```
Runs Domain Controller DNS diagnostics.

```cmd
whoami
```
Displays the currently authenticated Windows identity.

```cmd
gpupdate /force
```
Forces immediate Group Policy refresh.

```cmd
gpresult /r
```
Shows applied GPO summary.

```cmd
gpresult /h C:\GPReport.html
```
Creates a detailed Group Policy HTML report.

```cmd
gpresult /scope user /v
```
Shows verbose user-side Group Policy information.

```text
rsop.msc
```
Opens graphical Resultant Set of Policy.

```text
wf.msc
```
Opens Windows Defender Firewall with Advanced Security.

```text
shell:PrintersFolder
```
Opens the Printers folder directly.

```powershell
ssh kali@192.168.207.20
```
Starts an SSH session from Windows to LINUX01.

UNC/network paths:

```text
\\DC01
\\DC01\CompanyData
\\DC01\CompanyData\HR
\\DC01\CompanyData\Finance
\\DC01\CompanyData\IT
\\DC01\CompanyData\Management
\\DC01\CORP-Office-Printer-Shared
```

---

# 54. COMMAND QUICK REFERENCE — LINUX NETWORKING

```bash
ip addr
```
Shows interfaces and IP addresses.

```bash
ip route
```
Shows routing table/default gateway.

```bash
nmcli connection show
```
Shows NetworkManager connection profiles.

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.addresses 192.168.207.20/24
```
Sets static IPv4 address.

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.gateway 192.168.207.2
```
Sets default gateway.

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.dns 192.168.207.10
```
Sets DC01 as DNS.

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.method manual
```
Switches IPv4 configuration to manual/static.

```bash
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```
Reloads the connection.

```bash
cat /etc/resolv.conf
```
Shows resolver/DNS configuration.

```bash
ping -c 4 192.168.207.10
```
Tests DC01 by IP.

```bash
ping -c 4 dc01.corp.local
```
Tests DNS + connectivity to DC01.

---

# 55. COMMAND QUICK REFERENCE — LINUX SERVICES

```bash
sudo apt update
```
Refreshes package indexes.

```bash
sudo apt install openssh-server -y
```
Installs SSH server.

```bash
sudo systemctl enable ssh
```
Enables SSH at boot.

```bash
sudo systemctl start ssh
```
Starts SSH now.

```bash
sudo systemctl status ssh --no-pager
```
Checks SSH status.

```bash
sudo apt install apache2 -y
```
Installs Apache.

```bash
sudo systemctl enable apache2
```
Enables Apache at boot.

```bash
sudo systemctl start apache2
```
Starts Apache.

```bash
sudo systemctl status apache2 --no-pager
```
Checks Apache status.

```bash
sudo geany /var/www/html/index.html
```
Opens Apache homepage for GUI editing as root.

---

# 56. COMMAND QUICK REFERENCE — KERBEROS / SAMBA / WINBIND

```bash
kinit Administrator@CORP.LOCAL
```
Gets an AD Kerberos ticket for Administrator.

```bash
klist
```
Lists current Kerberos tickets.

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```
Backs up Samba config.

```bash
sudo geany /etc/samba/smb.conf
```
GUI-edit Samba config with root privileges.

```bash
testparm
```
Validates Samba configuration syntax.

```bash
sudo net ads join -U Administrator
```
Joins LINUX01 to the CORP Active Directory domain.

```bash
sudo systemctl enable --now winbind
```
Enables and starts Winbind.

```bash
sudo systemctl status winbind --no-pager
```
Checks Winbind status.

```bash
ls -l /run/samba
```
Checks Samba runtime state/files.

```bash
ls -l /run/samba/winbindd
```
Checks Winbind runtime socket directory.

```bash
sudo journalctl -u winbind -n 50 --no-pager
```
Shows recent Winbind service logs.

```bash
sudo net ads testjoin
```
Validates the machine's AD join/trust.

```bash
grep -E '^(passwd|group):' /etc/nsswitch.conf
```
Checks whether Linux identity lookup includes Winbind.

```bash
wbinfo -p
```
Pings local Winbind daemon.

```bash
sudo wbinfo -t
```
Checks AD trust secret through Winbind.

```bash
sudo wbinfo --ping-dc
```
Checks NETLOGON connectivity to the domain controller.

```bash
wbinfo --online-status
```
Shows Winbind domain online/offline status.

```bash
wbinfo -u
```
Lists AD users through Winbind.

```bash
wbinfo -g
```
Lists AD groups through Winbind.

```bash
getent passwd HR-User
```
Asks Linux NSS to resolve a specific AD user.

```bash
getent passwd
```
Lists passwd identities visible through NSS.

```bash
getent group
```
Lists group identities visible through NSS.

---

# 57. COMMAND QUICK REFERENCE — KALI PACKAGE TROUBLESHOOTING

```bash
apt policy sssd realmd adcli
```
Shows package installed/candidate versions.

```bash
cat /etc/apt/sources.list
```
Checks traditional APT sources file; on this Kali install it did not exist.

```bash
cat /etc/apt/sources.list.d/kali.sources
```
Shows the actual Kali repository configuration.

```bash
sudo apt clean
```
Clears downloaded APT package cache.

```bash
sudo rm -rf /var/lib/apt/lists/*
```
Removes cached package-list metadata so it can be fully downloaded again.

```bash
sudo apt update
```
Rebuilds package indexes.

```bash
apt policy sssd
```
Checks whether SSSD is available.

SSSD still could not be located, so the project moved to Winbind.

---

# 58. COMMAND QUICK REFERENCE — UFW

```bash
sudo apt install ufw -y
```
Installs UFW.

```bash
sudo ufw allow 22/tcp
```
Allows inbound SSH.

```bash
sudo ufw allow 80/tcp
```
Allows inbound HTTP.

```bash
sudo ufw enable
```
Turns UFW on.

```bash
sudo ufw status verbose
```
Shows firewall status, defaults, and rules.

---

# 59. COMMAND QUICK REFERENCE — LINUX USERS/PERMISSIONS

```bash
sudo groupadd webadmins
```
Creates local group.

```bash
sudo useradd -m webuser
```
Creates local user and home directory.

```bash
sudo usermod -aG webadmins webuser
```
Adds user to supplementary group without removing existing groups.

```bash
sudo mkdir -p /srv/intranet-files
```
Creates protected directory.

```bash
sudo chown root:webadmins /srv/intranet-files
```
Sets owner/group.

```bash
sudo chmod 770 /srv/intranet-files
```
Owner/group = rwx, others = none.

```bash
ls -ld /srv/intranet-files
```
Shows directory permission/ownership.

```bash
sudo -u webuser touch /srv/intranet-files/test.txt
```
Creates a test file as webuser.

```bash
sudo -u webuser ls -l /srv/intranet-files
```
Tests directory listing as webuser.

---

# 60. CONCEPT REVISION — THE MOST IMPORTANT THINGS TO REMEMBER

## AD DS

```text
Central identity and domain management
```

## DNS

```text
Translates names to IPs and lets AD clients locate domain services
```

## OU

```text
Organizes users/computers and helps target GPOs
```

## Security group

```text
Collects users so permissions can be assigned to the group instead of individuals
```

## Share permission

```text
Controls access through the network share
```

## NTFS permission

```text
Controls what a user can do to actual files/folders
```

## Inheritance

```text
Copies parent NTFS permission entries to child folders/files
```

## GPO

```text
Centrally configures Windows user/computer settings
```

## User Configuration

```text
Targets users
```

## Computer Configuration

```text
Targets computers
```

## Mapped drive

```text
Gives a user a convenient drive letter pointing to a network share
```

## SSH

```text
Secure remote shell for Linux administration
```

## Apache

```text
HTTP web server
```

## Kerberos

```text
Ticket-based authentication system used heavily by Active Directory
```

## Samba domain member

```text
Allows Linux/Samba to participate in a Windows AD environment
```

## Winbind

```text
Allows Linux/Samba to resolve and work with Windows domain users/groups
```

## NSS

```text
Linux identity lookup framework; nsswitch.conf decides where user/group lookups go
```

## UFW

```text
Simplified Linux firewall management
```

---

# 61. FINAL PROJECT STATUS

Core build:

```text
[PASS] Windows Server 2019
[PASS] DC01 static network configuration
[PASS] AD DS
[PASS] corp.local domain
[PASS] DNS
[PASS] Windows 10 domain join
[PASS] OUs
[PASS] Domain users
[PASS] Department security groups
[PASS] CompanyData file share
[PASS] Share permissions
[PASS] NTFS permissions
[PASS] Inheritance cleanup
[PASS] Department isolation
[PASS] Password policy configured
[PASS] Account lockout configured/tested
[PASS] User security restriction GPO
[PASS] Workstations OU
[PASS] Workstation firewall GPO
[PASS] Department mapped drives
[PASS] Print Server
[PASS] Shared printer
[PASS] Printer GPO troubleshooting/connection
[PASS] LINUX01 static IP
[PASS] Linux DNS through DC01
[PASS] DC01 DNS public forwarders
[PASS] Linux hostname
[PASS] SSH
[PASS] Windows-to-Linux SSH test
[PASS] Apache
[PASS] CORP internal intranet
[PASS] Samba + Winbind AD join
[PASS] AD user/group lookup from Linux
[PASS] UFW
[PASS] Linux local user/group permissions
[PASS] Final Windows validation
[PASS] Final Linux validation
```

The technical build is complete.

---

# 62. THINGS NOT INCLUDED IN THE CORE BUILD

These were intentionally left for later practice rather than making the project unnecessarily large:

```text
Advanced Linux hardening
HTTPS/certificate deployment
Multiple Apache virtual hosts
Monitoring stacks
Advanced automation
Complex Kerberos/LDAP tuning
Advanced sudo mappings for AD groups
Enterprise backup systems
High availability
Second domain controller
Pentesting/attack phase
```

These can be future exercises.

---

# 63. HOW TO REVISE THIS PROJECT AFTER A MONTH

Do not try to memorize every click.

Revise in this order:

## First: understand the architecture

Know:

```text
DC01 = AD DS + DNS + File + Print + GPO
Windows 10 = domain client
LINUX01 = Linux server
corp.local = domain
```

## Second: remember the permission model

```text
User -> Security Group -> NTFS Permission
Share = network gate
Inheritance = parent permission flow
```

## Third: remember GPO targeting

```text
User GPO -> user account / user OU
Computer GPO -> computer object / Workstations OU
```

## Fourth: remember the major commands

Windows:

```text
ipconfig /all
ping
nslookup
dcdiag /test:dns
whoami
gpupdate /force
gpresult /r
wf.msc
```

Linux:

```text
ip addr
ip route
systemctl status
ping
ssh
ufw status verbose
net ads testjoin
wbinfo
getent
```

## Fifth: remember the troubleshooting stories

Interviewers care about this.

Strong examples:

1. User had NTFS Modify but could not write -> Share permission was only Read.
2. HR could modify Finance -> inherited `Users` permissions were too broad.
3. Printer could not be shared -> wrong Microsoft Print to PDF driver -> used Generic/Text.
4. Printer GPO warning -> removed legacy deployment and used GPP Shared Printer.
5. Kali could resolve internal DNS but not internet -> configured DC01 DNS forwarders.
6. SSSD unavailable -> did not break Kali repos -> switched to Samba/Winbind.
7. `wbinfo -t` failed without root -> verified service/socket/join, then `sudo wbinfo -t` succeeded.
8. `kali` could not list protected Linux folder -> correct behavior because it was not in `webadmins`.

If you can explain those eight troubleshooting cases clearly, you understand a large part of the project.

---

# 64. ONE-MINUTE PROJECT EXPLANATION FOR YOURSELF

“I built a small enterprise lab using Windows Server 2019, Windows 10 and a Linux server. I configured DC01 as an Active Directory Domain Controller and DNS server for corp.local, joined a Windows 10 client to the domain, created departmental OUs, users and global security groups, and configured a centralized CompanyData share with NTFS and Share permissions. I fixed inheritance problems so each department could only access its own data. I created Group Policies for password/account lockout, user restrictions, workstation firewall settings and department network-drive mappings. I also configured a shared printer and troubleshot its GPO deployment. On the Linux side I configured a static IP and Windows DNS, SSH, an Apache intranet, UFW, Samba/Winbind Active Directory membership and Linux group-based file permissions. I then performed final validation of DNS, domain login, GPOs, file access, firewall, printer, Linux services and AD integration.”

---

# END OF MASTER REVISION FILE

**Important:** This file intentionally distinguishes verified facts from early setup details whose exact troubleshooting sequence was not preserved. It avoids inventing steps that were not reliably recorded.
