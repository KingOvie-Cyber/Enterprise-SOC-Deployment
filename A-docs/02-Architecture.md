# Architecture
## Enterprise SOC — Infrastructure and System Design

---

## Introduction

This document describes the technical architecture of the Enterprise 
Security Operations Centre. It covers the infrastructure design, 
component relationships, network topology, data flow, and the 
reasoning behind key architectural choices.

Every architectural decision was made deliberately, balancing 
security, operational resilience, performance, and maintainability.

---

## Architecture Principles

The SOC architecture was designed around four core principles:

**1. Separation of Concerns**
Each component has a single, clearly defined responsibility. 
The log collection layer, SIEM layer, and identity layer are 
separate — a failure or compromise of one does not automatically 
cascade to the others.

**2. Least Privilege by Design**
Every component operates with the minimum permissions required 
for its function. This applies to network access (firewall rules), 
OS accounts (no shared admin), and Active Directory (RODC, not 
a writable Domain Controller).

**3. Defense in Depth**
Security controls are layered — network-level controls at the 
perimeter (Meraki), OS-level controls on the server (hardened 
Rocky Linux), and application-level controls within Splunk 
(role-based access).

**4. Operational Resilience**
The architecture is designed to keep log collection running 
even if the SIEM requires maintenance. Persistent Docker volumes 
ensure data survives container restarts. The design avoids 
single points of failure where practical.

---

## High-Level Architecture

```
                    PRODUCTION NETWORK
                           |
           +---------------+---------------+
           |                               |
    [Cisco Meraki]                 [Windows Endpoints]
    Firewall/Switch                Sysmon Telemetry
    VPN Gateway                    Universal Forwarder
    URL Logging                    Windows Event Logs
    Syslog UDP 514                 TCP Port 9997
           |                               |
           +---------------+---------------+
                           |
              +------------+------------+
              |    VMware ESXi Host     |
              |    Rocky Linux (VM)     |
              |    CIS Hardened         |
              |                         |
              |  +-------------------+  |
              |  |  Docker Engine    |  |
              |  |                   |  |
              |  | [Splunk Enterprise]|  |
              |  |  Port 8000 Web UI  |  |
              |  |  Port 9997 Recv    |  |
              |  |  Port 514  Syslog  |  |
              |  |                   |  |
              |  | splunk-var volume  |  |
              |  | splunk-etc volume  |  |
              |  +-------------------+  |
              |                         |
              | [Windows Server 2025]   |
              |  RODC / Active Dir      |
              |  Identity Layer         |
              +-------------------------+
```

---

## Component Architecture

### Layer 1 — Hypervisor (VMware ESXi)

VMware ESXi runs on bare metal on a dedicated SOC server (Lenovo 
ThinkStation). ESXi provides the virtualization layer, enabling 
multiple isolated virtual machines to run on a single physical host.

**Key configuration decisions:**
- SOC VMs are on a dedicated management VLAN, isolated from 
  general user traffic
