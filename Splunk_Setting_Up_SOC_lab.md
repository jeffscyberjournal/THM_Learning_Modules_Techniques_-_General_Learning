# Splunk: Setting Up SOC lab

## Task1: Objectives overview

- Learn how to install Splunk and the Universal Forwarder on a Linux host
- Explore Splunk configuration via the command line
- Understand how to ingest Linux and Web logs into Splunk
- By the end, you'll have a working setup for centralized log monitoring using Splunk.

## Task2: Core Splunk architecture components
```
                        [ Server ]
                            │
                            ▼
                  [     Deployment Server     ]
                /                  │        \
               /                   │         \
              ▼                    ▼          ▼
[       Forwarders     ]         [Indexer]  [Search Head]
 │            │       │            ▲          ▲
 ▼            ▼       ▼            │          │
[Internet] [Linux] [Windows]    ───┘          │
 └────────────────────────────────────────────┘
```

- **Forwarders:** Lightweight agents installed on endpoints to collect and send data. Universal Forwarders send raw data, while Heavy Forwarders can parse or filter data before sending it.
- **Indexers:** Transform, parse, and store incoming data into a structured format called an "index" to make it searchable and available for analysis.
- **Search Heads:** The user interface where analysts run queries, view dashboards, and investigate logs. They perform search-time parsing (extracting fields dynamically during a query), which saves storage flexibility but increases CPU load.
- **Deployment Servers:** A centralized management hub used to remotely configure, update, and manage hundreds or thousands of scattered forwarders simultaneously.

**Splunk vs. Elastic Parsing Comparison**

**Feature	          Splunk	                             Elastic**
```
Parsing Time:       Search-time (on the Search Head)     Index-time (partial or complete)
Data Flexibility:   High (can reinterpret data           Lower (bound by pre-indexed      
                    without reindexing)                  structures)
Performance Impact: Increases CPU load during queries    Reduces query times; increases
                                                         ingestion processing
```
**Q1 Which part of the Splunk deployment architecture is responsible for processing incoming data from forwarders?** Indexer

**Q2 Part of your client's infrastructure includes hundreds of employee workstations.
Which Splunk feature would enable you to remotely manage forwarders on each system?** Deployment Server

## Task3: Linux Deployment
