# Splunk
## SIEM Configuration, Index Architecture, and Data Management

---

## Introduction

This document describes the Splunk Enterprise configuration 
used in the Enterprise SOC deployment. It covers index 
architecture, sourcetype design, parsing configuration, 
routing logic, receiving setup, and data lifecycle management.

Splunk Enterprise serves as the central platform for all 
log ingestion, parsing, detection, visualization, and 
compliance evidence generation in this environment.

---

## Splunk Deployment Overview

```
Platform:        Splunk Enterprise
Deployment type: Docker container (Rocky Linux host)
Access:          Web UI via TCP 8000
Data receive:    TCP 9997 (forwarder), UDP 514 (syslog)
Version:         10.4.0 (as deployed)
License:         Free tier (500MB/day) — Enterprise license 
                 procurement in progress
```

---

## Index Architecture

### Design Principles

Indexes were designed with three considerations:

**Source separation:** Each distinct data source has its 
own index. This enables source-specific retention policies, 
access controls, and search performance optimization.

**Retention differentiation:** High-volume sources with 
shorter investigative value (Sysmon, Meraki) use shorter 
retention. Compliance-relevant sources (Windows Security 
logs) use longer retention to support audit cycles.

**Size capping:** Each index has a maximum size cap to 
prevent any single source from consuming all available 
disk storage.

### Index Definitions

| Index | Source | Retention | Max Size | Rationale |
|---|---|---|---|---|
| `sysmon` | Windows Sysmon telemetry | 90 days | 100 GB | High volume, 90-day investigation window sufficient |
| `windows` | Windows Event Logs | 365 days | 50 GB | Compliance-relevant, annual audit cycle |
| `meraki` | Cisco Meraki syslog | 90 days | 50 GB | High volume, network events |
| `main` | Default catch-all | 30 days | 10 GB | Unclassified events |

### indexes.conf

```ini
[meraki]
homePath = $SPLUNK_DB/meraki/db
coldPath = $SPLUNK_DB/meraki/colddb
thawedPath = $SPLUNK_DB/meraki/thaweddb
maxTotalDataSizeMB = 50000
frozenTimePeriodInSecs = 7776000
# 90 days

[sysmon]
homePath = $SPLUNK_DB/sysmon/db
coldPath = $SPLUNK_DB/sysmon/colddb
thawedPath = $SPLUNK_DB/sysmon/thaweddb
maxTotalDataSizeMB = 100000
frozenTimePeriodInSecs = 7776000
# 90 days

[windows]
homePath = $SPLUNK_DB/windows/db
coldPath = $SPLUNK_DB/windows/colddb
thawedPath = $SPLUNK_DB/windows/thaweddb
maxTotalDataSizeMB = 50000
frozenTimePeriodInSecs = 31536000
# 365 days
```

### Data Lifecycle (Hot → Warm → Cold → Frozen)

```
Event arrives
     |
  [HOT]      Active writing — fast storage, SSD preferred
     |
  [WARM]     Recent, read-only — still in homePath
     |
  [COLD]     Aged data — moved to coldPath, slower storage
     |
  [FROZEN]   Expired — deleted (or archived if thawedPath configured)
```

Understanding this lifecycle is important for storage 
planning. The `maxTotalDataSizeMB` cap triggers data 
aging regardless of whether the time-based retention 
period has been reached — whichever limit is hit first 
causes the oldest data to roll to frozen.

---

## Sourcetype Design

Sourcetypes define how Splunk parses and interprets 
incoming log data. Correct sourcetype configuration 
is essential for accurate field extraction, timestamp 
parsing, and search performance.

### Sourcetypes in Use

| Sourcetype | Source | Format |
|---|---|---|
| `meraki_syslog` | Cisco Meraki appliance | Syslog key-value pairs |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | Sysmon | XML |
| `WinEventLog:Security` | Windows Security Event Log | Windows Event Log |
| `WinEventLog:System` | Windows System Event Log | Windows Event Log |
| `WinEventLog:Application` | Windows Application Event Log | Windows Event Log |

---

## Parsing Configuration (props.conf)

```ini
[meraki_syslog]
SHOULD_LINEMERGE = false
TIME_PREFIX = ^
TIME_FORMAT = %b %d %H:%M:%S
MAX_TIMESTAMP_LOOKAHEAD = 32

[source::udp:514]
TRANSFORMS-routing = route_meraki
sourcetype = meraki_syslog
```

### Configuration Explained

**SHOULD_LINEMERGE = false**
Prevents Splunk from incorrectly joining multiple Meraki 
log lines into a single event. Each syslog line is a 
discrete event.

**TIME_FORMAT**
Tells Splunk exactly how to parse Meraki's timestamp format. 
Incorrect timestamp parsing causes events to appear at the 
wrong time in searches and dashboards — one of the most 
common and disruptive parsing errors in SIEM deployments.

**TRANSFORMS-routing**
Links this source to the routing rule in transforms.conf 
that directs events to the correct index.

---

## Routing Configuration (transforms.conf)

```ini
[route_meraki]
REGEX = .
DEST_KEY = _MetaData:Index
FORMAT = meraki
```

This routing rule captures all events arriving on UDP 514 
(already identified as `meraki_syslog` via props.conf) 
and routes them to the `meraki` index.

