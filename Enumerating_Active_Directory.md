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
