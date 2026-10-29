# Splunk: Setting Up SOC lab

## Task1: Objectives overview

- Learn how to install Splunk and the Universal Forwarder on a Linux host
- Explore Splunk configuration via the command line
- Understand how to ingest Linux and Web logs into Splunk
- By the end, you'll have a working setup for centralized log monitoring using Splunk.

---
## Task2: Core Splunk architecture components
```
                        [─Server─]
                            │
                            ▼
                [─────Deployment Server────]
                /                  │        \
               /                   │         \
              ▼                    ▼          ▼
[─────Forwarders──────]       [Indexer]  [Search Head]
 │            │       │            ▲          ▲
 ▼            ▼       ▼            │          │
[Internet] [Linux] [Windows]──────┘           │
 └────────────────────────────────────────────┘
```

- **Forwarders:** Lightweight agents installed on endpoints to collect and send data. Universal Forwarders send raw data, while Heavy Forwarders can parse or filter data before sending it.
- **Indexers:** Transform, parse, and store incoming data into a structured format called an "index" to make it searchable and available for analysis.
- **Search Heads:** The user interface where analysts run queries, view dashboards, and investigate logs. They perform search-time parsing (extracting fields dynamically during a query), which saves storage flexibility but increases CPU load.
- **Deployment Servers:** A centralized management hub used to remotely configure, update, and manage hundreds or thousands of scattered forwarders simultaneously.

**Splunk vs. Elastic Parsing Comparison**

**Feature	          Splunk	                             Elastic**
```
Parsing Time:       | Search-time (on the Search Head)     | Index-time (partial or complete)
───────────────────────────────────────────────────────────────────────────────────────────────
Data Flexibility:   | High (can reinterpret data           | Lower (bound by pre-indexed      
                    | without reindexing)                  | structures)
───────────────────────────────────────────────────────────────────────────────────────────────
Performance Impact: | Increases CPU load during queries    | Reduces query times; increases
                    |                                      | ingestion processing
```
**Q1 Which part of the Splunk deployment architecture is responsible for processing incoming data from forwarders?** Indexer

**Q2 Part of your client's infrastructure includes hundreds of employee workstations.
Which Splunk feature would enable you to remotely manage forwarders on each system?** Deployment Server

---
## Task3: Linux Deployment

- Create a Splunk account and download the latest installation package.
- Trial enterprise version basically after intro period affectively becomes the community addition.
- https://www.splunk.com/en_us/download/splunk-enterprise.html?locale=en_us

### Linux Splunk Install: 

- Assuming its already downloaded in the Downloads directory
- Installation requires root privilege for installation

```
ubuntu@coffely:~$ cd ~/Downloads/splunk 
ubuntu@coffely:~/Downloads/splunk$ ls
splunk_installer.tgz  splunkforwarder.tgz
ubuntu@coffely:~/Downloads/splunk$ sudo su
root@coffely:/home/ubuntu/Downloads/splunk#  tar xvzf splunk_installer.tgz -C /opt
...
```
### Starting Splunk

- Start splunk as follows
- Create admin password
  
```
root@coffely:/home/ubuntu/Downloads/splunk# cd /opt/splunk/bin
root@coffely:/opt/splunk/bin# ./splunk start --accept-license

This appears to be your first time running this version of Splunk.

Splunk software must create an administrator account during startup. Otherwise, you cannot log in.
Create credentials for the administrator account.
Characters do not appear on the screen when you type in credentials.

Please enter an administrator username: admin
Password must contain at least:
   * 8 total printable ASCII character(s).
Please enter a new password: 
Please confirm new password: 
Copying '/opt/splunk/etc/openldap/ldap.conf.default' to '/opt/splunk/etc/openldap/ldap.conf'.
writing RSA key
...
Starting splunk server daemon (splunkd)...  
...
Waiting for web server at http://127.0.0.1:8000 to be available.................................. Done

If you get stuck, we're here to help.  
Look for answers here: http://docs.splunk.com

The Splunk web interface is at http://coffely:8000

root@coffely:/opt/splunk/bin# 
```
**Q1 In which Linux directory, typically used for add-on or third-party software, should Splunk be installed?** /opt

**Q2 Which port does the Splunk web interface run on by default?** 8000


