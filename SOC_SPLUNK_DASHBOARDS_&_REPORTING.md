# Splunk: Dashboards and Reports — ultra‑concise summary

Splunk collects huge volumes of security logs, and this module teaches you how to turn that raw data into structured, actionable insight.

### What you learn

**Why data organization matters**  
- Clean structure makes threat detection faster and reduces analyst overload.

**Reports**  
- Save recurring SPL searches so routine checks (auth logs, PowerShell activity, anomalies) become automated and repeatable.

**Alerts & rules**  
- Build conditions that trigger notifications for suspicious behaviour, forming Splunk’s core SIEM detection capability.

**Dashboards**  
- Convert saved searches into visual panels (charts, tables, indicators) for quick monitoring and trend analysis.

**Splunk as a SIEM**  
- Aggregates logs, correlates events, generates alerts, and provides visual monitoring across an enterprise environment.




---
## Task 2: Creating Reports for Recurring Searches — concise summary

Splunk’s Search & Reporting app lets analysts run queries across large datasets, but raw event logs (10k+ events in this lab) are too noisy to review manually. Reports solve this by saving useful searches and running them automatically.

**Why reports matter**
- SOCs rely on recurring searches (e.g., every shift, every few hours).
- Scheduled reports reduce load on the search head and ensure consistent visibility.
- They allow analysts to quickly review pre‑processed results instead of re-running heavy searches.

**Building a report**

1. Run a search in Search & Reporting.
Example used:
```
index = vpn_server | stats count by Username | sort - count
```
This counts VPN login events per user; Sarah has the most (86 events).

2. Save the search as a Report.

3. Add a title, description, and choose whether the time picker is active.

4. The report can now be viewed, edited, opened in Search, or added to dashboards.

**Scheduling reports**

- Splunk Enterprise allows scheduled execution (e.g., daily at midnight, every 8 hours).
- You can define:
  - Schedule frequency
  - Time range
  - Priority
  - Scheduling window
- Splunk Free does not support scheduling.

### Lab answers

**Q1 Task2: Head to the Splunk Reports tab and open the Web Connections by Source IP report.
Which Source_IP field value recorded the highest number of events?**
Highest Source_IP: 10.0.0.1

**Q2 Task2: While viewing the report above, click Edit → Open in Search to investigate the query behind the report.
What is the flag value hidden within the search query?**
Hidden flag in query: THM{splunk_report_wizard!}




---
## Task3: Detecting With Alerts and Rules

Splunk alerts allow analysts to detect suspicious activity automatically instead of manually reviewing large log volumes. Although alerts can’t be executed in the free Splunk instance, the room demonstrates how they are built and configured.

**Note:** 
- Due to the limitations of the free Splunk license, we cannot practice setting up alerts on the attached instance. However, you can follow along with the queries and screenshots to get some practice and learn how they're set up.

**Alerts**
- Alerts notify analysts when specific conditions occur (e.g., external IP accessing restricted pages, repeated failed logins).
- Example query to detect external access to /restricted.html:
```
index = web_logs URI = /restricted.html NOT Source_IP IN (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
```
- web_logs on its on its own with "all of time" set. shows 10000 entries.
- Under URI at this point there are only 7 listed, looking further at /restricted.html holds almost same number as the other URI, likely as the file was generated for this purpose, containing 1474, almost same as the other 6. Above shows adding 'URI = /restricted.html' and filtering internal IP ranges out with it, is how the above filter command is why its written that way.
- This filter shows there were no class A,B or C IP in that log linked with the URI.
- if the function is useful can be saved using **save as**, the community edition does not allow storing alerts it only contains in **save as** menu Report, Existing Dashboard, New Dashboard, Event type, the alert option is missing. It should provide options for title and description similar to saving a report.
- Alert configuration demonstrated options also available when saving alert along side title and description:
  - Real-time alert - runs continuous.
  - Per-result trigger - everytime a single event matches the search criteria
  - Send email action - An email will be sent to soc@tryhackme.com when triggered.

