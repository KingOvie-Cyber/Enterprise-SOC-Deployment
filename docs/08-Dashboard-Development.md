# Dashboard Development
## Splunk Dashboard Studio — Design, Build, and Operational Use

---

## Introduction

This document describes the dashboard development process 
for the Enterprise SOC. It covers the dashboard design 
philosophy, the four operational dashboards built, the 
technical implementation approach using Splunk Dashboard 
Studio, and lessons learned during development.

Dashboards are the primary interface between raw log data 
and SOC operational awareness. A well-designed dashboard 
enables an analyst to assess the security posture of the 
environment at a glance — and to drill into anomalies 
without writing a new search from scratch.

---

## Dashboard Framework Selection

### Dashboard Studio vs Classic Dashboards

All dashboards in this SOC were built using Splunk 
Dashboard Studio — not the legacy Simple XML / Classic 
Dashboard framework.

| Aspect | Dashboard Studio | Classic (Simple XML) |
|---|---|---|
| Status | Current standard | Legacy / deprecated path |
| Layout | Grid-based, drag-and-drop | Absolute pixel placement |
| Interactivity | Token-based, dynamic | Limited |
| Visualization options | Rich, modern | Basic |
| Career relevance | High | Declining |

The decision to use Dashboard Studio exclusively 
reflects a commitment to building skills on the 
current and future Splunk standard, not the 
deprecated approach.

---

## Design Philosophy

### Hierarchy of Information

Every dashboard follows a consistent information hierarchy:

```
Level 1: Summary metrics (single values at a glance)
         → "What is the overall state right now?"

Level 2: Trends over time (line charts)
         → "Is anything changing or spiking?"

Level 3: Distribution (pie/bar charts)
         → "Where is activity concentrated?"

Level 4: Detail (tables with raw event data)
         → "What specifically happened?"
```

This hierarchy matches how analysts actually use dashboards — 
starting with a quick health check and drilling down only 
when something looks anomalous.

### Audience-Appropriate Design

Two distinct dashboard audiences were identified:

**Analyst-facing dashboards (SOC Overview, Endpoint 
Telemetry, Meraki Network Monitoring):**
- Default time range: Last 24 hours
- Emphasis on real-time data and specific event detail
- Tables with raw event data for immediate investigation
- Designed for continuous monitoring use

**Compliance-facing dashboards (ISO 27001 Controls):**
- Default time range: Last 30 days
- Emphasis on sustained control operation evidence
- Trend charts showing control consistency over time
- Designed for audit review and evidence export

---

## Dashboard 1 — SOC Overview

### Purpose
General visibility across all three log sources 
(Sysmon, Windows, Meraki) in a single view. 
The primary "health check" dashboard for daily 
SOC operations.

### Panels

| Panel | Visualization | SPL Summary |
|---|---|---|
| Total Events Today | Single Value | `index=* \| stats count` |
| Active Hosts (24h) | Single Value | `dc(host)` across sysmon and windows |
| Event Distribution by Source | Pie Chart | `stats count by index` |
| Failed Logins Today | Single Value | `EventCode=4625 \| stats count` |
| Events Over Time | Line Chart | `timechart span=1h count by index` |
| Recent Process Creations | Table | Sysmon EventCode=1, last 1 hour |

### Key Design Decisions

**Color coding for single value panels:**
- Total Events: Blue (neutral — informational)
- Active Hosts: Green (availability indicator)
- Failed Logins: Red (security alert indicator)

Color choice is deliberate — an analyst glancing 
at the dashboard should immediately notice if 
Failed Logins is elevated without reading the number.

**Pie chart for source distribution:**
The pie chart makes it immediately visible if one 
source is generating a disproportionate share of 
events — which could indicate a configuration 
problem (one source flooding the SIEM) or a 
monitoring gap (one source unexpectedly silent).

---

## Dashboard 2 — Endpoint Telemetry (Sysmon)

### Purpose
Detailed visibility into Windows endpoint activity 
via Sysmon event codes. Organized by detection theme 
rather than event code number — reflecting how 
analysts think during investigation.

### Panel Organization

```
Section 1 — Process Activity
├── Process creation volume over time (Line Chart)
├── Top processes executed (Table)
├── Recent process creations with command lines (Table)
└── Suspicious parent-child relationships (Table)

Section 2 — Network Activity
├── Network connection volume over time (Line Chart)
├── Top destination IPs and ports (Table)
├── Recent network connections with process context (Table)
└── Top DNS queries (Table)

Section 3 — File and Persistence Activity
├── File creation volume over time (Line Chart)
├── Files created in sensitive locations (Table)
├── Recent registry modifications (Table)
└── Registry persistence key changes (Table)

Section 4 — Summary
└── Event volume by EventCode (Bar/Pie Chart)
```

### Key Design Decisions

**Organized by threat theme, not event code number:**
Grouping EventCode 1, 5 (process), EventCode 3, 22 
(network), EventCode 11, 12, 13, 14 (file/persistence) 
by theme rather than number makes the dashboard 
usable during an incident without requiring the 
analyst to remember which number corresponds to 
which event type.

**Parent-child relationship panel:**
The most specific and high-value detection panel 
on this dashboard — Office applications spawning 
shell processes is the most reliable indicator of 
macro-based malware execution and warrants its 
own dedicated visibility.

---

## Dashboard 3 — Meraki Network Monitoring

### Purpose
Network traffic visibility across all monitored 
sites via Cisco Meraki syslog data.

### Panels

| Panel | Visualization | Key Insight |
|---|---|---|
| Total Meraki Events | Single Value | Overall data flow health |
| Security Event Types | Pie Chart | VPN, routing, FIPS events breakdown |
| Event Volume Over Time | Line Chart | Traffic pattern baseline |
| Top Visited Domains | Table | Normal vs. anomalous destinations |
| Top Talkers | Table | Devices generating most traffic |
| Top Destination Ports | Table | Port usage baseline |
| DNS-over-HTTPS Usage | Table | Encrypted DNS visibility gap |

