# Splunk Fixit

## Task 2:

## Task 2 The Fixit Challenge

**Phase 1: Fixing Event Boundaries**
The first phase of your challenge is to fix the event boundaries for the incoming logs. As seen in the screenshot below and in your Splunk instance, the raw data is being ingested and Splunk cannot determine where one event ends and the next begins, making the data impossible to analyze. Go ahead and jump into the Fixit app's configuration files to get started!

A screenshot of the Splunk Fixit app search page displaying the query index = main, the time range last 24 hours, and the event logs returned from the search.

**Phase 2: Extracting Custom Fields**
The next phase of your challenge requires the extraction of meaningful fields from your client's event data. You can accomplish this by updating the Fixit app’s configuration files or by creating field extractions directly through the Splunk UI.

Use the sample logs below to help extract the following fields

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
Once the log data is flowing in correctly and the fields have been extracted, it's time to begin your analysis. Using the available data, apply your skills to uncover what's happening on the network!

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
Breaking Down the Required Pattern:
- \[ Escapes the opening square bracket so regex treats it as text. Network-log: Matches the exact word phrase.
- \] Escapes the closing square bracket
- ^\[ is required at the start of the regex pattern to match only to start of line.
  - ^\[ (Best practice): Matches only when [Network-log] is at the absolute start of a line. This prevents accidental splits if the text appears in the middle of a log message, and it makes Splunk parse data much faster.
  - \[ (Risky): Matches [Network-log] anywhere in the text. If the phrase appears in the middle of an error message, Splunk will accidentally cut your log entry in half.


_\____________\_:
^\[Network-log\]:
Check

After you’ve extracted the relevant fields, what Domain appears in the log data?

_________.___

Check
How many Username field values exist within the events generated?

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
