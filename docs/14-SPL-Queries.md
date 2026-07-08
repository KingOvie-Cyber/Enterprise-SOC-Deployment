# SPL Queries
## Search Processing Language Reference Library

---

## Introduction

This document contains the SPL (Search Processing Language) 
query library used in the Enterprise SOC. Queries are 
organized by data source and use case.

All queries in this document have been tested against 
real production data in this environment. They represent 
the actual searches used in dashboards, investigations, 
and detection rules — not theoretical examples.

---

## SPL Fundamentals Used in This Environment

### Core Commands

| Command | Purpose | Example |
|---|---|---|
| `stats` | Aggregate events | `\| stats count by host` |
| `timechart` | Time-based aggregation | `\| timechart span=1h count` |
| `table` | Select and display fields | `\| table _time, host, user` |
| `top` | Most frequent values | `\| top limit=10 Image` |
| `rex` | Regex field extraction | `\| rex field=_raw "src=(?<src_ip>...)"` |
| `where` | Filter results | `\| where count > 5` |
| `sort` | Order results | `\| sort -count` |
| `head` | Limit results | `\| head 20` |
| `eval` | Calculate new fields | `\| eval hours=round(diff/3600,1)` |
| `bucket` | Time bucketing | `\| bucket _time span=5m` |
| `iplocation` | Geo-resolve IPs | `\| iplocation dst_ip` |
| `dc` | Distinct count | `\| stats dc(host) as active_hosts` |
| `metadata` | Index/sourcetype metadata | `\| metadata type=sourcetypes` |

---

## Section 1 — Sysmon Queries

### Process Activity (EventCode 1)

**Process creation volume over time:**
```spl
index=sysmon EventCode=1
earliest=-24h latest=now
| timechart span=1h count
```

**Top processes executed:**
```spl
index=sysmon EventCode=1
earliest=-24h latest=now
| top limit=10 Image
```

**Recent process creations (full detail):**
```spl
index=sysmon EventCode=1
earliest=-1h latest=now
| table _time, host, User, Image, CommandLine, ParentImage
| sort -_time
| head 20
```

**Suspicious parent-child relationships:**
```spl
index=sysmon EventCode=1
ParentImage IN (
  "*\\winword.exe","*\\excel.exe",
  "*\\powerpnt.exe","*\\outlook.exe"
)
| table _time, host, User, ParentImage, Image, CommandLine
| sort -_time
```

**PowerShell execution:**
```spl
index=sysmon EventCode=1
Image="*\\powershell.exe" OR Image="*\\pwsh.exe"
earliest=-24h latest=now
| table _time, host, User, CommandLine, ParentImage
| sort -_time
```

---

### Network Activity (EventCode 3)

**Network connection volume over time:**
```spl
index=sysmon EventCode=3
earliest=-24h latest=now
| timechart span=1h count
```

**Full connection detail (source, destination, port, process):**
```spl
index=sysmon EventCode=3
earliest=-24h latest=now
| stats count as connection_count by 
  host, SourceIp, SourcePort, DestinationIp, DestinationPort
| sort -connection_count
```

**Outbound connections on unusual ports:**
```spl
index=sysmon EventCode=3 Initiated=true
| where NOT (DestinationPort IN (
    80, 443, 53, 22, 25, 587, 143, 993, 8080, 8443
  ))
| stats count by src_ip, DestinationIp, DestinationPort, Image
| sort -count
```

**Top destination IPs:**
```spl
index=sysmon EventCode=3
earliest=-24h latest=now
| top limit=10 DestinationIp
```

---

### DNS Queries (EventCode 22)

**Top DNS queries:**
```spl
index=sysmon EventCode=22
earliest=-24h latest=now
| top limit=10 QueryName
```

**DNS queries by process:**
```spl
index=sysmon EventCode=22
earliest=-24h latest=now
| stats count by Image, QueryName
| sort -count
```

---

### File Activity (EventCode 11)

**Recent files created:**
```spl
index=sysmon EventCode=11
earliest=-1h latest=now
| table _time, host, Image, TargetFilename
| sort -_time
| head 20
```

**Files created in sensitive locations:**
```spl
index=sysmon EventCode=11
TargetFilename IN (
  "*\\Startup\\*","*\\Temp\\*.exe",
  "*\\AppData\\*.exe","*\\AppData\\*.dll"
)
| table _time, host, Image, TargetFilename
| sort -_time
```

