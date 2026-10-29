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

---
## Task 4: Managing Splunk From CLI

- Run from the /opt/splunk/bin directory.

- **splunk start** command is used to start the Splunk server. Th
- Splunk can be stopped with **splunk stop**.
- **splunk restart** is useful when changes have been made to Splunk's configuration. 
```
root@coffely:/opt/splunk/bin# ./splunk start
Splunk> Finding your faults, just like mom.
…
Checking prerequisites...
    Checking http port [8000]: open
    Checking mgmt port [8089]: open
    Checking appserver port [127.0.0.1:8065]: open
    Checking kvstore port [8191]: open
    Checking configuration... Done.
...
...
The Splunk web interface is at http://coffely:8000
```

### Further Useful Commands

- **splunk enable** boot-start command to configure Splunk to automatically start as a service whenever the system boots.
- **splunk status** command is used to check the status of the Splunk server. Including whether it is running or not, and any errors that may be occurring.

```
**Splunk Status**
root@coffely:/opt/splunk/bin# ./splunk status
splunkd is running (PID: 45852).
splunk helpers are running (PIDs: 45853 46296 46305 46414 46511 46786).
```

The **splunk add oneshot** command is used to add a single event to the Splunk index. This is useful for testing purposes or for adding individual events that may not be part of a larger data stream.

**Splunk One Shot**
```
root@coffely:/opt/splunk/bin# ./splunk add oneshot /path/to/logfile -index yourindex
...
```

The **splunk search** command is used to search for data in the Splunk index. This command can be used to search for specific events, as well as to perform more complex searches using Splunk's search processing language.

**Splunk Search**
```
root@coffely:/opt/splunk/bin# ./splunk search coffely 
WARNING: Server Certificate Hostname Validation is disabled. Please see server.conf/[sslConfig]/cliVerifyServerName for details.
Feb 18 21:09:04 coffley ubuntu: coffely-has-the-best-coffee-in-town
Feb 18 13:48:17 coffely ubuntu: COFFELY
Feb 18 13:48:17 coffely ubuntu: COFFELY
...
```

Try out **splunk help** to view all available Splunk CLI commands, their descriptions, and usage syntax.


**Splunk Help**
```
root@coffely:/opt/splunk/bin# ./splunk help
Welcome to Splunk's Command Line Interface (CLI).

   Type these commands for more help:

       help [command]             type a command name to access its help page
       help [object]              type an object name to access its help page
       help [topic]               type a topic keyword to get help on a topic
       help commands              display a full list of CLI commands
       help clustering            commands that can be used to configure the clustering setup
       help shclustering          commands that can be used to configure the Search Head Cluster setup
       help control, controls     tools to start, stop, manage Splunk processes
       help datastore             manage Splunk's local filesystem use
    ...
```
These are just a few of the many CLI commands available in Splunk. Administrators can leverage the command-line interface to efficiently manage, configure, and troubleshoot their Splunk environments.

