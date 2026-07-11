# SOC Architecture Overview
## Enterprise Security Operations Centre — System Design

---

## Architecture Summary

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

## Component Descriptions

### VMware ESXi (Hypervisor Layer)
Bare-metal hypervisor running on dedicated SOC hardware. Hosts all SOC virtual machines in an isolated environment.

### Rocky Linux (OS Layer)
RHEL-compatible enterprise Linux distribution. Hardened to CIS Benchmark standards before production use. Hosts Docker Engine and the Splunk container.

### Docker + Splunk Enterprise (SIEM Layer)
Splunk deployed as a container with persistent named volumes. Provides log ingestion, indexing, detection, and dashboards.

### Cisco Meraki (Network Layer)
Cloud-managed firewall and switching infrastructure. Forwards syslog events to Splunk for network visibility.

### Windows Endpoints (Endpoint Layer)
Monitored Windows workstations running Sysmon +
Splunk Universal Forwarder for endpoint telemetry.

### Windows Server 2025 RODC (Identity Layer)
Read-Only Domain Controller — deliberately chosen over
a writable DC to limit the impact of a potential
SOC server compromise.

---

## Design Principles

| Principle | Implementation |
|---|---|
| Separation of concerns | Each component has one defined role |
| Least privilege | RODC, restricted firewall, minimal port exposure |
| Defense in depth | Network + OS + application layer controls |
| Operational resilience | Persistent volumes, auto-restart policy |

---

## Port Reference

| Port | Protocol | Direction | Purpose |
|---|---|---|---|
| 514 | UDP | Inbound | Meraki syslog receive |
| 8000 | TCP | Inbound | Splunk Web UI |
| 9997 | TCP | Inbound | Universal Forwarder data |
| 22 | TCP | Inbound | SSH admin access |
| 8089 | TCP | Internal only | Splunk management (not exposed) |

---

> **Confidentiality Note:** All IP addresses, hostnames,
> and network topology specifics have been sanitized.
