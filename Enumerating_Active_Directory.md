# Enumerating Active Directory


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
