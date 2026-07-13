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
  - (1) Real-time alert - runs continuous.
  - (2) Per-result trigger - everytime a single event matches the search criteria
  - (3) Send email action - An email will be sent to soc@tryhackme.com when triggered.

```
+---------------------------------------------------------------+
|                     SAVE AS ALERT (Splunk)                    |
+---------------------------------------------------------------+
| Title: Restricted URI Accessed by Outside IP                  |
| Description: Triggers when Source_IP outside expected range   |
|              accesses /restricted.html                        |
|                                                               |
| Permissions: [Private] [Shared in App]                        |
| Alert type:  [Scheduled] -> [Real-time]   (1)               |
| Expires:     [24 hour(s)]                                     |
|                                                               |
| Trigger Conditions:                                           |
|   Trigger alert when: [Per-Result]        (2)               |
|   Throttle: [ ]                                               |
|                                                               |
| Trigger Actions:                                              |
|   + Add Actions                                               |
|   When triggered: [Send email] -> soc@tryhackme.com  (3)    |
|                                                               |
| Buttons: [Cancel] [Save]                                      |
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


---
### Lab Questions

**Q1 Task3: How many Source_IP addresses outside of the expected range accessed /restricted.html?**
using the query: 
```
index = web_logs URI = /restricted.html NOT Source_IP IN (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
```
search for source_ip below on left. Listing 552 events out of the files ~10000.
```
Source_IP

2 Values, 100% of even

Values	    Count	    % 
192.0.2.1	  282	     51.087%	
203.0.113.1	270	     48.913%
```
shows only **2** source IP results. 

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


--- 
## Task 4 Creating Dashboards for Summarizing Results

Splunk dashboards provide fast, visual summaries of important event data. They allow analysts to spot trends, spikes, dips, and anomalies without manually running searches.
```
+--------------------------------------------------------------------------------+
| SEARCH | ANALYTICS | DATASETS | REPORTS | ALERTS | [DASHBOARDS] |              |
|--------------------------------------------------------------------------------|
| Dashboards                                                                     |
| Dashboards include interactive visualizations and input controls for data.     |
|--------------------------------------------------------------------------------|
| [Create New Dashboard]                                                         |
|--------------------------------------------------------------------------------|
| Latest Resources:                                                              |
| - Explore Dashboard Studio                                                     |
| - Intro to Dashboard Studio                                                    |
| - Intro to Classic Dashboards                                                  |
|--------------------------------------------------------------------------------|
| Available Dashboards                                                           |
|--------------------------------------------------------------------------------|
| Title                                         | Actions | Owner  | App   | Type|
|--------------------------------------------------------------------------------|
| Integrity Check of Installed PKGs             | Edit  | nobody | search|Classic|
| Java Details Dashboard                        | Edit  | nobody | search|Classic|
| Query Details                                 | Edit  | nobody | search|Classic|
| Organized Scheduled Searches, Reports, Alerts | Edit  | nobody | search|Classic|
| Web Logs Overview                             | Edit  | admin  | search|Classic|
+--------------------------------------------------------------------------------+
| [Search & Reporting] button (top right)                                        |
+--------------------------------------------------------------------------------+
```
Aside from choosing an existing dashboard, you have the option to create a new one in which you must assign a title, provide an optional description, adjust the permissions, and decide whether to use Classic Dashboards or Dashboard Studio.

Dashboard Studio is Splunk's newer dashboard builder, designed to provide users with greater customization options in exchange for a complex learning curve. Classic Dashboards, on the other hand, remain the most commonly encountered format and fully support all standard visualizations. In this room, we will cover Classic Dashboards.

```
+---------------------------------------------------------------+
|                   CREATE NEW DASHBOARD                        |
+---------------------------------------------------------------+
| Dashboard Title: [Sample Dashboard Title]                     |
| Edit ID: sample_dashboard_title                               |
|---------------------------------------------------------------|
| Description: [A Dashboard to Visualize Event Data]            |
|---------------------------------------------------------------|
| Permissions: [Private]                                        |
|---------------------------------------------------------------|
| How do you want to build your dashboard?                      |
|                                                               |
| [Classic Dashboards]  - The traditional Splunk dashboard builder
| [Dashboard Studio]    - A new builder for customizable dashboards
|---------------------------------------------------------------|
| Buttons: [Cancel] [Create]                                    |
+---------------------------------------------------------------+
```

Back in the dashboards tab, go ahead and select the Web Logs Overview dashboard. Currently, our dashboard has a single panel that visualizes the count of events over time from the web_logs index. This dashboard is helpful, but we can further enhance it by adding more visualizations to better understand the data from our web server. Go ahead and click Edit, then + Add Panel as highlighted in the screenshot below, so we can expand our current dashboard.
```
+---------------------------------------------------------------+
|                    EDIT DASHBOARD - SPLUNK                    |
+---------------------------------------------------------------+
| [UI] [Source] [Add Panel] [Add Input] [Dark Theme]            |
|---------------------------------------------------------------|
| Web Logs Overview                                             |
| This dashboard provides an overview of data from the web_logs.|
|---------------------------------------------------------------|
| No title                                                      |
|---------------------------------------------------------------|
| Time Chart By Hour                                            |
|---------------------------------------------------------------|
| >>> Time chart is here by time <<<                            |
|---------------------------------------------------------------|
| Buttons: [Cancel] [Save as...] [Save]                         |
+---------------------------------------------------------------+
```

Perhaps, we want to build a pie chart that shows the event count for the URI field in the web_logs index, helping us visualize how many times each URI was accessed within our available events. In the Add Panel pop-out, choose:

New → Pie Chart (1.)
Set the time to All time (2.)
Enter a Content Title (3. something like URI Distribution)
Enter the search string (4.)(index = web_logs | stats count by URI | sort - count)
Click Add to Dashboard
```
+---------------------------------------------------------------------------+
|                               ADD PANEL                                   |
+---------------------------------------------------------------------------+
| New (15)                                                                  |
|   - Events                                                                |
|   - Statistics Table                                                      |
|   - Line Chart                                                            |
|   - Area Chart                                                            |
|   - Column Chart                                                          |
|   - Bar Chart                                                             |
|   - Pie Chart   (1. Selected)                                             |
|   - Scatter Chart                                                         |
+---------------------------------------------------------------------------+
|                               NEW PIE CHART                               |
+---------------------------------------------------------------------------+
| Time Range: [All time] (2)                                                |
| Content Title: [URI Event Distribution] (3)                               |
| Search String: (4)                                                        |
|   index = web_logs | stats count by URI | sort - count                    |
|---------------------------------------------------------------------------|
| [Add to Dashboard] (5)                                                    |
+---------------------------------------------------------------------------+
| Pie Chart Visualization:                                                  |
|---------------------------------------------------------------------------|
| >>> Pie chart showing URI event distribution <<<                          |
| URIs represented: /pictures.html, /payments.html, /restricted.html,       |
|                   /index.html, /about.html, /trainings.html, /contact.html|
+---------------------------------------------------------------------------+
```

Great job! You've officially built an informative and visually appealing dashboard in Splunk, but why stop there? We can add more panels to display any information we like. In the previous task, we looked at the /restricted.html URI field. Let's create a stats table for our dashboard that shows:

- status_code field
- count of events
- percent of events
- total amount of events overall

Similarly doing the same for restricted.html in statistics table start with, Add Panel button as you did previously, but this time, choose New → Statistics Table. Next, set your Time Range to All time and enter the following query as the Search String:
```
index = web_logs URI = /restricted.html
| stats count by status_code
| eventstats sum(count) as total
| eval percent = round(count * 100.0 / total, 2) 
| sort - count
```
```
+--------------------------------------------------------------------------------+
|                                 ADD PANEL                                      |
+--------------------------------------------------------------------------------+
| New (15)                                                                       |
|   - Events                                                                     |
|   - Statistics Table   (Selected)                                              |
|   - Line Chart                                                                 |
|   - Area Chart                                                                 |
|   - Column Chart                                                               |
|   - Bar Chart                                                                  |
|   - Pie Chart                                                                  |
|   - Scatter Chart                                                              |
|   - Bubble Chart                                                               |
+--------------------------------------------------------------------------------+
|                             NEW STATISTICS TABLE                               |
+--------------------------------------------------------------------------------+
| Time Range: [All time]                                                         |
| Content Title: [status to /restricted.html]                                    |
| Search String:                                                                 |
|   index = web_logs URI = /restricted.html                                      |
|   | stats count by status_code                                                 |
|   | eventstats sum(count) as total                                             |
|   | eval percent = round(count * 100.0 / total, 2)                             |
|   | sort - count                                                               |
|--------------------------------------------------------------------------------|
| [Add to Dashboard]                                                             |
|--------------------------------------------------------------------------------|
| Panel Options: Displaying New Stats Table                                      |
|--------------------------------------------------------------------------------|
| status_code | count | percent | total                                          |
|--------------|--------|----------|--------                                     |
| 204          | 196    | 13.30    | 1474                                        |
| 201          | 189    | 12.82    | 1474                                        |
| 301          | 189    | 12.82    | 1474                                        |
...   
```


---
### Lab Questions

**Q1 Task4: Inspect the URI pie chart you built in the dashboard above. Which URI field value has the least amount of events present?**

/pictures.html is quickly determined using the instructions above


**Q2 Task4: Add another statistics table to your dashboard to view the Source_IP, URI, and status_code fields. How many times did 172.16.0.1 receive the status_code 200 from /payments.html?**

This requires minor adjustment to the query used, adding source_ip and URI:
```
index="web_logs" URI=/payments.html 
| stats count by Source_IP URI status_code
| eventstats sum(count) as total
| eval percent=round(count * 100.0 / total, 2)
| sort - count
```
Really only needs the first 2 and last line to sort it, giving us 50 times 172.16.0.1 received a status 200.


---
## Task4: Extending Splunk's Functionality

Splunk’s core platform ingests and indexes log data, but modern SOCs extend it with advanced security tools. Splunk Enterprise Security (ES) adds a full security operations framework on top of Splunk Enterprise/Cloud, providing correlation searches, notable events, MITRE ATT&CK mapping, risk scoring, investigation workflows, and the SOC Operations dashboard for visibility into analyst workload and efficiency.

Splunk UEBA enhances ES by detecting insider threats and compromised accounts through behavioral analytics. It evaluates user and entity activity over time, aggregates anomalies, assigns risk scores, and maps suspicious behavior to MITRE ATT&CK techniques.

Splunk SOAR (formerly Phantom) introduces automated response (note paid component of splunk not accessible in community addition). Using playbooks, SOAR can perform actions such as isolating hosts, disabling accounts, or checking IP reputation automatically, reducing manual workload and improving response consistency. Playbooks can include conditions, filters, and branching logic to adapt to different alert types and severity levels.

Together, ES + UEBA + SOAR transform Splunk from a visibility tool into a full detection, investigation, and automated response ecosystem.
