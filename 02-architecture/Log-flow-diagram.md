# Log Flow Diagram
## Data Pipeline — Source to Indexed Event

---

## End-to-End Log Flow

<img width="1472" height="1800" alt="image" src="https://github.com/user-attachments/assets/f645d4ab-8045-415e-9b8c-0381d4be9231" />

---

## Field Extraction Points

| Data Source | Extraction Method | Where It Happens |
|---|---|---|
| Sysmon (XML) | Automatic (renderXml=true) | Forwarder + Splunk |
| Windows Event Log | Automatic (built-in) | Splunk |
| Meraki URLs | Custom regex (rex command) | Search time |
| Meraki Events | Automatic (key=value) | Splunk |

---

## Pipeline Latency

| Path | Typical Latency |
|---|---|
| Endpoint → Splunk (TCP) | Under 30 seconds |
| Meraki → Splunk (UDP) | Under 10 seconds |

---

## Pipeline Failure Points

| Stage | Failure Mode | Detection |
|---|---|---|
| Sysmon service | Not running — no events generated | `Get-Service Sysmon64` |
| UF service | Not running — events not shipped | `Get-Service SplunkForwarder` |
| UF ACL | errorCode=5 — channel unreadable | splunkd.log |
| Network (9997) | Blocked — can't reach indexer | `Test-NetConnection` |
| Meraki config | Wrong IP — events to wrong host | Meraki dashboard |
| Docker port | Not exposed — Splunk not listening | `docker ps` |
| Index config | Wrong name — events misdirected | `index=* \| stats count by index` |
