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

Then try xfreerdp failed so did remmina, nmap showed 3389 was open.
```
xfreerdp /v:THMJMP1.za.tryhackme.com /u:za.tryhackme.com\conor.martin /p:Hell2020 /cert:ignore /dynamic-resolution
or
remmina -c "rdp://za.tryhackme.com\\iain.williams@thmjmp1.za.tryhackme.com"
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
