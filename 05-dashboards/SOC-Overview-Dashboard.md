# SOC Overview Dashboard
## Dashboard Studio Build Guide

---

## Purpose
General SOC visibility across all three log sources 
(Sysmon, Windows, Meraki) in a single view. Primary 
health-check dashboard for daily SOC operations.

---

## Dashboard Settings
```
Title:       SOC Dashboard Overview
Framework:   Dashboard Studio (Grid layout)
Time Token:  global_time (Last 24 hours default)
```

---

## Panel Layout

```
Row 1: [Total Events] [Active Hosts] [Event Distribution] [Failed Logins]
Row 2: [Events Over Time — Line Chart — full width]
Row 3: [Recent Process Creations] [Top Processes] [Recent Process Detail]
Row 4: [Process Terminated] [Network Connection Table]
Row 5: [Network Volume] [Recent Connections] [DNS Top Queries]
Row 6: [File Creation Volume] [Sensitive Files] [Registry Events]
Row 7: [Registry Modifications] [Registry Persistence Keys]
Row 8: [DLL Loads] [Unsigned DLLs]
Row 9: [File Creation Time Changed]
```

---

## Panel SPL Queries

### Total Events Today
```spl
index=meraki OR index=sysmon OR index=windows
earliest=$global_time.earliest$ latest=$global_time.latest$
| stats count
```

### Active Hosts (24h)
```spl
index=sysmon OR index=windows
earliest=$global_time.earliest$ latest=$global_time.latest$
| stats dc(host) as active_hosts
```

### Event Distribution by Source
```spl
index=meraki OR index=sysmon OR index=windows
earliest=$global_time.earliest$ latest=$global_time.latest$
| stats count by index
```

### Failed Logins Today
```spl
index=windows EventCode=4625
earliest=$global_time.earliest$ latest=$global_time.latest$
| stats count
```

### Events Over Time
```spl
index=meraki OR index=sysmon OR index=windows
earliest=$global_time.earliest$ latest=$global_time.latest$
| timechart span=1h count by index
```

### Recent Sysmon Process Creations
```spl
index=sysmon EventCode=1
earliest=$global_time.earliest$ latest=$global_time.latest$
| table _time, host, User, Image, CommandLine
| sort -_time
| head 20
```

---

## Color Coding (Single Value Panels)

| Panel | Color | Meaning |
|---|---|---|
| Total Events | Blue | Neutral — informational |
| Active Hosts | Green | Availability indicator |
| Failed Logins | Red | Security alert indicator |

---

## Known Issues and Fixes

**"Set token value to render visualization":**
Ensure SPL references `$global_time.earliest$` and 
`$global_time.latest$` directly in the query.
For non-time-bounded queries (eventcount, rest), 
set Time range to Static in panel settings.

**Duplicate time range input:**
Only one time picker should exist with token name 
`global_time`. Delete any duplicates that appear.

---

> See [13-Dashboard-Development.md](../01-docs/PHASE-5-OPERATIONS/13-Dashboard-Development.md) 
> for full step-by-step build instructions.
