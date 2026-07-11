# Log Flow Diagram
## Data Pipeline — Source to Indexed Event

---

## End-to-End Log Flow

```
ENDPOINT PATH
─────────────────────────────────────────────────
Windows Endpoint
    │
    ├── Sysmon → Microsoft-Windows-Sysmon/Operational
    └── Windows → Security / System / Application
                        │
              Splunk Universal Forwarder
              reads event log channels
              inputs.conf: index=sysmon / index=windows
                        │
                  TCP Port 9997
                        │
                        ▼

NETWORK PATH
─────────────────────────────────────────────────
Cisco Meraki Appliance
    │
    ├── Appliance Event Log (firewall, VPN, security)
    └── Appliance URLs (web requests)
                        │
                  UDP Syslog Port 514
                        │
                        ▼

SPLUNK RECEIVES (Both Paths)
─────────────────────────────────────────────────
                        │
              props.conf — parse timestamp
              props.conf — assign sourcetype
              transforms.conf — route to index
                        │
              ┌─────────┴──────────┐
              ▼                    ▼
        [sysmon index]      [meraki index]
        [windows index]
              │
    Hot → Warm → Cold → Frozen
    (Data lifecycle managed by indexes.conf)
              │
              ▼
    SEARCHABLE IN SPLUNK
    Dashboards / Alerts / Investigation
```

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
