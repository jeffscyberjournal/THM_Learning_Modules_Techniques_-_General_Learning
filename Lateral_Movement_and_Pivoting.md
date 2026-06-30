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
