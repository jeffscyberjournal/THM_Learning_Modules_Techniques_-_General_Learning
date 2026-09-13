# Hacking With Powershell

As per usual ICMP blocked, use namp -Pn <ip> to check connection is present if for any reason RDP fails. This module is quite difficult to connect to and may require a VPN connection to another location in order to complete.

Connect details provided for password, IP and username.
```
$ xfreerdp /u:Administrator /p:BHN2UVw0Q /v:<ip>
```

## Task 2: What is Powershell

Very light overview of powershell Nouns and Verbs and commands. 

## Task 3: Basic Powershell Commands

Takeaway: The section teaches how to explore, inspect, filter, sort, and manipulate PowerShell cmdlet output using core discovery commands (Get-Help, Get-Command) and object‑handling commands (Get-Member, Select-Object, Where-Object, Sort-Object).

### Core ideas

**1. Discovering commands**

- Get-Help <cmdlet> — shows documentation; add -Examples for usage demonstrations.
- Get-Command — lists all installed cmdlets, functions, and aliases.

Supports pattern matching:

- Get-Command Verb-*
- Get-Command *-Noun

**2. Understanding objects**

PowerShell pipes objects, not text.

Use Get-Member to inspect an object’s methods and properties:

- Verb-Noun | Get-Member
- Get-Command | Get-Member -MemberType Method

**3. Creating new objects**

Select-Object extracts chosen properties:

- Get-ChildItem | Select-Object Mode, Name
- Useful flags: -First, -Last, -Unique, -Skip.

**4. Filtering**

here-Object filters objects by property values:

- Verb-Noun | Where-Object -Property Status -eq Stopped

Or using script block:

- Verb-Noun | Where-Object { $_.Status -eq 'Stopped' }
- Operators include -Contains, -EQ, -GT, etc.

**5. Sorting**

Sort-Object sorts piped output:

- Verb-Noun | Sort-Object


### Lab Questions

**Q1 Task 3:  What is the location of the file "interesting-file.txt"** based on most information given, trying to keep inline with those commands if it were in same directory the following would work,
But its clearly not.
```
PS C:\Windows\system32> get-childitem | where-object -property name -eq 'interesting-file.txt.txt'
PS C:\Windows\system32> 
```
Recursive search is required and starting path point:
Note: -ErrorAction SilentlyContinue is there because searching the whole C:\ drive produces errors, and without suppressing them, PowerShell will spam your screen and stop the pipeline.
```
get-childitem -path c:\ -include 'interesting-file.txt.txt' -File -recurse -erroraction silentlycontinue

    Directory: C:\Program Files

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        10/3/2019  11:38 PM             23 interesting-file.txt.txt
```
Location is C:\Program Files folder.

**Q2 Task 3: Contents of interesting-file.txt.txt**:
```
C:\Windows\system32>type "c:\program Files\interesting-file.txt.txt"
notsointerestingcontent
C:\Windows\system32>
```

**Q3 Task 3: How many cmdlets are installed on the system(only cmdlets, not functions and aliases)?:**
```
PS C:\Windows\system32> get-command | where-object -property commandtype -eq cmdlet|measure-object

Count    : 6638
Average  :
Sum      :
Maximum  :
Minimum  :
Property :
```
**Q4 Task 3: Get the MD5 hash of interesting-file.txt**
```
PS C:\Windows\system32> get-filehash "c:\program files\interesting-file.txt.txt" -algorithm MD5

Algorithm    Hash                               Path
---------    ----                               ----
MD5          49A586A2A9456226F8A1B4CEC6FAB329   C:\program files\interesting-file.txt.txt
```
**Q5 Task 3: What is the command to get the current working directory?**
```
Get-location
```
**Q6 Task 3:Does the path "C:\Users\Administrator\Documents\Passwords" Exist (Y/N)?**
```
PS C:\Windows\system32> test-path "c:\users\Administrator\documents\password*"
False
```
**Q7 Task3: What command would you use to make a request to a web server?**
```
A common technique used to download files was:
Invoke-WebRequest -Uri "http://example.com:8000/file.txt" -OutFile "file.txt"
The answer is Invoke-WebRequest. 
```
**Q8 Task 3: Base64 decode the file b64.txt on Windows?**
```
PS C:\Windows\system32> get-childitem -path c:\ -include 'b64.txt' -File -recurse -erroraction silentlycontinue

    Directory: C:\Users\Administrator\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        10/3/2019  11:56 PM            432 b64.txt

PS C:\Windows\system32> certutil -decode "C:\Users\Administrator\Desktop\b64.txt" "decoded_hash.txt"
Input Length = 432
Output Length = 323
CertUtil: -decode command completed successfully.

PS C:\Windows\system32> type .\decoded_hash.txt
this is the flag - ihopeyoudidthisonwindows
the rest is garbage
...
```
## Task 4 Enumeration

