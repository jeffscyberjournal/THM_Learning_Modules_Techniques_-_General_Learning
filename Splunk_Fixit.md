# Splunk Fixit

The Fixit TryHackMe module introduces basic network and system troubleshooting techniques. It teaches you how to investigate and resolve common issues by analyzing logs, checking configurations, validating services, and using troubleshooting tools. The room focuses on developing a structured approach to identifying the root cause of problems, helping learners build practical diagnostic skills that are useful in both IT support and cybersecurity environments.

Skip Task 1 it's just connection information to target.

## Task 2 The Fixit Challenge

**Phase 1: Fixing Event Boundaries**
The core objective of Phase 1 is to fix improper log ingestion within the Fixit app's configuration files. Because Splunk cannot automatically detect where one event ends and the next begins, it is fragmenting single log entries into unreadable pieces. You must manually configure the event boundaries (using props.conf) to make the data cleanly structured and ready for analysis.

using a simple index = main in example we get the following example of events boundaries not configured:
```
>  11/21/25         Australia at: Fri Nov 21 02:18:21 2025
   2:18:21.000 AM   host = tryhackme   source = networks   sourcetype = network_logs

>  11/21/25         [Network-log]: User named Patricia Allen from Development department accessed the resource Cybertees.THM/checkout.html
   2:18:21.000 AM   from the source IP 192.168.1.4 and country
                    host = tryhackme   source = networks   sourcetype = network_logs
...
```
If you look closely at the two events highlighted in the green box:

1. The Second Event: Starts correctly with [Network-log]: User named Patricia Allen... at 2:18:21.000 AM.
2. The First Event: Contains the text Australia at: Fri Nov 21 02:18:21 2025 at that exact same timestamp (2:18:21.000 AM).
  
What Went WrongThe text in the first event actually belongs to the end of a previous log message. Because Splunk does not know where the log entries officially start and stop, it treated that trailing sentence fragment as a brand-new, standalone log entry.

Once you properly apply BREAK_ONLY_BEFORE = \[Network-log\]:, Splunk will stop creating these broken, fragmented events and cleanly merge those lines together.

**Phase 2: Extracting Custom Fields**
The core objective of Phase 2 is to extract specific, meaningful fields from the raw log data to make it searchable. You can do this either by manually updating the Fixit app's configuration files (transforms.conf and props.conf) or by using the Splunk Web UI field extraction wizard.

**Use the sample logs below to help extract the following fields:**

- Username
- Department
- Domain
- URI
- SourceIP
- Country

**Our sample logs:**
```
[Network-log]: User named Emily Clark from Finance department accessed the resource Cybertees.THM/contact.html from the source IP 192.168.1.4 and country 
Japan at: Mon Dec  1 10:13:38 2025
[Network-log]: User named Robert Wilson from HR department accessed the resource Cybertees.THM/signup.html from the source IP 10.0.0.2 and country 
Germany at: Mon Dec  1 10:13:42 2025
[Network-log]: User named Patricia Allen from Finance department accessed the resource Cybertees.THM/checkout.html from the source IP 172.16.0.1 and country 
Mexico at: Mon Dec  1 10:13:48 2025
```
**Phase 3: Analyzing Event Data**
Use Splunk search queries (SPL) on your freshly parsed and structured logs to investigate network activity and answer the final challenge questions.

### Lab Question Answers

**Q1 What is the full path to the Fixit app directory in your instance?**

/opt/splunk/etc/apps/fixit

**Q2 Investigate the inputs.conf configuration file of the Fixit app.**
What is the full path of the network-logs script?

/opt/splunk/etc/apps/fixit/bin/network-logs

**Q3 Which Splunk stanza setting will you use to define the event boundaries for the scenario logs?** 

Hint "This is the hint you’re looking for: This setting tells Splunk to break into a new event before a specified pattern."

BREAK_ONLY_BEFORE

This is the same answer to question 1 in Task 7 from Splunk Data Manipulation module.
```
Earlier in Task 7 states:
...
Let's go over the new stanza line by line.

- [auth_logs] Specifying the sourcetype we set in inputs.conf
- SHOULD_LINEMERGE = true Instructing Splunk that it should combine multiple lines into single events
- BREAK_ONLY_BEFORE = \[Authentication\] Break into a new event before the term [Authentication]
...

**Q1 Which configuration setting ensures Splunk breaks the event boundary before the regex pattern?**  

BREAK_ONLY_BEFORE
```

