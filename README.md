# Enterprise SOC Deployment
### Production Security Operations Centre — Architecture, Implementation & Operations

> **Note:** This repository documents a real production SOC deployment 
> for a live organization. All sensitive company data, IP addresses, 
> hostnames, and credentials have been sanitized. Architecture decisions, 
> technology choices, and implementation methodology are documented for 
> professional portfolio purposes.

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Business Problem](#business-problem)
3. [SOC Architecture](#soc-architecture)
4. [Infrastructure Stack](#infrastructure-stack)
5. [Log Sources](#log-sources)
6. [Implementation Phases](#implementation-phases)
7. [Security Hardening](#security-hardening)
8. [Detection Engineering](#detection-engineering)
9. [Challenges & Solutions](#challenges--solutions)
10. [Current Status](#current-status)
11. [Skills Demonstrated](#skills-demonstrated)

---

## Project Overview

Designed, built, and deployed a fully operational Security Operations 
Centre (SOC) for a live organization from the ground up — with zero 
existing security monitoring infrastructure at the start of the project.

This was not a lab exercise. This is production infrastructure actively 
monitoring real company assets.

**Scope:** End-to-end SOC deployment including physical infrastructure, 
virtualization, OS hardening, log pipeline architecture, SIEM deployment, 
network security monitoring, endpoint telemetry, and identity management.

---

## Business Problem

The organization had no centralized visibility into:
- Network traffic and firewall events
- Endpoint activity and process execution
- Authentication and identity events
- Security incidents and anomalous behavior

**Objective:** Build a SOC capable of centralizing logs from all company 
assets, detecting threats in real time, and providing a foundation for 
incident response.

---

## SOC Architecture

COMPANY NETWORK
                      |
          +-----------+-----------+
          |                       |
    [Cisco Meraki]          [Endpoints]
    Network Firewall         Sysmon
    & Switch Logs            Telemetry
          |                       |
          +----------+------------+
                     |
              [SYSLOG COLLECTOR]
              Rocky Linux VM
              (Hardened)
              Rsyslog / Syslog-ng
                     |
                     v
               [SPLUNK SIEM]
               Rocky Linux VM
               Docker Container
               Splunk Enterprise
                     |
               +-----+-----+
               |           |
         [Dashboards]  [Alerts]
         [Detection]   [Reports]
                     |
              [IDENTITY LAYER]
              Windows Server 2025
              Active Directory
              RODC Deployment


---

## Infrastructure Stack

### Physical Hardware
| Component | Specification |
|---|---|
| SOC Server 1 | Lenovo ThinkStation |
| SOC Server 2 | Lenovo ThinkStation |
| Hypervisor | VMware ESXi (bare metal) |
| Network | Cisco Meraki (managed) |

### Virtual Machines
| VM | OS | Role |
|---|---|---|
| syslog-collector | Rocky Linux (hardened) | Centralized log collection |
| splunk-host | Rocky Linux | SIEM host (Docker) |
| dc-01 | Windows Server 2025 | RODC / Active Directory |

### Software Stack
| Technology | Purpose | Version |
|---|---|---|
| VMware ESXi | Hypervisor / virtualization | Current |
| Rocky Linux | SOC server OS | 9.x |
| Splunk Enterprise | SIEM | Via Docker |
| Docker | Container runtime | Current |
| Sysmon | Endpoint telemetry | Latest |
| Rsyslog | Log aggregation | Current |
| Active Directory | Identity management | Server 2025 |

---

## Log Sources

### Currently Ingesting
| Source | Log Type | Method |
|---|---|---|
| Cisco Meraki | Firewall, traffic, security events | Syslog → collector |
| Windows Endpoint | Process creation, network connections, registry, file events | Sysmon → Splunk forwarder |

### Planned (In Progress)
| Source | Log Type |
|---|---|
| All company workstations | Sysmon telemetry |
| Servers | Windows Event Logs |
| Azure AD / Entra ID | Identity and authentication logs |
| Cloud assets | Azure Monitor / activity logs |

---

## Implementation Phases

### Phase 1 — Infrastructure ✅ Complete
- Procured and configured Lenovo ThinkStation servers
- Installed VMware ESXi on bare metal
- Provisioned virtual machines for each SOC function
- Configured internal networking and VLAN segmentation

### Phase 2 — OS Hardening ✅ Complete
- Deployed Rocky Linux on syslog collector and Splunk host
- Applied CIS Benchmark hardening (see /Hardening section)
- Disabled unnecessary services and ports
- Configured firewall rules (firewalld)
- Implemented SSH key authentication, disabled root login
- Configured auditd for system-level logging

### Phase 3 — Log Pipeline ✅ Complete
- Configured Cisco Meraki to forward syslog to collector
- Deployed and tuned Rsyslog on Rocky Linux collector
- Configured log routing and storage on collector
- Validated log integrity and timestamp accuracy

### Phase 4 — SIEM Deployment ✅ Complete
- Installed Docker on Rocky Linux Splunk host
- Deployed Splunk Enterprise as Docker container
- Configured persistent storage volumes for index data
- Configured Splunk to receive forwarded logs
- Created indexes for network and endpoint data
- Built initial sourcetype parsing for Meraki and Sysmon

### Phase 5 — Endpoint Telemetry ✅ (Pilot)
- Deployed Sysmon on pilot endpoint
- Configured Sysmon using community XML config
  (SwiftOnSecurity / Olaf Hartong configuration)
- Validated event generation in Splunk
- Events confirmed: process creation, network connections,
  file creation, registry modifications

### Phase 6 — Identity Layer ✅ Complete
- Deployed Windows Server 2025 as VM
- Configured Active Directory Domain Services
- Deployed Read-Only Domain Controller (RODC)
- RODC chosen deliberately — minimizes credential 
  exposure risk in SOC environment (least privilege design)

### Phase 7 — Scale & Detection ⚙️ In Progress
- Scaling Sysmon deployment to all company endpoints
- Building detection correlation searches in Splunk
- Developing dashboards for SOC visibility
- Writing incident response playbooks

---

## Security Hardening

### Rocky Linux Hardening (Applied to Both VMs)

**Access Control**
- Root login disabled via SSH
- SSH key-based authentication only (password auth disabled)
- Sudo access restricted to named accounts only
- Umask set to 027

**Network Hardening**
- Firewalld enabled and configured
- Only required ports open (syslog: 514, Splunk: 8000/9997)
- IPv6 disabled (not required in environment)
- TCP wrappers configured

**Services**
- All unnecessary services disabled and masked
- Postfix, Bluetooth, CUPS disabled
- Auditd enabled for system call logging

**File System**
- Separate partitions for /tmp, /var, /var/log
- noexec, nosuid flags on /tmp
- Sticky bit verified on world-writable directories

**Kernel Parameters (sysctl)**
- IP forwarding disabled
- ICMP redirects disabled
- Syn cookies enabled
- Reverse path filtering enabled

---

## Detection Engineering

### Splunk Detection Searches (Examples)

**Brute Force Login Detection**
```spl
index=windows EventCode=4625
| bucket _time span=5m
| stats count by _time, src_ip, user
| where count > 5
| table _time, src_ip, user, count
```

**New Process Spawned by Office Application**
```spl
index=sysmon EventCode=1
ParentImage IN ("*\\winword.exe","*\\excel.exe","*\\powerpnt.exe")
| table _time, host, ParentImage, Image, CommandLine, User
```

**Outbound Connection to Unusual Port**
```spl
index=sysmon EventCode=3
| where NOT (dest_port IN (80, 443, 53, 22, 25))
| stats count by src_ip, dest_ip, dest_port
| sort - count
```

**Cisco Meraki — Outbound Traffic Spike**
```spl
index=meraki
| timechart span=1h sum(bytes_sent) by src_ip
| where bytes_sent > 500000000
```

---

## Challenges & Solutions

| Challenge | Solution |
|---|---|
| Splunk persistent data lost on container restart | Configured Docker volume mounts for /opt/splunk/var |
| Meraki logs not parsing correctly in Splunk | Built custom props.conf and transforms.conf for Meraki sourcetype |
| Rocky Linux firewalld blocking syslog UDP 514 | Added permanent firewalld rule for syslog zone |
| ESXi networking — VMs not reaching collector | Configured vSwitch port group and verified VLAN tagging |
| Sysmon generating excessive volume | Tuned XML config to filter noise while keeping high-fidelity events |

---

## Current Status              
Infrastructure        ████████████████████  100% ✅
OS Hardening          ████████████████████  100% ✅
Log Pipeline          ████████████████████  100% ✅
SIEM Deployment       ████████████████████  100% ✅
Endpoint Telemetry    ████████░░░░░░░░░░░░   40% ⚙️  (scaling in progress)
Detection Rules       ██████░░░░░░░░░░░░░░   30% ⚙️  (building)
Dashboards            ████░░░░░░░░░░░░░░░░   20% ⚙️  (building)
IR Playbooks          ██░░░░░░░░░░░░░░░░░░   10% ⬜ (planned)
Full Asset Coverage   ████░░░░░░░░░░░░░░░░   20% ⚙️  (scaling)


---

## Skills Demonstrated

**Infrastructure & Virtualization**
- VMware ESXi bare metal deployment and configuration
- Virtual machine provisioning and resource allocation
- vSwitch and network configuration

**Linux Administration**
- Rocky Linux deployment and administration
- CIS Benchmark hardening implementation
- Rsyslog configuration and log pipeline management
- Docker deployment and container management
- Systemd service management

**SIEM Engineering**
- Splunk Enterprise deployment via Docker
- Index configuration and data model design
- Sourcetype parsing (props.conf, transforms.conf)
- SPL search development
- Detection rule creation

**Network Security**
- Cisco Meraki configuration and log forwarding
- Network traffic analysis
- Firewall log interpretation

**Endpoint Security**
- Sysmon deployment and XML configuration tuning
- Windows event log analysis
- Endpoint telemetry pipeline design

**Identity & Access Management**
- Active Directory deployment (Windows Server 2025)
- RODC architecture and security design
- Least privilege implementation

**Cloud & Hybrid**
- Azure AD / Entra ID
- Azure Virtual Machines, Blob Storage, NSG configuration
- Hybrid identity concepts

---

*Repository maintained by Ogheneovie Emezana — 
Security Operations Engineer*  
*[LinkedIn](https://www.linkedin.com/in/ogheneovie-emezana)*
