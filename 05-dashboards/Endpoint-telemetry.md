# Endpoint Telemetry Dashboard
## Dashboard Studio Build Guide — Sysmon Event Code Monitoring

---

## Purpose

Detailed visibility into Windows endpoint activity 
via Sysmon event codes. Organized by detection theme 
rather than event code number.

---

## Dashboard Settings
```
Title:       Endpoint Telemetry (Sysmon)
Framework:   Dashboard Studio (Grid layout)
Time Token:  global_time (Last 24 hours default)
```

---

## Panel Organization

```
Section 1 — Process Activity (EventCode 1, 5)
Section 2 — Network Activity (EventCode 3, 22)
Section 3 — File & Persistence (EventCode 11, 12, 13, 14, 7)
Section 4 — Summary (All EventCodes)
```

---

## Section 1 — Process Activity

### Process Creation Volume
```spl
index=sysmon EventCode=1
earliest=$global_time.earliest$ latest=$global_time.latest$
| timechart span=1h count
```

### Top Processes Executed
```spl
index=sysmon EventCode=1
earliest=$global_time.earliest$ latest=$global_time.latest$
| top limit=10 Image
```

### Suspicious Parent-Child Relationships
```spl
index=sysmon EventCode=1
ParentImage IN (
  "*\\winword.exe","*\\excel.exe",
  "*\\powerpnt.exe","*\\outlook.exe"
)
| table _time, host, User, ParentImage, Image, CommandLine
```

---

## Section 2 — Network Activity

### Network Connection Volume
```spl
index=sysmon EventCode=3
earliest=$global_time.earliest$ latest=$global_time.latest$
| timechart span=1h count
```

### Top DNS Queries
```spl
index=sysmon EventCode=22
earliest=$global_time.earliest$ latest=$global_time.latest$
| top limit=10 QueryName
```

---

## Section 3 — File and Persistence

### Files in Sensitive Locations
```spl
index=sysmon EventCode=11
TargetFilename IN (
  "*\\Startup\\*","*\\Temp\\*.exe","*\\AppData\\*.exe"
)
| table _time, host, Image, TargetFilename
```

### Registry Persistence Keys
```spl
index=sysmon EventCode=13
TargetObject IN (
  "*\\CurrentVersion\\Run\\*",
  "*\\CurrentVersion\\RunOnce\\*"
)
| table _time, host, User, Image, TargetObject, Details
```

### Unsigned DLLs Loaded
```spl
index=sysmon EventCode=7 Signed=false
earliest=$global_time.earliest$ latest=$global_time.latest$
| table _time, host, Image, ImageLoaded, Signed
```

---

## Section 4 — Summary

### Event Volume by EventCode
```spl
index=sysmon
earliest=$global_time.earliest$ latest=$global_time.latest$
| stats count by EventCode
| sort -count
```

---

> See [Sysmon.md](../docs/08-Sysmon.md) and 
> [SPL-Queries.md](../docs/14-SPL-Queries.md) for 
> full event code reference and query library.
