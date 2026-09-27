# Splunk 2


## Task 1 

**BOTSv2 Dataset:**

BOTSv2 is a realistic Splunk security dataset containing Windows endpoint logs, Sysmon events, firewall data, network traffic, and IDS alerts. It is used to practice SOC investigations, threat hunting, incident response, and SPL query analysis within Splunk.

This one run from attack box or VM.

## Task 2 Dive into the Data

In this scenario, you take the role of Alice Bluebird, a security analyst assisting Frothly with investigating security incidents using Splunk.

**What Kinds of Events Do We Have?**

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