The `REGEX = .` matches any non-empty string — effectively 
routing all events from this source to the meraki index 
without additional filtering.

---

## Receiving Configuration

### Forwarder Receiving (Port 9997)

Configured via Splunk Web:
```
Settings → Forwarding and receiving
→ Configure receiving → New Receiving Port → 9997
```

This enables Splunk to accept data from Splunk Universal 
Forwarders installed on Windows endpoints.

### Syslog Receiving (Port 514)

Configured via Docker port mapping in docker-compose.yml:
```yaml
ports:
  - "514:514/udp"
```

Splunk receives syslog data directly on UDP 514. No 
intermediate syslog collector (rsyslog/syslog-ng) is 
used at the current scale.

---

## Field Extraction for Meraki URL Logs

Meraki URL logs require regex-based field extraction 
at search time because they use a space-delimited format 
rather than standard key=value pairs throughout.

### URL Extraction

```spl
index=meraki sourcetype=meraki_syslog "urls"
| rex field=_raw "src=(?<src_ip>[^:]+):(?<src_port>\d+)\s+dst=(?<dst_ip>[^:]+):(?<dst_port>\d+)\s+mac=(?<mac>\S+)\s+request:\s+\S+\s+(?<url>https?://\S+)"
| table _time, src_ip, src_port, dst_ip, dst_port, mac, url
```

### Domain-Only Extraction (for Top Domains analysis)

```spl
index=meraki sourcetype=meraki_syslog "urls"
| rex field=_raw "request:\s+\S+\s+(?<url>https?://[^\s/]+)"
| top limit=10 url
```

### Source IP Extraction

```spl
index=meraki sourcetype=meraki_syslog "urls"
| rex field=_raw "src=(?<src_ip>[^:]+):"
| top limit=10 src_ip
```

---

## License Management

### Current License State

```
License type:     Free (development/evaluation)
Daily cap:        500 MB/day
Current usage:    ~3-5 GB/day estimated
Status:           Exceeds free tier — Enterprise 
                  license procurement in progress
```

### Monitoring License Usage

```
Settings → Licensing → Usage report
```

This shows daily ingestion against the license cap. 
Critical for planning the Enterprise license purchase 
at the correct volume tier.

### License Violation Behavior

When the free tier 500MB/day cap is exceeded, Splunk 
does NOT charge an overage — it simply **stops indexing** 
new data for the remainder of that calendar day. This 
creates a visibility gap: events are dropped, not queued.

This is why the free tier is not suitable for production 
security monitoring at this environment's data volume.

---

## Performance and Capacity Planning

### Estimated Daily Data Volume

| Source | Endpoints | Avg MB/day per host | Total |
|---|---|---|---|
| Sysmon | 16-18 | ~100 MB | ~1.8 GB |
| Windows Event Logs | 16-18 | ~50 MB | ~0.9 GB |
| Meraki (2-3 sites) | N/A | ~500 MB per site | ~1.5 GB |
| **Total** | | | **~4.2 GB/day** |

### Storage Planning

At ~4 GB/day:
```
sysmon index (90 days):    ~360 GB (capped at 100 GB — 
                            data ages faster than time limit)
windows index (365 days):  ~330 GB (capped at 50 GB)
meraki index (90 days):    ~135 GB (capped at 50 GB)
```

The `maxTotalDataSizeMB` caps prevent storage runaway. 
When caps are reached, oldest data is frozen (deleted) 
to make room for incoming data.

---

## Splunk Administration Quick Reference

### Health Checks

```spl
-- Check all indexes and event counts
| eventcount summarize=false index=*
| table index, count

-- Check log source health (last seen times)
| metadata type=sourcetypes index=*
| eval last_seen=strftime(lastTime, "%Y-%m-%d %H:%M:%S")
| eval hours_since=round((now()-lastTime)/3600,1)
| table sourcetype, last_seen, hours_since
| sort -hours_since

-- Check license usage
| rest /services/licenser/pools
| table title, used_bytes, effective_quota
```

### Index Size Check

```spl
| dbinspect index=*
| stats sum(sizeOnDiskMB) as size_mb by index
| eval size_gb=round(size_mb/1024,2)
| sort -size_mb
```

---

## Lessons Learned

**Index naming matters:** Using clear, descriptive index 
names (`sysmon`, `windows`, `meraki`) makes SPL queries 
readable and self-documenting. Avoid generic names like 
`index1` or `data`.

**Sourcetype consistency within indexes:** Mixing 
inconsistent sourcetypes in one index makes parsing 
and field extraction unpredictable. Each index should 
have well-defined, consistently-parsed sourcetypes.

**Timestamp parsing is critical:** Incorrect 
`TIME_FORMAT` in props.conf causes events to be indexed 
at the wrong time. This makes time-based searches 
unreliable and dashboards misleading. Always verify 
timestamps after onboarding a new data source.

**Test before tuning:** Every SPL query was tested in 
Search and Reporting before being added to a dashboard 
panel. Building dashboards with untested queries leads 
to cascading issues that are harder to debug.

---

> **Confidentiality Note:** All organization-specific 
> values including IP addresses, hostnames, index sizes, 
> and data volumes are approximations or placeholders. 
> Actual environment specifics are not documented here.