- VM resource allocation sized for production workload 
  (sufficient RAM and CPU for Splunk's indexing requirements)
- vSwitch configuration reviewed to prevent unintended 
  network exposure

---

### Layer 2 — Operating System (Rocky Linux)

Rocky Linux runs as the guest OS on the primary SOC VM. It serves 
as the host for Docker and the Splunk containerized deployment.

Rocky Linux was selected over alternatives for the following reasons:

| Consideration | Rocky Linux | Ubuntu | CentOS |
|---|---|---|---|
| RHEL compatibility | ✅ Full | ❌ No | ⚠️ EOL |
| Enterprise support lifecycle | ✅ Long | 🟡 Medium | ❌ EOL |
| Production standard | ✅ Yes | 🟡 Partial | ❌ No |
| Community maturity | ✅ Strong | ✅ Strong | ❌ Declining |

---

### Layer 3 — Containerization (Docker)

Splunk Enterprise runs inside a Docker container managed by 
Docker Compose. This decision was made deliberately for the 
following reasons:

**Clean lifecycle management:** Splunk can be upgraded by pulling 
a new image and recreating the container — without touching the 
underlying OS or risking configuration drift.

**Persistent data:** Two named Docker volumes (`splunk-var` and 
`splunk-etc`) ensure all indexed data and configuration persist 
independently of the container lifecycle. Data is not lost if 
the container is restarted, recreated, or upgraded.

**Separation of concerns:** The application (Splunk) is separated 
from the operating system (Rocky Linux). OS-level changes do not 
affect Splunk's configuration, and Splunk configuration does not 
pollute the base OS.

**Port exposure discipline:** Only required ports are mapped 
through Docker to the host — 8000 (Web UI), 9997 (forwarder 
input), and 514 (syslog). The Splunk management port 8089 is 
NOT exposed externally, reducing attack surface.

---

### Layer 4 — SIEM (Splunk Enterprise)

Splunk Enterprise serves as the central platform for log 
ingestion, parsing, indexing, detection, and visualization.

**Index architecture:**

```
meraki    ← Cisco Meraki network/firewall/URL events   (90 days)
sysmon    ← Windows Sysmon endpoint telemetry           (90 days)
windows   ← Windows Event Logs (Security/System/App)   (365 days)
```

Retention periods are deliberately differentiated:
- Sysmon and Meraki: high-volume, shorter retention (90 days)
- Windows Security logs: lower-volume, compliance-relevant, 
  longer retention (365 days) to support audit and 
  investigation requirements

**Sourcetype design:**
- `meraki_syslog` — Cisco Meraki syslog events
- `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` — 
  Sysmon XML-format events
- `WinEventLog:Security` — Windows Security Event Log
- `WinEventLog:System` — Windows System Event Log
- `WinEventLog:Application` — Windows Application Event Log

---

### Layer 5 — Endpoint Telemetry (Sysmon + Universal Forwarder)

Windows endpoints run two components:

**Sysmon** (System Monitor) captures granular endpoint activity 
invisible to standard Windows Event Logs:
- Process creation with full command line (EventCode 1)
- Network connections with process context (EventCode 3)
- File creation events (EventCode 11)
- Registry modifications (EventCode 12/13/14)
- DNS queries (EventCode 22)
- DLL/image loads (EventCode 7)
- And more — see [08-Sysmon.md](08-Sysmon.md) for full coverage

Sysmon configuration is based on the Olaf Hartong modular 
community configuration, tuned to balance detection coverage 
with log volume management.

**Splunk Universal Forwarder** reads from the Sysmon Operational 
event log channel and Windows Event Log channels, then ships 
events to Splunk Enterprise over TCP port 9997.

---

### Layer 6 — Network Telemetry (Cisco Meraki)

Cisco Meraki provides the network security layer across 2-3 
physical sites. Meraki was configured to forward syslog events 
directly to Splunk over UDP port 514.

**Log types collected:**
- Appliance Event Log — firewall events, VPN status, security events
- Appliance URLs — all web requests made by network clients

**Design note:** Meraki's syslog integration sends directly to 
Splunk's exposed port 514 rather than through an intermediate 
syslog collector. This simplifies the architecture for the 
current scale (a separate collector would be added if log 
volume grew substantially or multiple SIEM destinations were needed).

---

### Layer 7 — Identity (Windows Server 2025 / RODC)

A Read-Only Domain Controller (RODC) was deployed on Windows 
Server 2025 to provide identity services within the SOC environment.

**Why RODC and not a full Domain Controller?**

This is a deliberate security architecture decision, not a 
limitation. In a SOC environment:

- If the SOC server were ever compromised, an attacker should 
  gain minimal value from it
- A full writable DC would allow an attacker to modify AD objects, 
  reset passwords, and escalate privileges across the domain
- An RODC holds a read-only copy of AD data — it cannot be used 
  to modify the directory or extract writable credential hashes
- This implements the principle of least privilege at the 
  identity infrastructure level

---

## Data Flow Architecture

### Endpoint Log Flow

```
Windows Endpoint
      |
      ├── Sysmon writes events to:
      │   Windows Event Log: Microsoft-Windows-Sysmon/Operational
      |
      ├── Windows writes events to:
      │   Windows Event Log: Security / System / Application
      |
      └── Splunk Universal Forwarder reads both channels
          │
          │  inputs.conf:
          │  [WinEventLog://Microsoft-Windows-Sysmon/Operational]
          │  index = sysmon
          │  [WinEventLog://Security]
          │  index = windows
          │
          └── TCP Port 9997 → Splunk Enterprise
              → Indexed into sysmon / windows indexes
```

### Network Log Flow

```
Cisco Meraki Appliance
      |
      ├── Appliance Event Log (firewall, VPN, security)
      └── Appliance URLs (web requests)
          |
          └── UDP Syslog Port 514 → Splunk Enterprise
              → props.conf parses sourcetype: meraki_syslog
              → transforms.conf routes to meraki index
              → Indexed into meraki index
```

---

## Network Segmentation

```
SOC Management VLAN (isolated):
├── ESXi host management interface
├── Rocky Linux VM (Splunk server)
└── Windows Server 2025 (RODC)

Production Network:
├── Windows workstation endpoints
└── Cisco Meraki infrastructure

Traffic flows (allowed):
├── Endpoints → Splunk host: TCP 9997 (forwarder data)
├── Meraki → Splunk host: UDP 514 (syslog)
├── Analyst browser → Splunk host: TCP 8000 (Web UI)
└── SSH → Splunk host: TCP 22 (admin only, key-based)

Traffic flows (blocked):
├── Splunk host → endpoints: no outbound access required
├── Splunk management port 8089: not exposed externally
└── All other inbound traffic: dropped by firewalld
```

---

## Security Architecture Summary

| Control | Implementation |
|---|---|
| Network isolation | Dedicated SOC VLAN |
| OS hardening | CIS Benchmark-aligned Rocky Linux config |
| Firewall | firewalld — only required ports open |
| SSH access | Key-based authentication only, root login disabled |
| Container security | Minimal port exposure, non-root where possible |
| Identity | RODC — least privilege at AD layer |
| Data persistence | Named Docker volumes — survive container lifecycle |
| Log retention | Tiered by compliance requirement and volume |

---

## Architecture Limitations and Future Improvements

| Current Limitation | Planned Improvement |
|---|---|
| Single Splunk instance | Distributed indexing as data volume grows |
| No Microsoft 365 integration | Entra ID / M365 logs pending access |
| No vulnerability scanner integration | OpenVAS/Nessus planned |
| Free license (500MB/day cap) | Base Enterprise license procurement in progress |
| Manual alert review | SOAR-lite automation via Splunk alerting |

---

> **Confidentiality Note:** Network topology details, IP addressing, 
> VLAN configurations, and specific infrastructure identifiers have 
> been sanitized. This document describes architectural approach and 
> design decisions only.