**File creation volume over time:**
```spl
index=sysmon EventCode=11
earliest=-24h latest=now
| timechart span=1h count
```

---

### Registry Activity (EventCode 12/13/14)

**Recent registry modifications:**
```spl
index=sysmon (EventCode=12 OR EventCode=13 OR EventCode=14)
earliest=-1h latest=now
| table _time, host, Image, TargetObject, Details
| sort -_time
| head 20
```

**Persistence key modifications (Run/RunOnce):**
```spl
index=sysmon EventCode=13
TargetObject IN (
  "*\\CurrentVersion\\Run\\*",
  "*\\CurrentVersion\\RunOnce\\*"
)
| table _time, host, User, Image, TargetObject, Details
| sort -_time
```

**Registry change volume over time:**
```spl
index=sysmon (EventCode=12 OR EventCode=13 OR EventCode=14)
earliest=-30d latest=now
| timechart span=1d count
```

---

### DLL/Image Load (EventCode 7)

**Recent DLL loads:**
```spl
index=sysmon EventCode=7
earliest=-1h latest=now
| table _time, host, Image, ImageLoaded
| sort -_time
| head 20
```

**Unsigned DLLs loaded:**
```spl
index=sysmon EventCode=7 Signed=false
earliest=-24h latest=now
| table _time, host, Image, ImageLoaded, Signed
| sort -_time
```

---

### High-Value Detection Queries

**LSASS memory access (credential dumping indicator):**
```spl
index=sysmon EventCode=10
TargetImage="*\\lsass.exe"
NOT SourceImage IN (
  "*\\MsMpEng.exe","*\\svchost.exe",
  "*\\wininit.exe","*\\csrss.exe"
)
| table _time, host, SourceImage, TargetImage, GrantedAccess
| sort -_time
```

**Process injection indicator:**
```spl
index=sysmon EventCode=8
earliest=-24h latest=now
| table _time, host, SourceImage, TargetImage
| sort -_time
```

**Timestomping detection:**
```spl
index=sysmon EventCode=2
earliest=-24h latest=now
| table _time, host, Image, TargetFilename, 
  PreviousCreationUtcTime, CreationUtcTime
| sort -_time
```

**Alternate data stream creation:**
```spl
index=sysmon EventCode=15
earliest=-24h latest=now
| table _time, host, Image, TargetFilename
| sort -_time
```

---

### Sysmon Summary Queries

**Event volume by EventCode (all types):**
```spl
index=sysmon
earliest=-24h latest=now
| stats count by EventCode
| sort -count
```

**Event volume by EventCode and host:**
```spl
index=sysmon
earliest=-24h latest=now
| stats count by EventCode, host
| sort EventCode
```

**Coverage health check (which EventCodes are active):**
```spl
index=sysmon
| stats count by EventCode
| append [
    | makeresults
    | eval EventCode="1,2,3,5,7,8,10,11,12,13,14,15,17,18,22"
    | makemv delim="," EventCode
    | mvexpand EventCode
    | eval count=0
  ]
| stats sum(count) as total by EventCode
| sort EventCode
```

---

## Section 2 — Windows Event Log Queries

### Authentication Events

**Failed login spike detection:**
```spl
index=windows EventCode=4625
| bucket _time span=5m
| stats count by _time, src_ip, user
| where count > 5
| table _time, src_ip, user, count
| sort -count
```

**Failed login trend (30 days):**
```spl
index=windows EventCode=4625
earliest=-30d latest=now
| timechart span=1d count
```

**Successful logins:**
```spl
index=windows EventCode=4624
earliest=-24h latest=now
| table _time, host, user, src_ip, Logon_Type
| sort -_time
| head 20
```

---

### Account Management

**New user account created:**
```spl
index=windows EventCode=4720
earliest=-30d latest=now
| table _time, host, user
| sort -_time
```

**User added to privileged group:**
```spl
index=windows EventCode=4732
earliest=-30d latest=now
| table _time, host, user
| sort -_time
```

**Privileged group changes combined:**
```spl
index=windows 
(EventCode=4728 OR EventCode=4732 OR EventCode=4756)
earliest=-30d latest=now
| table _time, host, user
| sort -_time
```

---

## Section 3 — Meraki Network Queries

### URL and Web Traffic

