# GreenFlag SIEM

![C#](https://img.shields.io/badge/C%23-239120?style=for-the-badge\&logo=c-sharp\&logoColor=white)
![ASP.NET](https://img.shields.io/badge/ASP.NET-5C2D91?style=for-the-badge\&logo=.net\&logoColor=white)
![SQL Server](https://img.shields.io/badge/SQL_Server-CC2927?style=for-the-badge\&logo=microsoft-sql-server\&logoColor=white)
![UDP](https://img.shields.io/badge/UDP-00599C?style=for-the-badge\&logo=databricks\&logoColor=white)

GreenFlag is a lightweight SIEM solution developed with C#. It is designed around a modular architecture for collecting, processing, storing, and monitoring security-related event data.

## System Showcase

### UI Themes & Dashboard

|             Dashboard — Dark Mode            |             Dashboard — Light Mode             |
| :------------------------------------------: | :--------------------------------------------: |
| ![Dashboard Dark](Assets/GreenFlag-Dark.png) | ![Dashboard Light](Assets/GreenFlag-Light.png) |

### Log Monitoring & Management

|            Log History & Filtering           |             System Management Panel             |
| :------------------------------------------: | :---------------------------------------------: |
| ![Log History](Assets/GreenFlag-History.png) | ![Management Panel](Assets/GreenFlag-Panel.png) |

### Rules & Executive Reporting

|            Rule Management & Alerts            |                   Executive Summary — Excel                  |
| :--------------------------------------------: | :----------------------------------------------------------: |
| ![Rule Management](Assets/GreenFlag-Rules.png) | ![Executive Summary](Assets/GreenFlag-Executive-Summary.png) |

---

# Overview

GreenFlag is a lightweight SIEM solution developed with C#. The project focuses on collecting, processing, storing, and monitoring security-related event data through a modular architecture.

The current version collects Windows Event Logs through the GreenFlag Agent, processes them through the Agent and Ingestor pipeline, stores the resulting data in SQL Server, and provides a web-based interface for monitoring and management.

GreenFlag is currently under active development. The core collection, processing, storage, monitoring, and reporting pipeline is functional, while several advanced SIEM capabilities are still being developed.

# Architecture

GreenFlag consists of three main components:

```text
Windows Event Log
        │
        ▼
GreenFlag Agent
        │
        │ UDP / JSON
        ▼
GreenFlag Ingestor
        │
        ├── Agent Memory
        ├── Log Queue
        ├── Rule Memory
        ├── Heartbeat Queue
        ├── SQL Worker
        ├── Heartbeat Worker
        └── Log Doctor
        │
        ▼
    SQL Server
        ▲
        │
GreenFlag Web
```

The Web application does not communicate directly with the Ingestor. It reads and manages the data stored in SQL Server.

---

# GreenFlag Agent

The GreenFlag Agent runs on Windows endpoints and collects events from Windows Event Log (WEM).

## Event Tracking

At startup, the Agent checks the available event records and uses the latest processed `EventRecordId` as its reference point.

Instead of repeatedly processing the entire event history, it continues monitoring from the point it is tracking.

For example:

```text
long lastRecordId = GetCurrentMaxRecordId(channel);

while (!token.IsCancellationRequested)
{
    string queryStr =
        $"*[System[(EventRecordID > {lastRecordId})]]";

    var query = new EventLogQuery(
        channel.Name,
        PathType.LogName,
        queryStr
    );

    using var reader = new EventLogReader(query);

    EventRecord record;

    while ((record = reader.ReadEvent()) != null)
    {
        using (record)
        {
            if (record.RecordId.HasValue)
                lastRecordId = record.RecordId.Value;

            var pkt = parser.Parse(record);
            sendQueue.Add(pkt);
        }
    }
}
```

`EventRecordId` is different from the Windows Event ID.

```text
EventRecordId = 500
EventId       = 4624
```

## Log Channels

The Agent manages different Windows Event Log sources through the `ILogChannel` interface.

Current channels include:

* Security
* Application
* System

This allows different event sources to be processed independently.

## Event Parsing

Raw Windows event data is processed before being sent to the Ingestor.

The Agent uses the `IEventParser` architecture together with `ParserFactory` to convert raw event data into a structured format.

```text
Raw Windows Event
        │
        ▼
   IEventParser
        │
        ▼
   ParserFactory
        │
        ├── Security Parser
        ├── Application Parser
        └── System Parser
        │
        ▼
 Structured Event
```

The parser extracts and organizes relevant information such as event messages and machine information.

## Waiting for New Events

When no new event is available, the Agent does not continuously poll the event source.

It waits for approximately five seconds before checking again.

This reduces unnecessary CPU usage during periods with no new events.

---

# UDP Sender

Processed logs are stored as `Event_Paket` objects and grouped into batches before transmission.

A packet is sent when:

* The configured packet capacity is reached, or
* The configured time interval expires.

Logs are accumulated in memory and transmitted when either the batch reaches 20 events or the configured time interval expires:

```text
if (sendQueue.TryTake(out var pkt, 500))
{
    batch.Add(pkt);
}

bool timeUp =
    (DateTime.Now - lastFlush).TotalSeconds >= 5;

bool batchFull = batch.Count >= 20;

if ((timeUp || batchFull) && batch.Count > 0)
{
    SendBatch(batch);
    batch.Clear();
    lastFlush = DateTime.Now;
}
```
Oversized serialized batches are split into individual packets before transmission.

```text
if (data.Length > 60_000)
{
    foreach (var log in logs)
    {
        byte[] single = Encoding.UTF8.GetBytes(
            JsonSerializer.Serialize(new[] { log })
        );

        udpClient.Send(single, single.Length);
    }
}
else
{
    udpClient.Send(data, data.Length);
}
```

The Agent also determines its local IP address and includes this information in its heartbeat data.

---

# Agent Heartbeat

The Agent periodically sends heartbeat information to the Ingestor over UDP using JSON.

Heartbeat information can include:

* Agent ID
* Username
* IP Address
* Hostname
* Agent status information

```text
Agent
  │
  ▼
Heartbeat Object
  │
  ▼
JSON Serialization
  │
  ▼
UDP
  │
  ▼
Ingestor
```

---

# GreenFlag Ingestor

The Ingestor is the central processing component of GreenFlag.

It receives logs and heartbeat information from Agents over UDP, processes incoming data, manages queues, applies rules, enriches logs, and forwards processed data to SQL Server.

## UDP Server

The Ingestor operates as a UDP server and uses a 64 MB receive buffer.

The UDP socket is also handled to reduce Windows-side socket and port issues related to UDP communication.

## Incoming Packet Processing

Incoming UDP packets follow this general flow:

```text
UDP Packet
    ↓
JSON Deserialization
    ↓
Event_Paket List
    ↓
Log Validation
    ↓
Log Queue
```

If JSON deserialization fails:

* The invalid packet is counted.
* The error is recorded.
* The error is printed to the console.
* Processing continues.

This prevents a single malformed packet from terminating the entire Ingestor.

---

# Log Queue & Load Protection

Incoming logs are stored in `LogKuyruk`.

The current queue limit is:

```text
100,000 logs
```

This prevents unlimited memory growth under heavy load.

When the queue reaches its limit, the Ingestor checks the event's configured rule and severity.

Lower-priority logs can be dropped when the queue is full, while higher-priority logs are preserved whenever possible.

```text
Queue Full
   │
   ├── Low-priority log → DROP
   │
   └── Important log → KEEP
```

To prevent unlimited queue growth, the Ingestor applies load protection when the queue reaches its configured limit. Lower-priority events can be dropped while higher-priority events are preserved whenever possible.

```text
byte anlikSeverity = 1;
bool kuralvar = false;

if (KuralHafizasi.TryGetValue(
    log.event_id,
    out KuralBilgisi kural))
{
    anlikSeverity = kural.Severity;
    kuralvar = true;
}

if (LogKuyruk.Count >= kuyrukLimit &&
    (!kuralvar || anlikSeverity <= 1))
{
    Interlocked.Increment(ref DroppedQueueCount);
    continue;
}

LogKuyruk.Enqueue(log);
```

Dropped logs are counted separately through `DroppedQueueCount`.

This provides a basic load-protection mechanism for high-volume situations.

---

# Agent Memory

The Ingestor maintains Agent information received from heartbeats and incoming logs.

Information such as:

* Agent ID
* Hostname
* Username
* IP Address

can be kept in memory and used later for log enrichment.

For example:

```text
Hostname
    ↓
Agent Information
    ↓
AgentId
Username
IP Address
```

This allows incoming events to be associated with their corresponding Agent.

---

# Rule Memory

The Ingestor loads rules from SQL Server and keeps them in memory.

Rules are associated with event IDs and can define values such as:

* Severity
* Category

Conceptually:

```text
Event ID
    ↓
Rule Information
    ↓
Severity
Category
```
When an incoming event is processed, its `event_id` is compared against the rules stored in `KuralHafizasi`.

If a matching rule exists, the event's severity and category are updated accordingly.

---

# Asynchronous Workers

The Ingestor uses separate asynchronous workers for different responsibilities.

```text
                 GreenFlag Ingestor
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   SQL Worker    Heartbeat Worker   Log Doctor
        │               │               │
        ▼               ▼               ▼
    SQL Logs      Agent Statuses    Pipeline Stats
```

## SQL Worker

`SQLWorker` moves logs from the Log Queue to SQL Server.

The current implementation processes approximately 100 logs per batch.

```text
Log Queue
   ↓
Batch (100)
   ↓
Bulk DB Write
   ↓
SQL Server
```

After the batch is processed, it is cleared and the worker continues processing subsequent logs.

```text
int batchsize = 100;
List<Event_paket> batch = new List<Event_paket>(batchsize);

while (!token.IsCancellationRequested)
{
    while (batch.Count < batchsize &&
           LogKuyruk.TryDequeue(out var log))
    {
        batch.Add(log);
    }

    if (batch.Count > 0 &&
        (batch.Count >= batchsize || LogKuyruk.IsEmpty))
    {
        await dbyaz(batch);
        batch.Clear();
    }
}
```

## Heartbeat Worker

`HeartbeatWorker` processes heartbeat data received over UDP and writes the corresponding Agent information to the `AgentStatuses` table.

```text
Agent
 ↓
Heartbeat
 ↓
UDP
 ↓
Heartbeat Queue
 ↓
HeartbeatWorker
 ↓
AgentStatuses
```

## Log Doctor

`LogDoctor` periodically checks the health of the log processing pipeline.

It can monitor:

* Incoming log count
* Dropped log count
* Invalid JSON count
* Total traffic
* Lost/dropped data
* Loss rate

The results are recorded and also printed to the console.

---

# Log Processing & Enrichment

Before logs are written to SQL Server, they go through several processing stages.

## Default Values

Incoming logs initially receive:

```text
Severity = 1
```

If no category is available:

```text
Category = Unknown
```

Hostname information is preserved when available.

## Rule-Based Processing

The event's `event_id` is checked against the rule memory.

If a matching rule is found:

```text
Event ID
   ↓
Rule Match
   ├── Severity
   └── Category
```

The corresponding values are applied to the log.

## Agent Enrichment

Agent information can also be added to the log before it is stored.

```text
Hostname
    ↓
GetOrFetchAgentInfoAsync()
    ↓
Agent Information
    ├── Agent ID
    ├── Username
    └── IP Address
```

This allows logs to contain additional context that may not have been present in the original event.

---

# SQL Database Processing

After processing and enrichment, the log record contains information such as:

* Event Time
* Event ID
* Agent ID
* Hostname
* Username
* IP Address
* Category
* Source
* Severity
* Message
* Ingest Time

Logs are processed in batches and written to SQL Server in bulk.

This helps reduce database round trips and SQL Server overhead while improving throughput under higher log volumes.

```text
Queue
 ↓
Batch
 ↓
Processing
 ↓
Rule Processing
 ↓
Enrichment
 ↓
Bulk Insert
 ↓
SQL Server
```

---

# Dynamic Rule Loading

The Ingestor periodically checks SQL Server for updated rules.

New or modified rules are loaded into `KuralHafizasi`.

This allows rule changes to affect incoming log processing without restarting the Ingestor.

```text
SQL Server Rules
       ↓
Load Rules
       ↓
Rule Memory
       ↓
Incoming Logs
       ↓
Severity / Category
```
```text
string sql =
    "SELECT EventId, Severity, Category " +
    "FROM Rules WHERE IsEnabled = 1";

using SqlCommand cmd = new SqlCommand(sql, conn);
using SqlDataReader reader = cmd.ExecuteReader();

while (reader.Read())
{
    int id = reader.GetInt32(0);
    byte sev = reader.GetByte(1);
    string cat = reader.GetString(2);

    yeniHafiza[id] = new KuralBilgisi
    {
        Severity = sev,
        Category = cat
    };
}

KuralHafizasi = yeniHafiza;
```
---

# Agent Status

Agent information is stored in the `AgentStatuses` table.

Tracked information can include:

* Agent ID
* Hostname
* Username
* IP Address
* Heartbeat information
* Last known Agent status

This information is used for both Agent monitoring and log enrichment.

---

# System Statistics

The Ingestor records system-level statistics in SQL Server.

These can include:

* Incoming log count
* Dropped log count
* Invalid JSON count
* Traffic information
* Processing statistics

These statistics provide visibility into the health and performance of the ingestion pipeline.

---

# Error Handling

GreenFlag uses fault isolation throughout the Agent and Ingestor pipeline.

The system is designed so that a single:

* Malformed JSON packet
* Invalid UDP packet
* SQL error
* Worker error
* Parser error
* Unexpected event

does not unnecessarily terminate the entire pipeline.

Errors are handled by the relevant component and printed to the console, while processing continues whenever possible.

JSON deserialization errors and dropped logs are also counted separately, allowing the system to track potential data loss.

---

# GreenFlag Web

The Web application is the visual and management layer of GreenFlag.

The Web application does **not** communicate directly with the Ingestor.

Instead, it connects directly to SQL Server:

```text
GreenFlag Web
      │
      ▼
SQL Server
```

The Ingestor follows a separate pipeline:

```text
Agent
  ↓
UDP
  ↓
Ingestor
  ↓
SQL Server
```

## Web Features

The Web application currently provides:

* Log viewing
* Log filtering
* Log history
* Rule management
* User management
* Agent information
* Monitoring
* Reporting
* Statistics

---

# Reporting

### Detailed Excel Analytics

|                   Security Events Data                   |                       Report Statistics                      |
| :------------------------------------------------------: | :----------------------------------------------------------: |
| ![Security Events](Assets/GreenFlag-Security-Events.png) | ![Report Statistics](Assets/GreenFlag-Report-Statistics.png) |

GreenFlag includes an integrated Excel reporting system.

Reports can contain:

* Executive Summary
* Security Events
* Severity statistics
* Top Hostnames
* Top IP Addresses
* Top Event Categories

The reporting system uses the same filtered event data available through the Web interface.

---

# Complete Data Flow

```text
┌──────────────────────────┐
│ Windows Event Log        │
│ Security / Application / │
│ System                   │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ GreenFlag Agent          │
│                          │
│ EventRecord Tracking     │
│ ILogChannel              │
│ IEventParser             │
│ ParserFactory            │
│ Event_Paket              │
│ UDP Sender               │
│ Heartbeat                │
└────────────┬─────────────┘
             │
             │ UDP / JSON
             ▼
┌──────────────────────────┐
│ GreenFlag Ingestor       │
│                          │
│ UDP Server               │
│ JSON Processing          │
│ Agent Memory             │
│ Rule Memory              │
│ Log Queue                │
│ Heartbeat Queue          │
│                          │
│ ┌──────────────────────┐ │
│ │ SQL Worker           │ │
│ │ Heartbeat Worker     │ │
│ │ Log Doctor           │ │
│ └──────────────────────┘ │
└────────────┬─────────────┘
             │
             │ Batch / Bulk
             ▼
┌──────────────────────────┐
│ SQL Server               │
│                          │
│ Logs                     │
│ AgentStatuses            │
│ Rules                    │
│ System Statistics        │
│ Users                    │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ GreenFlag Web            │
│                          │
│ Monitoring               │
│ Log Management           │
│ Rule Management          │
│ User Management          │
│ Reporting                │
│ Statistics               │
└──────────────────────────┘
```

---

# Current Technical Capabilities

The current implementation provides:

* Windows Event Log collection
* Event tracking using `EventRecordId`
* Security, Application, and System log channels
* Parser-based event processing
* Parser selection through `ParserFactory`
* UDP-based log transmission
* JSON-based packet serialization
* Batched log transmission
* Agent heartbeat transmission
* Agent information tracking
* Centralized UDP-based log ingestion
* JSON error detection and counting
* 100,000-log queue capacity
* Load protection through selective low-priority log dropping
* Protection of higher-priority logs under queue pressure
* Rule memory
* Dynamic rule loading
* Severity and category enrichment
* Agent information enrichment
* Asynchronous worker-based processing
* Batch SQL processing
* Bulk SQL insertion
* Agent status monitoring
* System statistics
* Log pipeline health monitoring
* Loss-rate tracking
* Web-based log monitoring
* Log filtering
* Rule management
* User management
* Excel reporting
* Basic event statistics and analysis

---

# Vision & Roadmap

GreenFlag is currently under active development.

The long-term goal is to evolve the project into a more complete and extensible SIEM platform while preserving its modular architecture.

Planned improvements include:

* Event correlation engine
* More advanced detection capabilities
* Expanded alerting capabilities
* Improved incident investigation and management
* More advanced endpoint telemetry
* Further development of the existing Agent for more flexible endpoint data collection
* Linux Agent
* Multi-platform endpoint support
* Advanced analytics
* More advanced reporting

The current Agent already provides Windows event collection. Future development will focus on expanding its capabilities and making endpoint data collection more flexible and platform-independent.

The modular architecture is intended to allow these capabilities to be introduced progressively without requiring a complete redesign of the existing system.
