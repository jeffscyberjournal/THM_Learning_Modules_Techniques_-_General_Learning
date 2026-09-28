# Splunk 2


## Task 1 

**BOTSv2 Dataset:**

BOTSv2 is a realistic Splunk security dataset containing Windows endpoint logs, Sysmon events, firewall data, network traffic, and IDS alerts. It is used to practice SOC investigations, threat hunting, incident response, and SPL query analysis within Splunk.

This one run from attack box or VM.

## Task 2 Dive into the Data

In this scenario, you take the role of Alice Bluebird, a security analyst assisting Frothly with investigating security incidents using Splunk.

### What Data Is Available?
 
The `metadata` command provides a high-level summary of the data available within an index, similar to Splunk's Data Summary view. In this example, it is used to list the available sourcetypes in the `botsv2` index, along with their event counts and the first and last times data was observed.
 
The timestamp fields returned by `metadata` are stored as Unix epoch values. The `eval` command with `strftime()` is used to convert these timestamps into a human-readable date and time format.
 
```spl
| metadata type=sourcetypes index=botsv2
| eval firstTime=strftime(firstTime,"%Y-%m-%d %H:%M:%S")
| eval lastTime=strftime(lastTime,"%Y-%m-%d %H:%M:%S")
| eval recentTime=strftime(recentTime,"%Y-%m-%d %H:%M:%S")
| sort - totalCount
```

## Task 3 100 Series Questions (BOTSv2) - Summary

Scenario: Investigate Amber Turing's communications with a potential competitor and identify the website visited, emails exchanged, contacts involved, and files sent.

### Q1: Identify the Competitor Website

1. Find Amber's IP Address

Search for references to Amber:
```
index="botsv2" amber
```

Focus on PAN traffic logs:
```
index="botsv2" sourcetype="pan:traffic"
```
Output: Amber's IP address.

**NOTE:** `pan:traffic` is the Palo Alto Networks firewall traffic log sourcetype. It contains network connection information recorded by a Palo Alto Networks firewall, including source and destination IP addresses, ports, applications, actions, and web traffic details.

Those logs are ingested into Splunk as:
```
sourcetype=pan:traffic
```



### 2. Review Amber's HTTP Activity

Replace IPADDR with Amber's IP address:
```
index="botsv2" IPADDR sourcetype="stream:HTTP"
```
Output: HTTP requests made by Amber.

### 3. List Unique Websites Visited

Remove duplicates and display sites:
```index="botsv2" IPADDR sourcetype="stream:HTTP"
| dedup site
| table site
```
Output:
```
sitewebsite1
website2
...
```
The competitor's domain should stand out based on Frothly's industry.

Alternative: Filter by Industry

Use an industry-related keyword:

```
index="botsv2" IPADDR sourcetype="stream:HTTP" *INDUSTRY*
| dedup site
| table site
```
Output: Typically narrows results to the competitor website.

### Q2-Q7: Investigate Competitor Communications

### 4. Focus on Traffic to the Competitor Website
```
index="botsv2" IPADDR sourcetype="stream:HTTP" COMPETITOR_WEBSITE
```
Output: HTTP activity between Amber and the competitor's website.

Use table to extract useful fields:

```
index="botsv2" IPADDR sourcetype="stream:HTTP" COMPETITOR_WEBSITE
| table uri uri_path site
```
Output: URLs visited, including pages containing executive contact information.

### 5. Find Amber's Email Address

Identify Amber's email from previous results and use it for SMTP searches.

### 6. Investigate Email Communications
```
index="botsv2" sourcetype="stream:smtp" AMBERS_EMAIL COMPETITOR_WEBSITE
```
Output: Email exchanges between Amber and competitor personnel.

Useful fields may include:
```
| table sender recipient subject attachment
```
Output:

sender	recipient	subject	attachment


### Lab Question Answers

**Q1 Amber Turing was hoping for Frothly to be acquired by a potential competitor which fell through, but visited their website to find contact information for their executive team. What is the website domain that she visited?**

search using 
```
index="botsv2"  amber  sourcetype="pan:traffic"
```
There is only 1 IP for src_ip or client_ip likely related to amber: 10.0.2.101

Using that with further search:

index="botsv2" 10.0.2.101 sourcetype="stream:HTTP" *beer*

leads to just one site: www.berkbeer.com

**Q2 Amber found the executive contact information and sent him an email. What image file displayed the executive's contact information? Answer example: /path/image.ext**

Using the website found:

index="botsv2" 10.0.2.101 sourcetype="stream:HTTP" www.berkbeer.com
Then selecting the uri_path field in interesting fields there are 12 files. One most likely labelled ceoberk.png is likely the image with the email address.
/images/ceoberk.png

```
Top 10 Values             Count 	% 	 
/                         1       8.333% 	
/favicon.ico              1       8.333% 	
/images/bgimg01.jpg       1       8.333% 	
/images/ceoberk.png       1       8.333% 	
...
```

**Q3 What is the CEO's name? Provide the first and last name.**


**Q4 What is the CEO's email address?**


**Q5 After the initial contact with the CEO, Amber contacted another employee at this competitor. What is that employee's email address?**



**Q6 What is the name of the file attachment that Amber sent to a contact at the competitor?**



**Q7 What is Amber's personal email address?**
