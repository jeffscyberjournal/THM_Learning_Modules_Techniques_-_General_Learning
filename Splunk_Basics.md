# 

## Task 1 Is just start machine and 


## Task 2 Log Analysis with Splunk
- No VM required, states "you can connect to the Splunk SIEM by visiting https://**-**-**-**.reverse-proxy.cell-prod-????.vm.tryhackme.com in your browser." 

### 1. Explore the Logs

View all ingested logs:
- First select the "Search and Reporting" link to begin.
- Start with index=main
- Set time range to All Time.

There should be 2 available sourcetypes under "Selected Fields":

web_traffic → Web requests to/from the server
firewall_logs → Allowed/blocked network traffic

### 2. Initial Triage

Inially select Web_traffic this change search bar to:
```
index=main sourcetype=web_traffic
```
**Key observations:**

- Top is main search parameter space below is 4 tabs.
  - Events (7,876) this is main area for focusing on fields to narrow down searches.
  - Patterns this shows patterns associated with current search parameters.
  - Statistics - this if stats included in search parameters may allow statistics to be presented in findings.
  - Visualization - usually requires some form of stats, chart or timechart function to present data.

For now focusing on events: 
~17k web events

**Useful fields:**
**client_ip**
 Identifies the source IP address making requests to the web server. This field is useful for identifying suspicious hosts, determining which IP generated the most activity, spotting scanning or attack behavior, and tracking an attacker's actions throughout the intrusion.

**user_agent**
 Identifies the application, browser, script, or tool making the request. This field helps distinguish legitimate users (Chrome, Firefox, Safari, Mozilla) from automated attack tools such as curl, wget, sqlmap, Havij, or custom scripts. Unusual user agents are often strong indicators of malicious activity.

**path**
 Shows the URI or resource requested on the web server. This field reveals what the client attempted to access and can expose reconnaissance, exploitation, and post-exploitation activity. Examples include requests for sensitive files (/.env, /.git), path traversal attempts (../../etc/passwd), SQL injection payloads, webshell access (shell.php), and ransomware staging files (bunnylock.bin).

**status**
 Contains the HTTP response code returned by the server. This helps determine whether an attack succeeded or failed:

200 = Request succeeded
301/302 = Redirect
401 = Unauthorized
403 = Forbidden
404 = Resource not found
500 = Server error
504 = Gateway timeout (often seen during time-based SQL injection testing)

### 3. Find Peak Traffic Day

Daily event counts: (looking at sourcetype web_traffic for example)

Run search command:

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
