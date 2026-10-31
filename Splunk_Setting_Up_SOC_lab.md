# Splunk: Setting Up SOC lab

---
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

- **splunk enable boot-start** command to configure Splunk to automatically start as a service whenever the system boots.
- **splunk status** command is used to check the status of the Splunk server. Including whether it is running or not, and any errors that may be occurring.

```
root@coffely:/opt/splunk/bin# ./splunk enable boot-start
Init script installed at /etc/init.d/splunk.
Init script is not configured to run at boot.

root@coffely:/opt/splunk/bin# ./splunk status
splunkd is running (PID: 9045).
splunk helpers are running (PIDs: 9046 9224 9229 9332 9378 9576).
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

**Q1 Using the CLI, what command would you run to search Splunk for the term coffely?** ./splunk search coffely

**Q2 Which Splunk command would you run to get help with Splunk searches?** ./splunk help search

---
## Task 5: Splunk Universal Forwarder

- Configuring data ingestion, indexed searchable incoming data
- Wide range of log sources from operating systems, web applications, intrusion detection systems, and more.
- Forwarders similar for forwarding syslog data on wazuh siem.
- Forwarders are installed on remote machines, such as endpoints, servers, or network devices, and send their data to a central Splunk server for indexing and analysis. 


**Installing the Universal Forwarder**

- Check out the Splunk site to download the universal forwarder, which is available for all major OS versions.
- For this exercise, the 64-bit version for Linux has been placed in the ~/Downloads/splunk directory.
- The Linux Splunk forwarder options available on the Splunk website.

Go ahead and navigate back to the /home/ubuntu/Downloads/splunk directory. Ensure you're running as root, and run the tar command to install the forwarder and move it into the /opt directory.


**Forwarder Install**
```
ubuntu@coffely:~/Downloads/splunk/$ ls
splunk_installer.tgz splunkforwarder.tgz
ubuntu@coffely:~/Downloads/splunk/$ sudo su
root@coffely:~/Downloads/splunk/#
root@coffely:~/Downloads/splunk/# tar xvzf splunkforwarder.tgz -C /opt
splunkforwarder/
splunkforwarder/swidtag/
...
```
**Starting the Forwarder**

- Navigate to **/opt/splunkforwarder/bin** and use **./splunk start --accept-license** to start the forwarder and enter your new credentials.
- By default, both Splunk Enterprise and the Splunk Universal Forwarder use port 8089 for management traffic (splunkd).
- Since both are running on the same machine, this creates a port conflict. When prompted, change the forwarder management port to 8090 so both services can run simultaneously.


**Forwarder Start**
```
root@coffely:~/Downloads/splunk/# cd /opt/splunkforwarder/bin
root@coffey:/opt/splunkforwarder/bin# ./splunk start --accept-license
This appears to be your first time running this version of Splunk.
...
...
Please enter an administrator username: admin
Password must contain at least:
  * 8 total printable ASCII character(s).
Please enter a new password: 
Please confirm new password: 
...
...
Checking prerequisites...
    Checking mgmt port [8089]: not available
ERROR: mgmt port [8089] - port is already bound.  Splunk needs to use this port.
Would you like to change ports? [y/n]: y
Enter a new mgmt port: 8090
Setting mgmt to port: 8090
The server's splunkd port has been changed.
    Checking mgmt port [8090]: open        