**Top visited domains:**
```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=-24h latest=now
| rex field=_raw "request:\s+\S+\s+(?<url>https?://[^\s/]+)"
| top limit=10 url
```

**Top source devices by web request volume:**
```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=-24h latest=now
| rex field=_raw "src=(?<src_ip>[^:]+):"
| top limit=10 src_ip
```

**Full URL detail (source, destination, URL):**
```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=-24h latest=now
| rex field=_raw "src=(?<src_ip>[^:]+):(?<src_port>\d+)\s+dst=(?<dst_ip>[^:]+):(?<dst_port>\d+)\s+mac=(?<mac>\S+)\s+request:\s+\S+\s+(?<url>https?://\S+)"
| table _time, src_ip, dst_ip, url
| sort -_time
```

**Top destination ports:**
```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=-24h latest=now
| rex field=_raw "dst=[^:]+:(?<dst_port>\d+)"
| top limit=10 dst_port
```

---

### Network Security Events

**Event type breakdown:**
```spl
index=meraki sourcetype=meraki_syslog
earliest=-24h latest=now
| stats count by type
| sort -count
```

**Network event volume over time:**
```spl
index=meraki sourcetype=meraki_syslog
earliest=-24h latest=now
| timechart span=1h count
```

**DNS-over-HTTPS usage detection:**
```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=-24h latest=now
("dns.google" OR "cloudflare-dns.com" OR "doh.opendns.com")
| rex field=_raw "src=(?<src_ip>[^:]+):"
| stats count by src_ip
| sort -count
```

**Geographic destination analysis:**
```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=-24h latest=now
| rex field=_raw "dst=(?<dst_ip>[^:]+):"
| iplocation dst_ip
| top limit=10 Country
```

---

## Section 4 — SOC Health and Compliance Queries

### Pipeline Health

**All index event counts:**
```spl
| eventcount summarize=false index=*
| where index!="_internal" AND index!="_audit" 
  AND index!="_introspection"
| table index, count
```

**Source health check (last seen times):**
```spl
| metadata type=sourcetypes index=*
| eval last_seen=strftime(lastTime, "%Y-%m-%d %H:%M:%S")
| eval hours_since=round((now()-lastTime)/3600,1)
| table sourcetype, last_seen, hours_since
| sort -hours_since
```

**Active hosts (distinct count):**
```spl
index=sysmon OR index=windows
earliest=-24h latest=now
| stats dc(host) as active_hosts
```

### Storage and Capacity

**Index storage utilization:**
```spl
| dbinspect index=*
| stats sum(sizeOnDiskMB) as size_mb by index
| eval size_gb=round(size_mb/1024,2)
| sort -size_mb
```

**Log retention policy:**
```spl
| rest /services/data/indexes
| search title!="_*"
| table title, frozenTimePeriodInSecs
| eval retention_days=round(frozenTimePeriodInSecs/86400,0)
| table title, retention_days
| rename title as "Index", retention_days as "Retention (Days)"
```

### Cross-Source Overview

**Total events across all production indexes:**
```spl
index=meraki OR index=sysmon OR index=windows
earliest=-24h latest=now
| stats count
```

**Events over time across all sources:**
```spl
index=meraki OR index=sysmon OR index=windows
earliest=-24h latest=now
| timechart span=1h count by index
```

---

## SPL Writing Best Practices

**Test before deploying:**
Every query should be tested in Search & Reporting 
before being added to a dashboard panel. Broken 
queries inside a dashboard are harder to debug 
than broken queries in the search interface.

**Use `head` during development:**
Add `| head 100` while developing a query to limit 
result sets and keep searches fast. Remove before 
deploying to production.

**Explicit time ranges:**
Always include `earliest=` and `latest=` in 
production queries to control the time window. 
Relying on the Splunk default time window can 
cause unexpected behavior when time pickers are 
configured.

**Field names are case-sensitive:**
`EventCode` and `eventcode` are not the same field 
in Splunk. Always verify exact field names from 
raw event data before building queries.

**Index before filtering:**
Always specify the index at the start of the search 
to limit Splunk's search scope. Never start a 
production query with `index=*` followed by 
filtering — specify the exact index(es) needed.

---

> **Confidentiality Note:** All SPL queries in 
> this document operate on sanitized or generalized 
> data. No organization-specific field values, 
> IP addresses, or identifiers are embedded 
> in query logic.