**Q1 Task 4: How many users are there on the machine?**
Five users.
```
PS C:\Windows\system32> get-localuser

Name           Enabled Description
----           ------- -----------
Administrator  True    Built-in account for administeri...
DefaultAccount False   A user account managed by the sy...
duck           True
duck2          True
Guest          False   Built-in account for guest acces...
```
**Q2 Task 4: Which local user does this SID(S-1-5-21-1394777289-3961777894-1791813945-501) belong to?**
Guest user.
```
PS C:\Windows\system32> get-localuser | select name,sid

Name           SID
----           ---
Administrator  S-1-5-21-1394777289-3961777894-1791813945-500
DefaultAccount S-1-5-21-1394777289-3961777894-1791813945-503
duck           S-1-5-21-1394777289-3961777894-1791813945-1008
duck2          S-1-5-21-1394777289-3961777894-1791813945-1009
Guest          S-1-5-21-1394777289-3961777894-1791813945-501
```
**Q3 Task 4: How many users have their password required values set to False?**
4 Users.
```
PS C:\Windows\system32> get-localuser | select name,sid,passwordrequired

Name              SID                                             PasswordRequired
----              ---                                             ----------------
Administrator     S-1-5-21-1394777289-3961777894-1791813945-500   True
DefaultAccount    S-1-5-21-1394777289-3961777894-1791813945-503   False
duck              S-1-5-21-1394777289-3961777894-1791813945-1008  False
duck2             S-1-5-21-1394777289-3961777894-1791813945-1009  False
Guest             S-1-5-21-1394777289-3961777894-1791813945-501   False
```
**Q4 Task 4: How many local groups exist?**
24 groups.
```
PS C:\Windows\system32> get-localgroup

Name                                    Description
----                                    -----------
Access Control Assistance Operators     Members of this group can remotely query authorization attributes and permissions.
Administrators                          Administrators have complete and unrestricted access to the computer.
Backup Operators                        Backup Operators can override security restrictions for the sole purpose of backing up or restoring files.
Certificate Service DCOM Access         Members of this group are allowed to connect to Certification Authorities using DCOM.
Cryptographic Operators                 Members are authorized to perform cryptographic operations.
Distributed COM Users                   Members are allowed to launch, activate and use Distributed COM objects on this computer.
Event Log Readers                       Members of this group can read event logs from the local machine.
Guests                                  Guests have the same access as members of the Users group by default.
Hyper-V Administrators                  Members of this group have complete and unrestricted access to all Hyper-V features.
IIS_IUSRS                               Built-in group used by Internet Information Services.
Network Configuration Operators         Members of this group can have some administrative privileges to manage network configuration.
Performance Log Users                   Members of this group may schedule logging of performance counters and collect logs.
Performance Monitor Users               Members of this group can access performance counter data locally and remotely.
Power Users                             Power Users are included for backward compatibility and possess limited administrative rights.
Print Operators                         Members can administer printers installed on domain controllers.
RDS Endpoint Servers                    Servers in this group run virtual machines and host sessions where users connect remotely.
RDS Management Servers                  Servers in this group can perform routine administrative actions on Remote Desktop Services.
RDS Remote Access Servers               Servers in this group enable users of RemoteApp programs and personal virtual desktops.
Remote Desktop Users                    Members in this group are granted the right to log on remotely.
Remote Management Users                 Members of this group can access WMI resources over management protocols.
Replicator                              Supports file replication in a domain.
Storage Replica Administrators          Members of this group have complete and unrestricted access to all Storage Replica features.
System Managed Accounts Group           Members of this group are managed by the system.
Users                                   Users are prevented from making accidental or intentional system-wide changes.

or

PS C:\Windows\system32> get-localgroup | measure
Count    : 24
```
**Q5 Task 4: What command did you use to get the ip address info?**
Get-NetIPAddress similar to ipconfig
```
PS C:\Windows\system32> Get-NetIPAddress

IPAddress         : fe80::5efe:10.64.138.140%2
InterfaceIndex    : 2
InterfaceAlias    : isatap.ec2.internal
AddressFamily     : IPv6
Type              : Unicast
PrefixLength      : 128
PrefixOrigin      : WellKnown
SuffixOrigin      : Link
AddressState      : Deprecated
ValidLifetime     : Infinite ([TimeSpan]::MaxValue)
PreferredLifetime : Infinite ([TimeSpan]::MaxValue)
SkipAsSource      : False
PolicyStore       : ActiveStore
IPAddress         : fe80::246f:7a4f:f5bf:7573%7
InterfaceIndex    : 7
InterfaceAlias    : Local Area Connection* 3
AddressFamily     : IPv6
Type              : Unicast
PrefixLength      : 64
PrefixOrigin      : WellKnown
SuffixOrigin      : Link
AddressState      : Preferred
ValidLifetime     : Infinite ([TimeSpan]::MaxValue)
PreferredLifetime : Infinite ([TimeSpan]::MaxValue)
SkipAsSource      : False
PolicyStore       : ActiveStore
...
```
**Q6 Task 4: How many ports are listed as listening?**
20, avoiding netstat the best options is get-nettcpconnection -state listen
```
 get-nettcpconnection -state listen

LocalAddress    LocalPort  RemoteAddress  RemotePort  State
-------------   ---------  -------------- ----------- -----
::              49673      ::             0           Listen
::              49670      ::             0           Listen
::              49667      ::             0           Listen
::              49666      ::             0           Listen
::              49664      ::             0           Listen
::              47001      ::             0           Listen
::              5985       ::             0           Listen
::              3389       ::             0           Listen
::              445        ::             0           Listen
::              135        ::             0           Listen
0.0.0.0         49673      0.0.0.0        0           Listen
0.0.0.0         49670      0.0.0.0        0           Listen
0.0.0.0         49667      0.0.0.0        0           Listen
0.0.0.0         49666      0.0.0.0        0           Listen
0.0.0.0         49664      0.0.0.0        0           Listen
0.0.0.0         445        0.0.0.0        0           Listen
0.0.0.0         3389       0.0.0.0        0           Listen
10.64.138.140   139        0.0.0.0        0           Listen
0.0.0.0         135        0.0.0.0        0           Listen
0.0.0.0         5985       0.0.0.0        0           Listen

PS C:\Windows\system32> get-nettcpconnection -state listen | measure

Count    : 20
Average  :
Sum      :
Maximum  :
Minimum  :
Property :
```
**Q7 Task 4: What is the remote address of the local port listening on port 445?**
::, localhost is the IP set for remote port listening.

