# Log Flow
## End-to-End Data Pipeline — From Source to Indexed Event

---

## Introduction

This document describes the complete log flow architecture 
of the Enterprise SOC — from the moment an event occurs 
on an endpoint or network device, through every processing 
step, to its final indexed state in Splunk where it becomes 
searchable and available for detection and investigation.

Understanding the full data pipeline is essential for:
- Diagnosing why events are not appearing in Splunk
- Estimating data volume and storage requirements
- Designing accurate detection rules
- Explaining the monitoring architecture to stakeholders

---

## Pipeline Overview

```
EVENT OCCURS
     |
     ├── On a Windows endpoint → Sysmon/Windows Event Log path
     └── On the Cisco Meraki   → Network syslog path

                    ↓                           ↓

        ENDPOINT PATH                   NETWORK PATH
        ───────────                     ────────────
        Sysmon captures event           Meraki captures event
              |                               |
        Windows Event Log                     |
              |                               |
        UF reads event                        |
              |                               |
        UF ships via TCP 9997          UDP Syslog port 514
              |                               |
              └───────────┬───────────────────┘
                          |
                   SPLUNK RECEIVES
                          |
                   Props.conf parsing
                   Sourcetype assignment
                   Timestamp normalization
                          |
                   Transforms.conf routing
                   Index assignment
                          |
                   INDEX WRITTEN
                   (Hot bucket → Warm → Cold → Frozen)
                          |
                   SEARCHABLE IN SPLUNK
                   Dashboards / Alerts / Investigation
```

---

## Path 1 — Windows Endpoint Log Flow

### Step 1 — Event Generation (Sysmon)

Sysmon monitors system activity in real time via a 
kernel-level driver. When a monitored event occurs 
(process creation, network connection, file write, 
registry change, DNS query), Sysmon immediately 
writes a structured XML event to the Windows 
Event Log channel:

```
Microsoft-Windows-Sysmon/Operational
```

The event contains all relevant context:
- Timestamp (UTC)
- Event type (EventCode)
- Process details (path, hash, command line, parent)
- Network details (source/destination IP and port)
- User context
- Additional event-specific fields

### Step 2 — Windows Event Log Storage

The Sysmon event is stored locally in the Windows 
Event Log subsystem. It is readable via Event Viewer 
or PowerShell:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10
```

Events remain in the local Windows Event Log until 
the log reaches its configured maximum size (at which 
point the oldest events are overwritten). The local 
Event Log is not a reliable long-term store — it is 
a temporary buffer that the Universal Forwarder 
reads from continuously.

### Step 3 — Universal Forwarder Reads Events

The Splunk Universal Forwarder runs as a Windows 
service (`SplunkForwarder`). It continuously monitors 
the configured Event Log channels defined in 
`inputs.conf`:

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = sysmon
disabled = false
renderXml = true
```

The forwarder maintains a "fishbucket" — an internal 
bookmark tracking which events have already been 
forwarded — to ensure events are not duplicated or 
missed across forwarder restarts.

### Step 4 — Universal Forwarder Ships Events

The forwarder ships events to Splunk Enterprise 
over TCP port 9997. The connection is persistent 
and auto-reconnects if interrupted.

The forwarder performs minimal processing:
- Reads the raw event from the Windows Event Log
- Applies `index = sysmon` from inputs.conf
- Compresses the payload (if `compressed = true`)
- Ships to the configured Splunk server

No parsing, field extraction, or filtering occurs 
at the forwarder — this happens at the indexer.

### Step 5 — Splunk Receives and Parses

Splunk Enterprise receives the event on port 9997. 
It applies configuration from `props.conf` to:

- Identify the sourcetype (`XmlWinEventLog:Microsoft-Windows-Sysmon/Operational`)
- Parse the timestamp from the event
- Extract fields from the XML structure (because `renderXml = true` was set), but later set to false because of the **SPL** error when searching.

### Step 6 — Splunk Routes to Index

The `index = sysmon` directive from `inputs.conf` 
routes this event to the `sysmon` index.

### Step 7 — Event Written to Index

The event is written to the `sysmon` index hot bucket. 
It is immediately searchable in Splunk Web.

**Total pipeline latency:** Typically under 30 seconds 
from event occurrence to searchable in Splunk under 
normal network conditions.

---

## Path 2 — Cisco Meraki Network Log Flow

### Step 1 — Event Generation (Meraki)

Cisco Meraki appliances generate events continuously:
- Every web request made by a network client (URLs role)
- Every firewall rule match (Appliance Event log role)
- VPN state changes, routing changes, security events

Meraki is cloud-managed — its syslog configuration 
is set via the Meraki Dashboard (not on the device locally).

### Step 2 — Meraki Sends Syslog

The Meraki appliance sends syslog events via UDP 
to the configured syslog server (the Splunk host) 
on port 514.

```
Protocol: UDP (connectionless — no delivery guarantee)
Port: 514
Format: RFC 3164 syslog with Meraki-specific payload
```

**Important:** UDP syslog has no acknowledgement 
mechanism. If Splunk is unavailable when a Meraki 
event is generated, that event is lost — Meraki 
does not queue or retry. This is a known limitation 
of the direct syslog approach.

### Step 3 — Splunk Receives on Port 514