**Q4 Which regex pattern should be used to define the start of each event?**
Start of each event starts with [Network-logs]: this is pattern of interest here. 

Breaking down of the required pattern:
- \[ Escapes the opening square bracket so regex treats it as text. Network-log: Matches the exact word phrase.
- \] Escapes the closing square bracket
- ^\[ is required at the start of the regex pattern to match only to start of line.
  - ^\[ (Best practice): Matches only when [Network-log] is at the absolute start of a line. This prevents accidental splits if the text appears in the middle of a log message, and it makes Splunk parse data much faster.
  - \[ (Risky): Matches [Network-log] anywhere in the text. If the phrase appears in the middle of an error message, Splunk will accidentally cut your log entry in half.


Answer required is ^\[Network-log\]:

**Q5 After you’ve extracted the relevant fields, what Domain appears in the log data?**

Well its completely overkill at this point but the boundaries of logs are split so, using data maniplation module for splunk I have added the inputs.conf and props.conf to merge into related events:

From default it looks like the following 
```
9/23/26
12:32:32.000 PM	
India at: Wed Sep 23 12:32:32 2026

    host = tryhackme
    source = networks
    sourcetype = network_logs

	9/23/26
12:32:32.000 PM	
[Network-log]: User named Daniel Martin from IT department accessed the resource Cybertees.THM/dashboard.html from the source IP 172.16.0.4 and country 

    host = tryhackme
    source = networks
    sourcetype = network_logs
```

The following is something like what it should look like, bit it 
```
9/23/26
12:56:32.000 PM	
[Network-log]: User named Robert Wilson from Custom department accessed the resource Cybertees.THM/dashboard.html from the source IP 192.168.1.100 and country 
Canada at: Wed Sep 23 12:56:32 2026

    host = tryhackme
    source = networks
    sourcetype = network_logs
```
Given sourcetype = network_logs and trying to include inputs.conf and props.conf file that don't presently exist.

Inputs.conf
```
[script:///opt/splunk/etc/apps/Fixit/bin/network-logs]
index = main
sourcetype = network_logs
host = tryhackme
interval = 5
```
props.conf
```
[network_logs]
SHOULD_LINEMERGE = true
BREAK_ONLY_BEFORE = ^\[Network-log\]:
```
_________.___


How many Username field values exist within the events generated?
This time I had a quick look in the fields recognised, none presently set for **Usernames** or anything like that.

Based on Data Manipulation module and using a similar method. We need to add transforms.conf and add to props.conf file.

transforms.conf
```
[network_user_extract]
REGEX = User named ([A-Za-z]+)\s([A-Za-z]+)\sfrom\s([A-Za-z]+)\sdepartment
FORMAT = firstname::$1 lastname::$2 department::$3
WRITE_META = true
```
Add to props.conf
```
[network_logs]
SHOULD_LINEMERGE = true
BREAK_ONLY_BEFORE = ^\[Network-log\]:
TRANSFORMS-userfields = network_user_extract
```
And a fields.conf file
```
[firstname]
INDEXED = true
 
[lastname]
INDEXED = true

[department]
INDEXED =  true
```

This partially worked it showed 24 first names and 24 last names but its not really addressing usernames, I suspect there are combinations of them altering the value required. 

Instead change transforms.conf to 
```
[username_extract]
REGEX = User named ([A-Za-z]+\s[A-Za-z]+)
FORMAT = Username::$1
WRITE_META = true
```
And for props either add or replace with 
```
[network_logs]
SHOULD_LINEMERGE = true
BREAK_ONLY_BEFORE = ^\[Network-log\]:
TRANSFORMS-userfields = network_user_extract
TRANSFORMS-user = network_user_extract

```
Then restart Splunk to accept the changes.
```
/opt/splunk/bin/splunk restart
```
__

Check
How many URI field values were you able to extract from the available logs?

__

Check
As you begin analyzing the network traffic, how many individual /products pages appear in the data?

_

Check

What is the only URI field value found in the event data without a file extension?

/_____/

Check
Who is the most active User on the network?

______ ______

Check
How many unique IP ranges are represented in the observed network traffic?

_

Check
Which user accessed the secret-document.pdf on your client's server?

_____ ____

Check
