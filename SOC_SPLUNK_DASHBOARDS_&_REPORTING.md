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

Lab answers
**Q1 Task2: Head to the Splunk Reports tab and open the Web Connections by Source IP report.
Which Source_IP field value recorded the highest number of events?**
Highest Source_IP: 10.0.0.1

**Q2 Task2: While viewing the report above, click Edit → Open in Search to investigate the query behind the report.
What is the flag value hidden within the search query?**
Hidden flag in query: THM{splunk_report_wizard!}

## Task3: 
