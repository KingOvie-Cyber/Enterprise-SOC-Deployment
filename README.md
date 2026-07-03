# Enterprise SOC Deployment
### Production Security Operations Centre — Built From The Ground Up

![Status](https://img.shields.io/badge/Status-Operational-brightgreen)
![Platform](https://img.shields.io/badge/Platform-VMware%20ESXi-blue)
![SIEM](https://img.shields.io/badge/SIEM-Splunk%20Enterprise-orange)
![OS](https://img.shields.io/badge/OS-Rocky%20Linux-green)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

---

## Project Overview

This repository documents the complete design, deployment, configuration, 
and operation of a production Security Operations Centre (SOC) built 
entirely from the ground up for a live organization.

This is not a home lab, a tutorial replica, or a simulated environment. 
Every architectural decision, infrastructure deployment, security 
hardening measure, and detection rule documented here was designed, 
implemented, tested, and is currently operational in a real production 
environment.

The project demonstrates the full lifecycle of SOC engineering — from 
bare-metal hypervisor deployment through SIEM configuration, log 
pipeline architecture, endpoint telemetry, network security monitoring, 
threat detection, and compliance evidence generation.

---

## Key Features

- Production SOC deployed on VMware ESXi bare metal infrastructure
- Splunk Enterprise SIEM containerized via Docker on Rocky Linux
- Multi-source log ingestion: endpoints, network, and cloud-adjacent sources
- Sysmon endpoint telemetry with tuned detection configuration
- Cisco Meraki network/firewall log integration via syslog
- Custom detection rules mapped to MITRE ATT&CK framework
- Dashboard engineering using Splunk Dashboard Studio
- ISO 27001 Annex A Technological Controls evidence dashboard
- Rocky Linux server hardened to production security standards
- Full log pipeline documented from source to indexed event

---

## Architecture Overview

```
                        PRODUCTION NETWORK
                               |
               +---------------+---------------+
               |                               |
        [Cisco Meraki]                  [Windows Endpoints]
        Firewall/Switch                  Sysmon + UF
        Syslog UDP 514                   Port 9997
               |                               |
               +---------------+---------------+
                               |
                    [Rocky Linux Server]
                    VMware ESXi (Bare Metal)
                    +-----------------------+
                    |  Docker Container     |
                    |  Splunk Enterprise    |
                    |  Port 8000 (Web UI)   |
                    |  Port 9997 (Receive)  |
                    |  Port 514  (Syslog)   |
                    +-----------------------+
                               |
                    +----------+----------+
                    |                     |
              [Dashboards]           [Alerts]
              [Detection]            [Reports]
              [Compliance]           [Evidence]
```

---

## Technology Stack

| Category | Technology | Purpose |
|---|---|---|
| Hypervisor | VMware ESXi | Bare-metal virtualization |
| Operating System | Rocky Linux 9.x | SOC server OS (RHEL-compatible) |
| Containerization | Docker + Docker Compose | Splunk deployment and lifecycle |
| SIEM | Splunk Enterprise | Log ingestion, detection, dashboards |
| Endpoint Monitoring | Sysmon (Olaf Hartong config) | Windows telemetry |
| Endpoint Agent | Splunk Universal Forwarder | Log shipping to SIEM |
| Network Security | Cisco Meraki | Firewall, VPN, URL logging |
| Log Transport | Syslog UDP 514 | Network → SIEM |
| Forwarder Protocol | Splunk Receiving Port 9997 | Endpoint → SIEM |

---

## Skills Demonstrated

### Infrastructure & Virtualization
- VMware ESXi bare-metal deployment and configuration
- Virtual machine provisioning and resource management
- vSwitch and network configuration

### Linux Administration & Hardening
- Rocky Linux deployment and production hardening
- CIS Benchmark-aligned security configuration
- Docker installation and container management
- Systemd service management and audit logging

### SIEM Engineering (Splunk)
- Splunk Enterprise deployment via Docker Compose
- Index architecture and data lifecycle management
- Sourcetype parsing (`props.conf`, `transforms.conf`)
- SPL query development for detection and investigation
- Dashboard Studio development (not legacy Classic/XML)
- Token-based interactive dashboard design

### Detection Engineering
- Custom detection rules mapped to MITRE ATT&CK
- Sysmon-based endpoint threat detection
- Network anomaly detection via Meraki log analysis
- Brute force, persistence, lateral movement detection

### Network Security
- Cisco Meraki syslog integration
- Network traffic baseline establishment
- DNS-over-HTTPS visibility gap identification
- Top-talker and outbound port anomaly monitoring

### Compliance & Governance
- ISO 27001 Annex A Technological Controls mapping
- Splunk-based compliance evidence generation
- Log retention policy design and implementation
- Audit-ready dashboard development

---

## Log Flow

```
ENDPOINT (Windows)
Sysmon → Windows Event Log → Splunk Universal Forwarder
→ TCP Port 9997 → Splunk Enterprise → sysmon/windows index

NETWORK (Cisco Meraki)
Meraki Appliance → UDP Syslog Port 514
→ Splunk Enterprise → meraki index

SPLUNK (Internal)
Splunk → _internal / _audit indexes → Health monitoring
```

---

## Splunk Index Architecture

| Index | Data Source | Retention |
|---|---|---|
| `sysmon` | Windows Sysmon endpoint telemetry | 90 days |
| `windows` | Windows Event Logs (Security/System/App) | 365 days |
| `meraki` | Cisco Meraki syslog events | 90 days |
| `main` | Default catch-all | 30 days |

---

## Dashboard Gallery

| Dashboard | Purpose | Data Sources |
|---|---|---|
| SOC Overview | General SOC visibility at a glance | Sysmon, Windows, Meraki |
| Endpoint Telemetry | Sysmon event code monitoring | Sysmon |
| Meraki Network Monitoring | Network traffic, DNS, top talkers | Meraki |
| ISO 27001 Controls Evidence | Technological controls compliance | All sources |

> Screenshots available in `/screenshots/` directory

---

## Detection Engineering

Custom detection rules developed and operational:

| Detection Rule | MITRE Technique | Data Source |
|---|---|---|
| Brute Force Login Detection | T1110 | Windows Security (EventCode 4625) |
| Malicious Office Macro Execution | T1566 | Sysmon EventCode 1 |
| Outbound C2 Beaconing (Unusual Ports) | T1071 | Sysmon EventCode 3 |
| Persistence via Run Registry Key | T1547 | Sysmon EventCode 13 |
| LSASS Memory Access | T1003 | Sysmon EventCode 10 |
| Timestomping Detection | T1070.006 | Sysmon EventCode 2 |
| Unsigned DLL Load | T1574 | Sysmon EventCode 7 |

---

## Repository Structure

```
Enterprise-SOC-Deployment/
│
├── README.md                        ← This file
├── LICENSE
├── CHANGELOG.md
├── SECURITY.md
│
├── docs/                            ← Full technical documentation
│   ├── Project-Overview.md
│   ├── Architecture.md
│   ├── Design-Decisions.md
│   ├── Deployment.md
│   ├── Hardening.md
│   ├── Docker.md
│   ├── Splunk.md
│   ├── Sysmon.md
│   ├── Universal-Forwarder.md
│   ├── Cisco-Meraki.md
│   ├── Log-Flow.md
│   ├── Threat-Detection.md
│   ├── Dashboard-Development.md
│   ├── SPL-Queries.md
│   ├── Troubleshooting.md
│   └── Lessons-Learned.md
│
├── architecture/                    ← Architecture diagrams
│   ├── SOC-Architecture-Overview.md
│   └── Log-Flow-Diagram.md
│
├── configurations/                  ← Sanitized config files
│   ├── indexes.conf
│   ├── props.conf
│   ├── transforms.conf
│   └── docker-compose.yml
│
├── hardening/                       ← OS hardening documentation
│   └── Rocky-Linux-Hardening-Checklist.md
│
├── dashboards/                      ← Dashboard build guides
│   ├── SOC-Overview.md
│   ├── ISO-27001-Controls.md
│   ├── Meraki-Network-Monitoring.md
│   └── Endpoint-Telemetry.md
│
├── spl/                             ← SPL query library
│   ├── Sysmon-EventCode-Searches.md
│   ├── Meraki-Searches.md
│   └── Windows-Security-Searches.md
│
├── detections/                      ← Detection rules
│   └── Detection-Rules.md
│
├── playbooks/                       ← Incident response
│   └── Incident-Response-Playbook.md
│
└── screenshots/                     ← Dashboard screenshots
    └── README.md
```

---

## Scope Note

> This repository documents technical methodology, architectural 
> decisions, and implementation approach. In accordance with 
> information security best practices and ISO 27001 confidentiality 
> principles, all organization-specific details including IP addresses, 
> hostnames, network topology specifics, and security posture 
> information have been sanitized or generalized. No sensitive 
> organizational information is exposed in this repository.

---

## Current Status

```
Infrastructure deployment          ████████████████████  Complete ✅
OS hardening                       ████████████████████  Complete ✅
Splunk deployment (Docker)         ████████████████████  Complete ✅
Log pipeline (Meraki + Sysmon)     ████████████████████  Complete ✅
Index configuration                ████████████████████  Complete ✅
Detection rules                    ████████████░░░░░░░░  In Progress ⚙️
Dashboard development              ████████████████░░░░  In Progress ⚙️
Endpoint scaling (16-18 hosts)     ████████░░░░░░░░░░░░  In Progress ⚙️
Email alerting                     ██████░░░░░░░░░░░░░░  In Progress ⚙️
ISO 27001 compliance dashboard     ████████████████████  Complete ✅
```

---

## Future Improvements

- Scale Sysmon deployment to all 16-18 production endpoints
- Integrate Microsoft 365 / Entra ID sign-in logs
- Add vulnerability scanner log ingestion (OpenVAS/Nessus)
- Implement automated incident response playbook execution
- Add Wazuh alongside Splunk for EDR capability comparison
- Pursue Splunk Enterprise base license for production scale
- Build SOAR-lite automation using Splunk's built-in alerting

---

## Author

**Ogheneovie Emezana (Kings)**
Security/IT Engineer | SOC Analyst | ISC2 CC | ISO 27001 Lead Auditor

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue)](https://www.linkedin.com/in/ogheneovie-emezana)
[![GitHub](https://img.shields.io/badge/GitHub-KingOvie--Cyber-black)](https://github.com/KingOvie-Cyber)

---

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for details.

---

> *"This environment was not built by following a tutorial. 
> It was built by solving real problems, diagnosing real failures, 
> and making deliberate architectural decisions for a live organization."*