Starting splunk server daemon (splunkd)...  
Done
```

---
## Task 6: Configuring the Forwarder

Now that we have installed the forwarder, it needs to know where to send the data. So we will configure it on the host end to send the data and configure Splunk so that it knows from where it is receiving the data from.

**Splunk Configuration** 

- Log in to Splunk main management interface (localhost:8000), head to Settings, and click Forwarding and receiving.
- You will see multiple options to configure both forwarding and receiving. We want to receive data from the Linux endpoint, so click + Add new in the Receive data section.
- By default, the Splunk instance receives data from the forwarder on port 9997. It's up to us whether we want to use this port or change it. For now, we will configure our Splunk to start listening on port 9997 and Save.
- At any time, you can return to this page and disable or delete this configuration.

**Creating an Index**

- Settings again and click on the Indexes option. We will create a new index to store the data we want to receive. If no index is created and specified, Splunk will store received data in the default index main.
- The indexes tab contains all the indexes created by the user or by default. By default, Splunk contains internal logs used to monitor its own operations and performance.  It shows important metadata about the indexes, like size, event count, and home path. Go ahead and click the New Index button.
- The Splunk Indexes menu with the Splunk default internal indexes and New Index button highlighted.
- Fill index name with linux_host and keep the default settings for the rest. Save your new index.

**Configuring the Forwarder**

- It's time to configure the forwarder to ensure it sends the data to the right destination. On the Linux host terminal, add the forward server using the command below.


```
root@coffely:/opt/splunkforwarder/bin# ./splunk add forward-server 10.67.187.209:9997
Splunk username: admin
Password: 
Added forwarding to: 10.67.187.209:9997.
```

**Linux Log Sources**

- Linux stores all of its important logs into the /var/log directory.
- In our case, we will ingest the syslog file into Splunk.
- All other logs can be ingested using the same method.

```
root@coffely:/opt/splunkforwarder/bin# ls /var/log
README              btmp.1                  gpu-manager.log      speech-dispatcher
Xorg.0.log          cloud-init-output.log   hp                   sssd
Xorg.0.log.old      cloud-init.log          journal              syslog
alternatives.log    cloud-init.log.1        kern.log             syslog.1
alternatives.log.1  cups                    kern.log.1           sysstat
amazon              cups-browsed            landscape            ubuntu-advantage.log
apache2             dist-upgrade            lastlog              ubuntu-advantage.log.1
apport.log          dmesg                   lightdm              unattended-upgrades
apport.log.1        dmesg.0                 openvpn              upgrade
apt                 dpkg.log                prime-offload.log    wtmp
auth.log            dpkg.log.1              prime-supported.log
auth.log.1          fontconfig.log          private
btmp                gpu-manager-switch.log  samba
```

Next, use the ./splunk add monitor command below to add the syslog file to our newly created index.
```
root@coffely:/opt/splunkforwarder/bin# ./splunk add monitor /var/log/syslog -index linux_host
Added monitor of '/var/log/syslog'.
```
**Inputs**
- Splunk inputs define what data Splunk should collect.
- Can be manually edited inputs.conf to specify new data sources or modify existing input settings.
```
root@coffely:/opt/splunkforwarder/bin# cat /opt/splunkforwarder/etc/apps/search/local/inputs.conf
[monitor:///var/log/syslog]
disabled = false
index = linux_host
Using the Logger Utility
```

**Logger Utility For Testing**
Logger is a built-in command-line tool to add test logs to the syslog file. We are already monitoring the syslog file and sending all logs to Splunk, so the log generated with the command below will be searchable in our Splunk instance.

```
root@coffely:/opt/splunkforwarder/bin# logger "coffely-has-the-best-coffee-in-town"
root@coffely:/opt/splunkforwarder/bin# tail -1 /var/log/syslog
2025-11-06T05:37:11.415531+00:00 coffely root: coffely-has-the-best-coffee-in-town
```

Search & Reporting app in your Splunk: test log with the following query:
```
index = linux_host "coffely-has-the-best-coffee-in-town"
```

### Lab Questions and Answer:

**Q1 Follow the steps from above, but this time to ingest the /var/log/auth.log file into the linux_host index. What is the value in the sourcetype field?**

**Start by adding /var/log/auth.log similar to what was done for syslog above:**
```
root@coffely:/opt/splunkforwarder/bin# ./splunk add monitor /var/log/auth.log -index linux_host
Added monitor of '/var/log/auth.log'.
```
**Verify its been added to the searchable inputs.conf:**
```
root@coffely:/opt/splunkforwarder/bin# cat /opt/splunkforwarder/etc/apps/search/local/inputs.conf
[monitor:///var/log/syslog]
disabled = false
index = linux_host

[monitor:///var/log/auth.log]
disabled = false
index = linux_host
root@coffely:/opt/splunkforwarder/bin# 
```
- Revisit splunk search and reporting and just start with index=linux_host
- View sourcetypes
```
sourcetype

Values 	Count	 
syslog  3,327 	
auth    60
```
**Answer is auth**

**Q2 Create a new user named analyst using the command adduser analyst. 
Once created, look at the syslog events generated in Splunk related to the user creation activity. What is the value of the process field when the new user is added?**

- Using a simple search index=linux_host "new user"
- Or index = linux_host analyst
Then search All fields and add process field

**From index=linux_host "new user" search of 3 events.**
```
10/30/26 1:21:26.278 PM	
2026-10-30T13:21:26.278390+00:00 coffely adduser[10579]: Adding new user `analyst' to supplemental / extra groups `users' ...
Event Actions
Type             Field             Value
Selected	       host              coffely	
                 source            /var/log/syslog	
                 sourcetype        syslog	
Event            pid               10579	
                 process           adduser	
Time             _time             2026-10-30T13:21:26.278+00:00
Default          index             linux_host	
                 linecount         1	
                 punct             --::.+:__[]:____'___/___'_...	
                 splunk_server     coffely	​‌
```
**From adding field:**
```
process
1 Value, 12.5% of events
Events with this field
Values 	Count 	 	 
adduser  1
```
---
## Task 7: Windows forwarding

- Splunk supports a range of log forwarding, including Windows, email servers, or firewalls.
- Log aggregation at central location, normalization and indexing of logs.
- Allows control of correlation and clearer view of incidents.
- This section is read along only but covers the configuration for forwarding from a Windows system.
- This will require atleast 2 VM for Windows and Linux architecture. You can install the universal forwarder on the Windows machine and specify the IP address of your Linux instance to act as the indexer during the installation process.
- Using PowerShell, we can head to the forwarder directory and add a new monitor. Not dissimilar to adding syslog in linux.
```
PS C:\> cd "Program Files\SplunkUniversalForwarder\bin"
PS C:\Program Files\SplunkUniversalForwarder\bin> .\splunk.exe add monitor C:\Windows\System32\winevt\Logs\Security.evtx
Added monitor of 'C:\Windows\System32\winevt\Logs\Security.evtx'.
```
Test same way with search and reporting in splunk.

### Deployment Server

- An alternative you can turn your Splunk instance into a deployment server, allowing you to manage other Splunk instances and forwarders remotely.
```
PS C:\> cd "Program Files\SplunkUniversalForwarder\bin"
PS C:\Program Files\SplunkUniversalForwarder\bin> .\splunk.exe add monitor C:\Windows\System32\winevt\Logs\Security.evtx
Added monitor of 'C:\Windows\System32\winevt\Logs\Security.evtx'.
```
- Login to deployment server used earlier in linux, go to Settings, select Data Inputs, scroll down to Forwarder inputs, and choose **+ Add new** that is on right next to Windows Event Logs.
- We now see Windows machines available in the 'available hosts' column, select it and give it a 'New Server Class name'. Server classes are used to define a single instance or group of Splunk deployment clients (Splunk instance, forwarder, indexer, or search head).
- Finally, we can create a new index named Windows, review our data input, and submit just as we did in the previous steps.
- Test in splunk with index=windows.
- Check sources should show 4 WinEventlog Security, System, Application and Setup logs.

### Lab Question Answer

**Q1 What command would you use on Linux to enable your Splunk instance as a deployment server?**

Answer: ./splunk enable deploy-server 


---
## Task 8: Ingesting Web Logs

Ingesting logs from the website already on the VM where Splunk deployment server in installed. Accessed via http://coffely.thm:8000 from the VM. This task configures to receive web logs to trace and improve coffee sales.
- Site will enable logging or coffee orders online, track request and responses and orders placed.

### Getting Started

- Running on Apache, access logs we need '/var/log/apache2/access.log'. This can be done via the command line method used like for syslog logs, or via Splunk interface.

### Adding Data

- In Splunk, navigate to 'Settings' → 'Add Data'.
- Scroll down and select 'Monitor'.
- Choose 'Files & Directories', then enter the path of the file to 'monitor':
```
/var/log/apache2/access.log
```
- The reference guide recommends configuring the file for continuous monitoring rather than a one-time import. This ensures new log entries are automatically ingested into Splunk as they are written to the file.
- Next select SourceType dropdown, select Web, and then choose 'access_combined'.
- Next assign a name to the host field value. Let's change it to 'coffelyweb', then create a new index named 'web', just like we did for the previous logs.
- Finally, review then submit, start searching through newly monitored file. Logs won't be added automatically. You need to interact with the website 'http://coffely.thm:8000' after all its continuous monitoring. Then start with:
```
index=web sourcetype=access_combined host=coffelyweb source="/var/log/apache2/access.log"
```
### Lab Questions

**Q1 Which sourcetype should be used to parse and categorize Apache access logs correctly?**


**Q2 After browsing the Coffely site, which root field value has the highest count?**


**Q3 What is the full uri_path field value for cappuchino.png?**

/______/_____________/__________.___


**Q4 Navigate to http://coffely.thm:8080/secret-flag.html Check out the list of recent orders and find the flag!**