Docker maps the host port 514 to the container 
port 514. Splunk listens for incoming syslog via 
the `[source::udp:514]` input defined in `props.conf`.

### Step 4 — Sourcetype Assignment and Routing

`props.conf` identifies events arriving on UDP 514 
as `meraki_syslog` and triggers the routing transform:

```ini
[source::udp:514]
TRANSFORMS-routing = route_meraki
sourcetype = meraki_syslog
```

`transforms.conf` routes these events to the 
`meraki` index:

```ini
[route_meraki]
REGEX = .
DEST_KEY = _MetaData:Index
FORMAT = meraki
```

### Step 5 — Timestamp Parsing

`props.conf` defines how Splunk reads the timestamp 
from Meraki's syslog format:

```ini
TIME_PREFIX = ^
TIME_FORMAT = %b %d %H:%M:%S
```

Correct timestamp parsing is critical — events 
indexed with incorrect timestamps appear in the 
wrong time range during searches, making 
investigation unreliable.

### Step 6 — Event Written to Index

The event is written to the `meraki` index. 
It is immediately searchable in Splunk Web.

**Total pipeline latency:** Typically under 10 
seconds for Meraki events — the UDP path has 
less processing overhead than the TCP forwarder path.

---

## Pipeline Health Monitoring

The pipeline can fail silently at multiple points. 
A monitoring check should be run regularly to confirm 
all sources are actively delivering data.

### Source Health Check

```spl
| metadata type=sourcetypes index=*
| eval last_seen=strftime(lastTime, "%Y-%m-%d %H:%M:%S")
| eval hours_since_last_event=round((now()-lastTime)/3600,1)
| table sourcetype, last_seen, hours_since_last_event
| sort -hours_since_last_event
```

If any sourcetype shows `hours_since_last_event` 
greater than 2-4 hours during business hours, 
the pipeline for that source should be investigated.

### Index Event Count Check

```spl
| eventcount summarize=false index=*
| where index!="_internal" AND index!="_audit"
| table index, count
```

All production indexes (meraki, sysmon, windows) 
should show non-zero counts.

### Pipeline Failure Points Reference

| Location | Failure Mode | Detection Method |
|---|---|---|
| Sysmon service | Stopped — no events generated | `Get-Service Sysmon64` on endpoint |
| Forwarder service | Stopped — events not shipped | `Get-Service SplunkForwarder` on endpoint |
| Forwarder ACL | errorCode=5 — Sysmon channel unreadable | splunkd.log on endpoint |
| Network path (9997) | Blocked — forwarder cannot reach indexer | `Test-NetConnection` from endpoint |
| Meraki syslog config | Wrong IP — events sent to wrong destination | Meraki Dashboard verification |
| Docker port mapping | Port not exposed — Splunk not listening | `docker ps` on SOC server |
| Splunk index config | Wrong index name — events to unexpected index | `index=* \| stats count by index` |

---

## Data Volume Estimates

| Source | Events/day (est.) | MB/day (est.) |
|---|---|---|
| Sysmon (16-18 endpoints) | 250,000-400,000 | 1,500-2,700 MB |
| Windows Event Logs (16-18 endpoints) | 50,000-100,000 | 500-900 MB |
| Meraki (2-3 sites, URLs + events) | 80,000-120,000 | 800-1,200 MB |
| **Total** | **380,000-620,000** | **~2.8-4.8 GB** |

These estimates are based on observed production volumes 
in this environment. Actual volumes depend on endpoint 
activity levels, Sysmon configuration verbosity, and 
Meraki URL logging volume.

---

## Retention and Lifecycle

Once events reach their configured index, the data 
lifecycle is managed by Splunk's bucket aging process:

```
Day 0-N:   HOT bucket  — active writes, fast storage
Day N-M:   WARM bucket — read-only, recent searches
Day M-X:   COLD bucket — aged data, slower storage
Day X+:    FROZEN      — deleted (or archived)
```

Where X is determined by:
- `frozenTimePeriodInSecs` (time-based limit), OR
- `maxTotalDataSizeMB` (size-based limit)
- Whichever is reached first

Current retention configuration:

| Index | Time Limit | Size Limit |
|---|---|---|
| sysmon | 90 days | 100 GB |
| windows | 365 days | 50 GB |
| meraki | 90 days | 50 GB |

---

## Lessons Learned

**Silent failures are the hardest to diagnose:** 
Multiple pipeline failures in this deployment failed 
silently — no errors in Splunk, the dashboard just 
showed no data. The diagnostic approach developed 
through experience: work backward from Splunk to 
the source, verifying each stage independently.

**The fishbucket can get confused after Sysmon 
config changes:** When Sysmon's configuration 
was updated (applying the Olaf Hartong config), 
the Sysmon service briefly restarted. The 
Universal Forwarder's internal bookmark for 
the Sysmon event log channel lost its position, 
and forwarding did not resume automatically. 
Restarting the forwarder service after a 
Sysmon config change resolves this.

**UDP has no delivery guarantee:** Meraki's 
syslog uses UDP — if the Splunk server is 
unavailable, those events are lost permanently. 
For environments requiring continuous log 
availability, an intermediate syslog collector 
with disk buffering would be appropriate.

---

> **Confidentiality Note:** All IP addresses, 
> hostnames, and volume figures are approximations 
> or placeholders. Actual environment specifics 
> are not documented in this repository.
