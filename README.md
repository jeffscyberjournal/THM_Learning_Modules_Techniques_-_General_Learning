# THM_Learning_Modules_Techniques_-_General_Learning

# Task 1 Why AD Enumeration Matters

After breaching AD and obtaining your first valid credentials, you gain the ability to authenticate into the domain. Even low‑privileged credentials unlock a huge amount of information about the AD environment. Enumeration is the process of discovering:

- domain structure
- users, groups, and computers
- permissions and misconfigurations
- potential attack paths

In real red‑team operations, enumeration and exploitation happen in cycles:

you enumerate → find a weakness → exploit → enumerate again from your new privilege level.


### What This Module Covers

You will learn several major methods for enumerating AD:

- MMC AD Snap‑ins (GUI tools)
- net.exe commands (Command Prompt)
- PowerShell AD‑RSAT cmdlets
- BloodHound (graph‑based attack path mapping)

This is not an exhaustive list — enumeration depends heavily on what access you gained during the breach.

### Connecting to the Lab Network

To work through the room:

AttackBox

- You’re auto‑connected when launched from the room.
- You must configure DNS manually using the THMDC IP.
- DNS resets every ~3 hours, so you may need to reapply settings.
- Make note of your VPN IP (interface: enumad).

Own Machine / Kali / Other Hosts

- Download the OpenVPN config for the EnumeratingAD network.
- Connect using an OpenVPN client.
- Configure DNS to point to the domain controller (THMDC).
- Verify DNS with nslookup thmdc.za.tryhackme.com.

### Getting Your Initial Credentials

Visit:

http://distributor.za.tryhackme.com/creds

Click Get Credentials to receive your first AD username/password.

These credentials allow:

- RDP access to THMJMP1.za.tryhackme.com
- SSH access to the same jump host

This simulates gaining a foothold inside a real corporate network.

### Checklist (as required by the room)

- Completed the Breaching AD network
- Connected to the Enumerating AD network and configured DNS
- Requested credentials and verified RDP/SSH access to THMJMP1
