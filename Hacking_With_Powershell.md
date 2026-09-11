# Hacking With Powershell

Note ICMP blocked, ping wont work, use namp -Pn <ip> to check connection is present if for any reason RDP fails.

```
┌──(hacktopuser㉿hacktop)-[/mnt/VBoxShare/openvpn-troubleshooting-2025-infra-upgrades]
└─$ nmap -Pn 10.49.143.197 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-10 21:31 +1000
Nmap scan report for 10.49.143.197
Host is up (0.43s latency).
Not shown: 999 filtered tcp ports (no-response)
PORT     STATE SERVICE
3389/tcp open  ms-wbt-server

Nmap done: 1 IP address (1 host up) scanned in 26.79 seconds
```                                                                  

Connect when sure its present.
```
$ xfreerdp /u:Administrator /p:BHN2UVw0Q /v:10.49.143.197
```

## Task 2: What is Powershell

Very light overview of powershell Nouns and Verbs and commands. 

## Task 3: Basic Powershell Commands

Takeaway: The section teaches how to explore, inspect, filter, sort, and manipulate PowerShell cmdlet output using core discovery commands (Get-Help, Get-Command) and object‑handling commands (Get-Member, Select-Object, Where-Object, Sort-Object).

### Core ideas

**1. Discovering commands**

Get-Help <cmdlet> — shows documentation; add -Examples for usage demonstrations.

Get-Command — lists all installed cmdlets, functions, and aliases.

Supports pattern matching:

Get-Command Verb-*

Get-Command *-Noun

**2. Understanding objects**

PowerShell pipes objects, not text.

Use Get-Member to inspect an object’s methods and properties:

Verb-Noun | Get-Member

Get-Command | Get-Member -MemberType Method

**3. Creating new objects**

Select-Object extracts chosen properties:

Get-ChildItem | Select-Object Mode, Name

Useful flags: -First, -Last, -Unique, -Skip.

**4. Filtering**

here-Object filters objects by property values:

Verb-Noun | Where-Object -Property Status -eq Stopped

Or using script block:

Verb-Noun | Where-Object { $_.Status -eq 'Stopped' }

Operators include -Contains, -EQ, -GT, etc.

**5. Sorting**

Sort-Object sorts piped output:

Verb-Noun | Sort-Object


### Lab Questions

**Q1 Task 3:  What is the location of the file "interesting-file.txt"** based on most information given, trying to keep inline with those commands if it were in same directory the following would work,
But its clearly not.
```
PS C:\Windows\system32> get-childitem | where-object -property name -eq 'interesting-file.txt.txt'
PS C:\Windows\system32> 
```
Recursive search is required and starting path point:
Note: -ErrorAction SilentlyContinue is there because searching the whole C:\ drive produces errors, and without suppressing them, PowerShell will spam your screen and stop the pipeline.
```
get-childitem -path c:\ -include 'interesting-file.txt.txt' -File -recurse -erroraction silentlycontinue


    Directory: C:\Program Files


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        10/3/2019  11:38 PM             23 interesting-file.txt.txt
```
**Q2 Task 3: Contents of interesting-file.txt.txt**:
```
C:\Windows\system32>type "c:\program Files\interesting-file.txt.txt"
notsointerestingcontent
C:\Windows\system32>
```

**Q3 Task 3: How many cmdlets are installed on the system(only cmdlets, not functions and aliases)?:**
```
PS C:\Windows\system32> get-command | where-object -property commandtype -eq cmdlet|measure-object

Count    : 6638
Average  :
Sum      :
Maximum  :
Minimum  :
Property :
```

```
PS C:\Windows\system32> get-filehash "c:\program files\interesting-file.txt.txt" -algorithm MD5

Algorithm       Hash                                                                   Path
---------       ----                                                                   ----
MD5             49A586A2A9456226F8A1B4CEC6FAB329                                       C:\program files\interesting-file.txt.txt
```
**Q4 Task 3: What is the command to get the current working directory?**
```
Get-location
```
**Q5 Task 3:Does the path "C:\Users\Administrator\Documents\Passwords" Exist (Y/N)?**
```
PS C:\Windows\system32> test-path "c:\users\Administrator\documents\password*"
False
```
**Q6 Task3: What command would you use to make a request to a web server?**
```
A common technique used to download files was:
Invoke-WebRequest -Uri "http://example.com:8000/file.txt" -OutFile "file.txt"
The answer is Invoke-WebRequest. 

```
**Q7 Task 3: Base64 decode the file b64.txt on Windows?**
```
PS C:\Windows\system32> get-childitem -path c:\ -include 'b64.txt' -File -recurse -erroraction silentlycontinue

    Directory: C:\Users\Administrator\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        10/3/2019  11:56 PM            432 b64.txt

PS C:\Windows\system32> certutil -decode "C:\Users\Administrator\Desktop\b64.txt" "decoded_hash.txt"
Input Length = 432
Output Length = 323
CertUtil: -decode command completed successfully.
PS C:\Windows\system32> type .\decoded_hash.txt
this is the flag - ihopeyoudidthisonwindows
the rest is garbage
...
```