```
+---------------------------------------------------------------+
|                     SAVE AS ALERT (Splunk)                    |
+---------------------------------------------------------------+
| Title: Restricted URI Accessed by Outside IP                  |
| Description: Triggers when Source_IP outside expected range    |
|              accesses /restricted.html                        |
|                                                               |
| Permissions: [Private] [Shared in App]                        |
| Alert type:  [Scheduled] -> [Real-time]   (1️⃣)               |
| Expires:     [24 hour(s)]                                     |
|                                                               |
| Trigger Conditions:                                           |
|   Trigger alert when: [Per-Result]        (2️⃣)               |
|   Throttle: [ ]                                              |
|                                                               |
| Trigger Actions:                                              |
|   + Add Actions                                               |
|   When triggered: [Send email] -> soc@tryhackme.com  (3️⃣)    |
|                                                               |
| Buttons: [Cancel] [Save]                                     |
+---------------------------------------------------------------+

Right panel (Email Action Details)
---------------------------------------------------------------
| Send email to: soc@tryhackme.com                              |
| Priority: Normal                                              |
| Subject: Splunk Alert: Restricted URI Access                  |
| Message: A Source_IP outside of expected range attempted      |
|          to access /restricted.html                           |
| Include: [Link to Alert] [Link to Results]                    |
| Buttons: [Cancel] [Save]                                      |
---------------------------------------------------------------
```

**Baseline & Threshold Rules**
- Used to detect abnormal spikes in activity (e.g., excessive 404 errors).
- Baseline query for /payments.html 404 responses:
- Baseline found: ~7.6 404s per hour
```
+---------------------------------------------------------------------+
| SPLUNK SEARCH & REPORTING APP                                       |
+------------------------------------------------------------------------+
| Query:                                                                 |
| index = web_logs uri="/payments.html" status_code=404                  |
| | bin _time span=1h                                                    |
| | stats count AS hits BY _time                                         |
| | eventstats avg(hits) AS avg_hits                                     |
| | eval avg_hits = round(avg_hits, 1)                                   |
+------------------------------------------------------------------------+
| Results Table                                                          |
|------------------------------------------------------------------------|
|_time (One Hour Intervals)| hits (404 Responses)| avg_hits (Average/hr) |
|------------------------------------------------------------------------|
|   2023-01-28 04:00       |       7                |          7.4       |
|   2023-01-28 05:00       |       8                |          7.4       |
|   2023-01-28 06:00       |       9                |          7.4       |
|   2023-01-28 07:00       |       5                |          7.4       |
+------------------------------------------------------------------------+
| Notes:                                                                 |
| - Data grouped by 1-hour intervals                                     |
| - Counts 404 responses per hour                                        |
| - Calculates average (≈7.6/hr) for baseline                            |
+------------------------------------------------------------------------+
```
- Threshold rule example (>11 hits in 1 hour):
```
| where hits > 11
| eval alert = "HIGH 404s: ".hits." in 1h (normal: ~7.6/hr)"
```
Or in full query:
```
index = web_logs URI = /payments.html status_code="404" 
|  bin _time span=1h 
|  stats count AS hits BY _time
| where hits > 11
| eval alert = "HIGH 404s: ".hits." in 1h (normal: ~7.6/hr)" 
```



### Lab Questions

**Q1 Task3: How many Source_IP addresses outside of the expected range accessed /restricted.html?**
using the query: 
```index = web_logs URI = /restricted.html NOT Source_IP IN (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
```
search for source_ip below on left. 
shows only **2** results 

**Q2 Task 3: How many total 404 status_code were recorded at /payments.html?**
The previous query just using the first line only 
```
index = web_logs URI = /payments.html status_code="404"'
```
**Q3 Task 3: Total 404s at /payments.html:**
189, found by just using the first line from above:
```
index = web_logs URI = /payments.html status_code = 404
```
**Q4 Task3: Highest 404 count in any hour:**
16 is answer, the example above gives a complete answer and gives concise example.