**Q8 Task 4: How many patches have been applied?**
20 seems to be popular here!
```
PS C:\Windows\system32> get-hotfix

Source        Description      HotFixID      InstalledBy          InstalledOn
------        -----------      --------      -----------          -----------
EC2AMAZ-5M... Update           KB3176936                          10/18/2016 12:00:00 AM
EC2AMAZ-5M... Update           KB3186568     NT AUTHORITY\SYSTEM  6/15/2017 12:00:00 AM
EC2AMAZ-5M... Update           KB3192137     NT AUTHORITY\SYSTEM  9/12/2016 12:00:00 AM
EC2AMAZ-5M... Update           KB3199209     NT AUTHORITY\SYSTEM  10/18/2016 12:00:00 AM
EC2AMAZ-5M... Update           KB3199986     EC2AMAZ-5M13VM2\A... 11/15/2016 12:00:00 AM
EC2AMAZ-5M... Update           KB4013418     EC2AMAZ-5M13VM2\A... 3/16/2017 12:00:00 AM
EC2AMAZ-5M... Update           KB4023834     EC2AMAZ-5M13VM2\A... 6/15/2017 12:00:00 AM
EC2AMAZ-5M... Update           KB4035631     NT AUTHORITY\SYSTEM  8/9/2017 12:00:00 AM
EC2AMAZ-5M... Update           KB4049065     NT AUTHORITY\SYSTEM  11/17/2017 12:00:00 AM
EC2AMAZ-5M... Update           KB4089510     NT AUTHORITY\SYSTEM  3/24/2018 12:00:00 AM
EC2AMAZ-5M... Update           KB4091664     NT AUTHORITY\SYSTEM  1/10/2019 12:00:00 AM
EC2AMAZ-5M... Update           KB4093137     NT AUTHORITY\SYSTEM  4/11/2018 12:00:00 AM
EC2AMAZ-5M... Update           KB4132216     NT AUTHORITY\SYSTEM  6/13/2018 12:00:00 AM
EC2AMAZ-5M... Security Update  KB4465659     NT AUTHORITY\SYSTEM  11/19/2018 12:00:00 AM
EC2AMAZ-5M... Security Update  KB4485447     NT AUTHORITY\SYSTEM  2/13/2019 12:00:00 AM
EC2AMAZ-5M... Security Update  KB4498947     NT AUTHORITY\SYSTEM  5/15/2019 12:00:00 AM
EC2AMAZ-5M... Security Update  KB4503537     NT AUTHORITY\SYSTEM  6/12/2019 12:00:00 AM
EC2AMAZ-5M... Security Update  KB4509091     NT AUTHORITY\SYSTEM  9/6/2019 12:00:00 AM
EC2AMAZ-5M... Security Update  KB4512574     NT AUTHORITY\SYSTEM  9/11/2019 12:00:00 AM
EC2AMAZ-5M... Security Update  KB4516044     NT AUTHORITY\SYSTEM  9/11/2019 12:00:00 AM

PS C:\Windows\system32> get-hotfix | measure

Count    : 20
```
**Q8 Task 4: Find the contents of a backup file?**
backpassflag was found after several failed iterations of file name searches.  Some programs make backups with filename.extention.bak (where extention depending on application could be anything like json,txt,ini...) or the other way around, like file.bak.txt.
```
Get-ChildItem -Path C:\ -Recurse -Include *backup* -ErrorAction SilentlyContinue

Get-ChildItem -Path C:\ -Recurse -Filter *.bak -ErrorAction SilentlyContinue
nothing found here.
Get-ChildItem -Path C:\ -Recurse -Filter *.txt -ErrorAction SilentlyContinu
Extensive file list. 
PS C:\windows\winsxs> get-childitem -path C:\ -recurse -filter *.bak* -ErrorAction SilentlyContinue


    Directory: C:\Program Files (x86)\Internet Explorer


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        10/4/2019  12:42 AM             12 passwords.bak.txt


PS C:\windows\winsxs> get-content "C:\Program Files (x86)\Internet Explorer\passwords.bak.txt"
backpassflag
```
**Q9 Task 4: Search for all files containing API_KEY
Tried a few things starting with file names and only one showed up multiple times
```
PS C:\windows\winsxs> get-childitem -path C:\ -recurse -filter *API_KEY* -ErrorAction SilentlyContinue


    Directory: C:\Windows\System32\migwiz\dlmanifests


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        7/16/2016   1:18 PM           4533 dpapi_keys-DL.man
... (several same file name in different locations)
```
In the file itself several options but the others took way longer even though more direct. This got the answer without wondering if the process is frozen.
```
PS C:\windows\winsxs> get-childitem c:\* -recurse | select-string "API_KEY"
...hundreds of lines omitted
C:\Users\Public\Music\config.xml:1:API_KEY=fakekey123
```
**Q10 Task 4: What command do you do to list all the running processes?**

Get-process


