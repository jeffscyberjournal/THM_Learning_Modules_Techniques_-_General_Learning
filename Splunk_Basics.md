# 

## Task 1 Is just start machine and 


## Task 2 Log Analysis with Splunk

### 1. Explore the Logs

View all ingested logs:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
index=main
Show more lines

Set time range to All Time.

Available sourcetypes:

web_traffic → Web requests to/from the server
firewall_logs → Allowed/blocked network traffic
Web server IP: 10.10.1.5
2. Initial Triage

Show all web traffic:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
index=main sourcetype=web_traffic
Show more lines

Key observations:

~17k web events
Noticeable traffic spike indicating attack activity
Useful fields:
client_ip
user_agent
path
status
3. Find Peak Traffic Day

Daily event counts:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
index=main sourcetype=web_traffic | timechart span=1d count
Show more lines

Sort highest day first:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
index=main sourcetype=web_traffic
| timechart span=1d count
| sort by count
| reverse
Show more lines

Output: Identifies the attack day with the highest event volume.

4. Identify Suspicious User Agents

Exclude normal browsers:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
index=main sourcetype=web_traffic
user_agent!=*Mozilla*
user_agent!=*Chrome*
user_agent!=*Safari*
user_agent!=*Firefox*
Show more lines

Output: Reveals malicious/non-browser tools and a dominant attacker IP.

Top suspicious IPs:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
sourcetype=web_traffic
user_agent!=*Mozilla*
user_agent!=*Chrome*
user_agent!=*Safari*
user_agent!=*Firefox*
| stats count by client_ip
| sort -count
| head 5
Show more lines

Output: Top attacker IP appears at the top.

5. Trace the Attacker

Replace <ATTACKER_IP> with the identified IP.

Reconnaissance

Search for exposed files:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
sourcetype=web_traffic
client_ip="<ATTACKER_IP>"
AND path IN ("/.env","/*phpinfo*","/.git*")
| table _time path user_agent status
Show more lines

Output:

curl
wget
401 / 403 / 404 responses
Path Traversal & Enumeration

Find traversal and redirect testing:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
sourcetype=web_traffic
client_ip="<ATTACKER_IP>"
AND path="*..*"
OR path="*redirect*"
Show more lines

Count attempts:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
sourcetype=web_traffic
client_ip="<ATTACKER_IP>"
AND path="*..\/..\/*"
OR path="*redirect*"
| stats count by path
Show more lines

Output: Attempts to access sensitive files such as /etc/passwd.

SQL Injection

Search for SQLMap/Havij:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
sourcetype=web_traffic
client_ip="<ATTACKER_IP>"
AND user_agent IN ("*sqlmap*","*Havij*")
| table _time path status
Show more lines

Output:

SQLMap/Havij activity
SLEEP(5) payloads
504 responses suggesting successful time-based SQLi
Data Exfiltration

Look for archive downloads:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
sourcetype=web_traffic
client_ip="<ATTACKER_IP>"
AND path IN ("*backup.zip*","*logs.tar.gz*")
| table _time path user_agent
`
Show more lines

Output:

Downloads via curl, zgrab, etc.
Evidence of data theft
Webshell & Ransomware

Search for webshell execution:

Apache Config
spl isn’t fully supported. Syntax highlighting is based on Apache Config.
sourcetype=web_traffic
client_ip="<ATTACKER_IP>"
AND path IN ("*bunnylock.bin*","*shell.php?cmd=*")
| table _time path user_agent status
Show more lines

Output:

Successful webshell access
Execution of:
Plain Text
/shell.php?cmd=./bunnylock.bin
Show more lines
Confirms RCE and ransomware deployment
6. Confirm C2 Communication

Check outbound connections:

Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
sourcetype=firewall_logs
src_ip="10.10.1.5"
AND dest_ip="<ATTACKER_IP>"
AND action="ALLOWED"
| table _time action protocol src_ip dest_ip dest_port reason
Show more lines

Output:

Outbound connection from compromised server
Suspicious C2 destination port
reason=C2_CONTACT
7. Calculate Exfiltrated Data

Total bytes sent to C2:

Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
sourcetype=firewall_logs
src_ip="10.10.1.5"
AND dest_ip="<ATTACKER_IP>"
AND action="ALLOWED"
| stats sum(bytes_transferred) by src_ip
`
Show more lines

Output: Total volume of data exfiltrated from the server.

Attack Chain Summary
Recon → Probed .env, .git, phpinfo
Enumeration → Tested path traversal and redirects
SQLi → Used SQLMap/Havij and time-based payloads
Exfiltration → Downloaded backups and logs
RCE → Accessed shell.php
Payload Execution → Ran bunnylock.bin
C2 Communication → Outbound connection confirmed in firewall logs
Data Theft → Large volume transferred to attacker infrastructure
Questions to Answer

Use the queries above to determine:

Attacker IP
Peak traffic date (YYYY-MM-DD)
Number of Havij user-agent events
Number of path traversal attempts
Total bytes transferred to the C2 server
