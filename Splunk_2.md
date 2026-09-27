# Splunk 2


## Task 1 

**BOTSv2 Dataset:**

BOTSv2 is a realistic Splunk security dataset containing Windows endpoint logs, Sysmon events, firewall data, network traffic, and IDS alerts. It is used to practice SOC investigations, threat hunting, incident response, and SPL query analysis within Splunk.

This one run from attack box or VM.

## Task 2 Dive into the Data

In this scenario, you take the role of Alice Bluebird, a security analyst assisting Frothly with investigating security incidents using Splunk.

### What Kinds of Events Do We Have?

Use the metadata command to quickly discover the data available in an index. It provides information similar to Splunk's Data Summary, including:

- Available sourcetypes
- Event counts
- First and last observed events

**Note:** Timestamps are returned in **epoch format**, so eval and strftime() are used to convert them into a readable date/time format.

```
| metadata type=sourcetypes index=botsv2
| eval firstTime=strftime(firstTime,"%Y-%m-%d %H:%M:%S")
| eval lastTime=strftime(lastTime,"%Y-%m-%d %H:%M:%S")
| eval recentTime=strftime(recentTime,"%Y-%m-%d %H:%M:%S")
| sort - totalCount
```
**Purpose**

This query inventories the **BOTSv2** dataset by listing all available sourcetypes, their event counts, and the first/last times data was seen, helping analysts understand available data before beginning a hunt.

**Why does THM start with metadata in the search command here?*

The room is teaching a standard investigation workflow:

1. What data do I have? ← metadata
2. Which sourcetypes matter? ← metadata type=sourcetypes
3. Search those sourcetypes ← index=botsv2 ...
4. Investigate

This just raises confusion as index is a sub category of one or more of the sub categories, and basically requires digging to find which ones are available. 

But in a real unknown environment, there's a missing step:

Real Workflow
-------------
1. What indexes exist?
2. What data do I have in those indexes?
3. Which sourcetypes matter?
4. Search events
5. Investigate

**Running | metadata on its own will raise error it must have a type,** there are only three valid type= values:
**Note: metadata must also start with pipe also!**
```
| metadata type=hosts
| metadata type=sources
| metadata type=sourcetypes
```

### Why metadata Exists

Imagine being dropped into an unfamiliar Splunk environment. You may not know whether HTTP, DNS, Sysmon, email, firewall, or IDS logs are available.

So you first run:
```
| metadata type=sourcetypes
```
to get a high-level inventory of available log categories (sourcetypes). From the results, you might discover:
```
stream:http
stream:smtp
pan:traffic
suricata
XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
```

You can then investigate specific data sources:
```
index=botsv2 sourcetype="stream:http"
```
or
```
index=botsv2 sourcetype="stream:smtp"
```

**Limitation**

metadata only inventories hosts, sources, and sourcetypes. It does not discover indexes, show events, expose fields, or tell you which index a sourcetype belongs to. In a truly unknown environment, index discovery must usually happen separately.

**For this THM room**

You can realistically skip metadata and start with:
```
index=botsv2
...
```
because the exercise already tells you the relevant sourcetypes (pan:traffic, stream:HTTP, stream:smtp).

In this context, the metadata section is mainly included to demonstrate a Splunk data-discovery technique rather than to help solve the challenge.

## Task 3 100 Series Questions (BOTSv2) - Summary

Scenario: Investigate Amber Turing's communications with a potential competitor and identify the website visited, emails exchanged, contacts involved, and files sent.

Q1: Identify the Competitor Website
1. Find Amber's IP Address

Search for references to Amber:

Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
index="botsv2" amber
Show more lines

Focus on PAN traffic logs:

Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
index="botsv2" sourcetype="pan:traffic"
Show more lines

Output: Amber's IP address.

2. Review Amber's HTTP Activity

Replace IPADDR with Amber's IP address:

Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
index="botsv2" IPADDR sourcetype="stream:HTTP"
Show more lines

Output: HTTP requests made by Amber.

3. List Unique Websites Visited

Remove duplicates and display sites:

Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
index="botsv2" IPADDR sourcetype="stream:HTTP"
| dedup site
| table site
Show more lines

Output:

sitewebsite1
website2
...

The competitor's domain should stand out based on Frothly's industry.

Alternative: Filter by Industry

Use an industry-related keyword:

Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
index="botsv2" IPADDR sourcetype="stream:HTTP" *INDUSTRY*
| dedup site
| table site
Show more lines

Output: Typically narrows results to the competitor website.

Q2-Q7: Investigate Competitor Communications
4. Focus on Traffic to the Competitor Website
Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
index="botsv2" IPADDR sourcetype="stream:HTTP" COMPETITOR_WEBSITE
Show more lines

Output: HTTP activity between Amber and the competitor's website.

Use table to extract useful fields:

Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
index="botsv2" IPADDR sourcetype="stream:HTTP" COMPETITOR_WEBSITE
| table uri uri_path site
Show more lines

Output: URLs visited, including pages containing executive contact information.

5. Find Amber's Email Address

Identify Amber's email from previous results and use it for SMTP searches.

6. Investigate Email Communications
Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
index="botsv2" sourcetype="stream:smtp" AMBERS_EMAIL COMPETITOR_WEBSITE
Show more lines

Output: Email exchanges between Amber and competitor personnel.

Useful fields may include:

Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
| table sender recipient subject attachment
Show more lines

Output:

sender	recipient	subject	attachment
Answers Covered by These Searches

From the HTTP and SMTP results, determine:

Competitor website domain
Image file containing executive contact information
CEO's full name
CEO's email address
Second employee's email address
File attachment sent by Amber
Amber's personal email address
