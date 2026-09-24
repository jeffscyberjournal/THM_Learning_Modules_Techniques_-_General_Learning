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

Using findings from **Data Maniplation** module for Splunk I have added the inputs.conf and props.conf to merge into related events:

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
[script:///opt/splunk/etc/apps/fixit/bin/network-logs]
index = main
sourcetype = network_logs
host = tryhackme
interval = 5
```
This does not create the index. It tells Splunk:

- Run the script network-logs
- Whatever the script outputs becomes events
- Store those events in the existing main index
- Label them with sourcetype network_logs
- Set the host field to tryhackme


props.conf (fixit_fields points to transforms.conf)
```
[network_logs]
SHOULD_LINEMERGE = true
BREAK_ONLY_BEFORE = ^\[Network-log\]:
REPORT-fixit = fixit_fields
```
transforms.conf
```
[fixit_fields]
REGEX = User named (.*?) from (.*?) department accessed the resource ([^/]+)/([^\s]+) from the source IP (\d{1,3}(?:\.\d{1,3}){3}) and country ([^ ]+) at:
FORMAT = Username::$1 Department::$2 Domain::$3 URI::$4 SourceIP::$5 Country::$6
WRITE_META = true
```
fields.conf
```
[Username]
INDEXED = true
 
[Department]
INDEXED = true
 
[Domain]
INDEXED = true
 
[URI]
INDEXED = true
 
[SourceIP]
INDEXED = true
 
[Country]
INDEXED = true
```
Fields are then added, but you need to select from below the interesting fields, which is below selected fields. Note below the Country, Department, Domain, URI, Username, SourceIP were added after and wont be initially present.
```
Selected Fields

    a Country 12
    a Department 6
    a Domain 1
    a host 1
    a punct 14
    a source 1
    a SourceIP 52
    a sourcetype 1
    a URI 12
    a Username 28

Interesting Fields

    a index 1
    # linecount 2
    a splunk_server 1
    a timestamp 1

10 more fields
Extract New Fields 
```
Selecting Domain only 1 will be listed.

Cybertees.THM




**Q6 How many Username field values exist within the events generated?**

From previous questions answer is 28.


**Q7 How many URI field values were you able to extract from the available logs?**

From URI its 12. These results are necessary in next few questions.
```
URI
...
12 Values, 65.152% of events
...
Events with this field
Top 10 Values 				Count 		% 	 
login.html 					450 	10.063% 	
index.html 					425 	9.504% 	
about.html 					412 	9.213% 	
signup.html 				403 	9.012% 	
contact.html 				400 	8.944% 	
dashboard.html 				395 	8.833% 	
sales/ 						385 	8.609% 	
products/product1.html 		380 	8.497% 	
products/product2.html 		370 	8.274% 	
profile.html 				365 	8.162%
```

**Q8 As you begin analyzing the network traffic, how many individual /products pages appear in the data?**

From previous question answer is 2

**Q9 What is the only URI field value found in the event data without a file extension?**

From second last question:
/sales/

**Q10 Who is the most active User on the network?**

```
Username
...
28 Values, 65.152% of events
...
Events with this field
Top 10 Values 	Count 	% 	 
Robert Wilson 	910 	20.349% 	
Alice Smith 	170 	3.801% 	
Kevin Jackson 	166 	3.712% 	
Nancy Lewis 	152 	3.399% 	
Karen Harris 	151 	3.376% 	
Bob Johnson 	144 	3.22% 	
Alice Johnson 	141 	3.153% 	
Michael Brown 	139 	3.108% 	
Mary Davis 		138 	3.086% 	
Michael Taylor 	138 	3.086%
```


**Q11 How many unique IP ranges are represented in the observed network traffic?**

52 different IP sources but only 3 IP ranges.
```
SourceIP
52 Values, 75.909% of events
...
Events with this field
Top 10 Values 	Count 	% 	 
192.168.1.8 	166 	2.202% 	
192.168.1.5 	164 	2.176% 	
192.168.1.101 	163 	2.163% 	
192.168.0.6 	161 	2.136% 	
192.168.2.1 	161 	2.136% 	
192.168.1.3 	160 	2.123% 	
192.168.0.11 	159 	2.11% 	
10.0.0.4 		158 	2.096% 	
10.0.0.8 		158 	2.096% 	
192.168.0.10 	158 	2.096%
```
Catch is only TOP 10 listed. To show full range use statistics view
```
index=main sourcetype=network_logs
| dedup SourceIP
| table SourceIP
| sort SourceIP"
```

**Q12 Which user accessed the secret-document.pdf on your client's server?**

Selecting the URI will list the files but since 12 requires selecting rarest URI for other 2 listed. But can filter Username connected to the URI using the following: 
```
index=main sourcetype=network_logs "secret-document.pdf"
| table Username
```
Sarah Hall
