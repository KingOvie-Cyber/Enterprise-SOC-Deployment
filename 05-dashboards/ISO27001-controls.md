# ISO 27001 Technological Controls Evidence Dashboard
## Dashboard Studio Build Guide

---

## Purpose

Audit-facing dashboard providing exportable evidence 
that ISO 27001 Annex A Technological Controls are 
operational and continuously monitored.

**Scope:** Technological controls only (Annex A).
Organizational, People, and Physical controls are 
documented separately through policy documentation.

---

## Dashboard Settings
```
Title:       ISO 27001 Technological Controls Evidence
Framework:   Dashboard Studio (Grid layout)
Time Token:  global_time (Last 30 days default — 
             compliance view, not real-time)
```

---

## Control Mapping

| Panel | ISO 27001 Control | Evidence Provided |
|---|---|---|
| Active Log Sources | A.8.15 Logging | Proof logging is active |
| Log Retention Policy | A.8.15 Logging | Configured retention per index |
| Monitoring Source Health | A.8.16 Monitoring | Last seen times per sourcetype |
| Failed Login Trend | A.5.15 Access Control | 30-day authentication monitoring |
| Privileged Group Changes | A.5.18 Access Rights | Privilege change tracking |
| Malware Detection Activity | A.8.7 Protection Against Malware | Endpoint detection coverage |
| Config Change Volume | A.8.9 Configuration Management | Registry change tracking |
| Detection Rule Inventory | A.5.7 Threat Intelligence | Active detection coverage |
| Storage Utilization | A.8.6 Capacity Management | Splunk storage monitoring |
| Total Monitored Sources | A.8.16 Monitoring | Monitoring scope summary |

---

## Key Panel SPL Queries

### Active Log Sources (A.8.15)
```spl
| eventcount summarize=false index=*
| where index!="_internal" AND index!="_audit" 
  AND index!="_introspection"
| table index, count
```
*Time range: Static (all-time totals)*

### Log Retention Policy (A.8.15)
```spl
| rest /services/data/indexes
| search title!="_*"
| table title, frozenTimePeriodInSecs
| eval retention_days=round(frozenTimePeriodInSecs/86400,0)
| table title, retention_days
| rename title as "Index", retention_days as "Retention (Days)"
```
*Time range: Static*

### Monitoring Source Health (A.8.16)
```spl
| metadata type=sourcetypes index=*
| eval last_seen=strftime(lastTime, "%Y-%m-%d %H:%M:%S")
| eval hours_since_last_event=round((now()-lastTime)/3600,1)
| table sourcetype, last_seen, hours_since_last_event
| sort -hours_since_last_event
```
*Time range: Static*

### Failed Login Trend — 30 Days (A.5.15)
```spl
index=windows EventCode=4625
earliest=-30d latest=now
| timechart span=1d count
```

### Privileged Group Changes — 30 Days (A.5.18)
```spl
index=windows 
(EventCode=4728 OR EventCode=4732 OR EventCode=4756)
earliest=-30d latest=now
| table _time, host, user
| sort -_time
```

---

## Export for Audit

```
Dashboard → Actions → Export as PDF
```

Provides a point-in-time evidence document for auditors.

---

> See [Dashboard-Development.md](../docs/13-Dashboard-Development.md)
> for full build instructions and token wiring details.