### Key Design Decisions

**DNS-over-HTTPS panel:**
This panel was added specifically to monitor a 
security gap discovered during deployment — 39 
devices using encrypted DNS that bypasses standard 
DNS monitoring. The panel exists not because DoH 
is inherently malicious, but because it creates 
a blind spot the SOC needs to track and manage.

**Top Talkers panel:**
Source IP ranking by URL request volume quickly 
surfaces anomalous devices. A workstation generating 
10x the web request volume of its peers warrants 
investigation — this could indicate automated 
activity, malware beaconing, or unusual user behavior.

---

## Dashboard 4 — ISO 27001 Technological Controls Evidence

### Purpose
Audit-facing dashboard providing exportable evidence 
that ISO 27001 Annex A Technological Controls are 
operational and continuously monitored.

**Important scope note:** This dashboard covers 
the Technological control theme only. ISO 27001 
has four control themes — Organizational, People, 
Physical, and Technological. The first three 
require policy documentation, HR records, and 
physical access controls respectively — none of 
which are captured in a SIEM.

### Control Mapping

| Panel | Control | Evidence Provided |
|---|---|---|
| Active Log Sources | A.8.15 Logging | Proof logging is active |
| Log Retention Policy | A.8.15 Logging | Proof retention is configured |
| Monitoring Source Health | A.8.16 Monitoring | Proof monitoring is continuous |
| Failed Login Trend (30 days) | A.5.15 Access Control | Sustained access monitoring |
| Privileged Group Changes | A.5.18 Access Rights | Privilege change tracking |
| Malware Detection Activity | A.8.7 Protection Against Malware | Detection coverage proof |
| Configuration Change Volume | A.8.9 Configuration Management | Change tracking evidence |
| Detection Rule Inventory | A.5.7 Threat Intelligence | Detection coverage documentation |
| Storage Utilization | A.8.6 Capacity Management | Capacity monitoring proof |
| Total Monitored Sources | A.8.16 Monitoring | Scope statement |

### Key Design Decisions

**Static time range for some panels:**
Panels using `eventcount` (historical totals) 
and `rest` (live configuration data) do not 
use time-bound queries — they use the `Static` 
time range setting to decouple from the global 
time picker. This prevents the "Set token value 
to render visualization" error that occurs when 
non-time-bounded searches are wired to a time 
range token.

**Export capability:**
The dashboard has the export button enabled, 
allowing PDF export directly from the Splunk 
Web UI. This is the mechanism for producing 
audit evidence documents — an assessor can 
request a current-state evidence export and 
receive it immediately.

---

## Technical Implementation Notes

### Token-Based Time Range Interactivity

All time-bounded dashboards use a shared global 
time range input:

```
Token name: global_time
Default: Last 24 hours
```

Each panel's SPL references the token:
```spl
earliest=$global_time.earliest$ latest=$global_time.latest$
```

This enables a single time picker to update all 
panels simultaneously — a critical usability 
feature for incident investigation where the 
analyst needs to change the time window across 
all panels at once.

### Common Dashboard Studio Pitfalls

**"Set token value to render visualization" error:**
Caused when a panel's visualization is bound to a 
time token but the panel's SPL does not reference 
that token. Fix: either add the token reference 
to the SPL, or set the panel's Time range to 
`Static` if the query is not time-bounded.

**Adding a tab instead of a panel:**
Dashboard Studio presents "Add a tab" as the 
primary action — but panels are added via the 
toolbar icons, not the tab system. Tabs are for 
separate dashboard pages, not additional panels 
on the same page. This distinction caused 
significant rework during initial dashboard 
construction.

**Duplicate time range inputs:**
Clicking "Add Input" twice accidentally creates 
two time range pickers with different token names. 
Panels bound to the second (incorrect) token 
show no data. Delete the duplicate and verify 
all panels use the single `global_time` token.

---

## Dashboard Screenshots

Screenshots of all operational dashboards are 
available in the `/screenshots/` directory:

```
screenshots/
├── SOC-Overview-Dashboard.png
├── Endpoint-Telemetry-Dashboard.png
├── Meraki-Network-Monitoring-Dashboard.png
└── ISO-27001-Controls-Dashboard.png
```

> Note: Screenshots have been reviewed to ensure 
> no organization-specific identifiers are visible 
> before being added to this repository.

---

## Lessons Learned

**Build in Search & Reporting first:** Every SPL 
query was tested and confirmed working in Search 
& Reporting before being added to a dashboard panel. 
Attempting to debug broken queries inside a 
dashboard panel is significantly more difficult 
than debugging in the search interface.

**One panel at a time:** Each panel was added, 
confirmed rendering correctly, and then the next 
was added. Building all panels first and then 
troubleshooting is much harder than verifying 
incrementally.

**Layout last:** Panel resizing and layout 
arrangement was done after all panels were 
confirmed working — not during construction. 
Mixing layout work with functionality debugging 
adds unnecessary confusion.

**The ISO 27001 dashboard teaches audit thinking:** 
Building the compliance dashboard required 
genuinely understanding what an auditor needs 
to see — not just what data exists. The distinction 
between "evidence that controls are configured" 
and "evidence that controls are continuously 
operating" shaped the panel design significantly.

---

> **Confidentiality Note:** Dashboard screenshots 
> in the /screenshots/ directory have been reviewed 
> to ensure no organization-specific identifiers, 
> IP addresses, or sensitive security findings are 
> visible. Panel data shown is representative of 
> the monitoring capability, not a disclosure of 
> the organization's security posture.
