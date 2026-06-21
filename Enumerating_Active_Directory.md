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
