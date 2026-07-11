# Design Decisions
## Engineering Choices — Rationale and Trade-offs

---

## Introduction

This document explains the key design decisions made during the 
planning and deployment of the Enterprise SOC. For each decision, 
the reasoning is provided alongside the alternatives considered 
and the trade-offs accepted.

Understanding *why* decisions were made is as important as 
understanding *what* was built. This document is intended to 
demonstrate deliberate engineering judgment — not just technical 
execution.

---

## Decision 1 — VMware ESXi Over Cloud Infrastructure

### Decision
Deploy the SOC on bare-metal VMware ESXi rather than a cloud 
provider (AWS, Azure, GCP).

### Rationale
The organization already owned physical server hardware. Using 
existing infrastructure eliminated recurring cloud compute costs 
and kept all security monitoring data on-premises — an important 
consideration for an organization with data residency and 
confidentiality requirements.

### Alternatives Considered

| Option | Pros | Cons | Decision |
|---|---|---|---|
| AWS/Azure cloud | Managed infra, scalable | Ongoing cost, data leaves premises | Rejected |
| Bare-metal ESXi | On-prem, cost-effective, full control | Requires physical hardware management | Selected |
| VMware Workstation | Easy to set up | Not production-grade | Rejected |

### Trade-offs Accepted
Physical hardware requires maintenance, and there is no 
built-in high availability at the hypervisor level. Accepted 
given the organization's size, budget, and risk tolerance.

---

## Decision 2 — Rocky Linux Over Ubuntu or Windows Server

### Decision
Use Rocky Linux as the SOC server operating system.

### Rationale
Rocky Linux is RHEL-compatible, enterprise-grade, and has a long 
support lifecycle. It mirrors what most enterprise environments 
run in production, making the skills and configuration directly 
transferable.

### Alternatives Considered

| Option | Pros | Cons | Decision |
|---|---|---|---|
| Ubuntu Server | Large community, familiar | Not RHEL-compatible, less enterprise-standard | Rejected |
| Rocky Linux | RHEL-compatible, long lifecycle, enterprise-grade | Slightly smaller community than Ubuntu | Selected |
| Debian | Stable, lightweight | Not common in enterprise SOC environments | Rejected |
| Windows Server | Familiar to Windows admins | Licensing cost, larger attack surface | Rejected |

### Trade-offs Accepted
Rocky Linux has a smaller immediate community than Ubuntu, 
meaning some troubleshooting resources are less readily available. 
Accepted given the long-term operational and compatibility benefits.

---

## Decision 3 — Splunk in Docker Over Native Installation

### Decision
Deploy Splunk Enterprise inside a Docker container managed by 
Docker Compose rather than installing it directly on the OS.

### Rationale
Docker provides clean separation between the application and the 
operating system. Splunk can be upgraded, restarted, or recreated 
without affecting the host OS. Persistent named volumes ensure 
indexed data survives the container lifecycle.

### Alternatives Considered

| Option | Pros | Cons | Decision |
|---|---|---|---|
| Native RPM install | Simpler restart behavior, no container layer | OS-level coupling, harder to upgrade cleanly | Rejected |
| Docker (selected) | Clean lifecycle, persistent volumes, portable | Restart via Web UI can be problematic | Selected |
| Splunk Cloud | No infrastructure management | Cost, data leaves premises | Rejected |

### Trade-offs Accepted
The official Splunk Docker image uses an Ansible-based entrypoint 
that can cause issues when restarting Splunk via the Web UI's 
"Server Controls" menu. The workaround is to restart via 
`docker compose restart splunk` from the host — a minor 
operational consideration, not a fundamental limitation.

### Lesson Learned
Restarting Splunk should always be done from the host using 
`docker compose restart splunk` rather than from within 
Splunk Web.

---

## Decision 4 — RODC Over Full Domain Controller

### Decision
Deploy a Read-Only Domain Controller (RODC) on Windows Server 
2025 rather than a full writable Domain Controller.

### Rationale
This is a deliberate security architecture decision. In a SOC 
environment, the principle of least privilege extends to identity 
infrastructure. If the SOC server were ever compromised, an RODC 
limits the blast radius — an attacker cannot modify AD objects 
or extract writable credential hashes from a read-only replica.

### Alternatives Considered

| Option | Security Posture | Risk if Compromised | Decision |
|---|---|---|---|
| Full writable DC | Standard | High — full AD control | Rejected |
| RODC | Stronger | Low — read-only, limited credential exposure | Selected |
| No AD at all | N/A | N/A | Rejected |

### Trade-offs Accepted
An RODC cannot process certain write operations — password 
changes must be referred to a writable DC elsewhere in the domain. 
Acceptable given the security benefit.

---

## Decision 5 — Sysmon with Olaf Hartong Config Over Default Config

### Decision
Deploy Sysmon using the Olaf Hartong modular community 
configuration rather than a minimal or default configuration.

### Rationale
The default Sysmon configuration captures very little. The 
Olaf Hartong modular config provides MITRE ATT&CK-aligned 
coverage across process, network, file, registry, DNS, and DLL 
events while including noise reduction rules to avoid flooding 
the SIEM with low-value events.

### Alternatives Considered

| Option | Coverage | Volume | Detection Value | Decision |
|---|---|---|---|---|
| Default Sysmon config | Low | Low | Low | Rejected |
| Olaf Hartong modular | High | Medium | High | Selected |
| SwiftOnSecurity config | Medium | Low-Medium | Medium | Considered |
| Custom from scratch | Variable | Variable | Variable | Too time-intensive |

### Trade-offs Accepted
Higher log volume than a minimal configuration, which has 
implications for Splunk license sizing.

---

## Decision 6 — Meraki Direct-to-Splunk Syslog

### Decision
Configure Cisco Meraki to send syslog directly to Splunk's 
exposed UDP port 514 rather than through an intermediate 
syslog collector.

### Rationale
For the current environment scale (2-3 sites, one SIEM 
destination), an intermediate collector adds unnecessary 
complexity without meaningful benefit.

### Alternatives Considered

| Option | Complexity | Resilience | Decision |
|---|---|---|---|
| Direct to Splunk | Low | Medium | Selected |
| rsyslog to Splunk | Medium | Higher (buffering) | Considered |
| Multi-hop collection | High | Highest | Over-engineered for scale |

### Trade-offs Accepted
Without an intermediate collector, logs are lost during any 
Splunk downtime window. Acceptable at current scale.

---

## Decision 7 — Tiered Index Retention

### Decision
Apply different retention periods to different indexes based 
on data volume and compliance requirements.

### Rationale

| Index | Retention | Reasoning |
|---|---|---|
| sysmon | 90 days | High volume, loses investigative value after 90 days |
| meraki | 90 days | High volume, shorter investigation window needed |
| windows | 365 days | Lower volume, compliance-relevant, needed for annual audit cycles |

### Trade-offs Accepted
Minor additional configuration complexity versus a single 
global retention setting.

---

## Decision 8 — Dashboard Studio Over Classic Dashboards

### Decision
Build all dashboards using Splunk Dashboard Studio rather 
than the legacy Simple XML / Classic Dashboard framework.

### Rationale
Dashboard Studio is Splunk's current and future-forward 
dashboard framework. Building in Dashboard Studio demonstrates 
current Splunk skills rather than familiarity with a 
deprecated approach.

### Alternatives Considered

| Option | Modernity | Features | Decision |
|---|---|---|---|
| Classic/Simple XML | Legacy | Limited | Rejected |
| Dashboard Studio | Current | Rich | Selected |
| Splunk ES | Requires ES license | Very rich | Cost-prohibitive |

### Trade-offs Accepted
Dashboard Studio has a steeper initial learning curve than 
Classic, and some community troubleshooting resources still 
reference the Classic approach.

---

## Summary of Key Decisions

| Decision | Choice Made | Primary Reason |
|---|---|---|
| Hypervisor | VMware ESXi (bare metal) | On-prem, cost-effective, full control |
| OS | Rocky Linux | Enterprise-grade, RHEL-compatible |
| SIEM deployment | Docker + Docker Compose | Clean lifecycle, persistent volumes |
| AD architecture | RODC | Least privilege — limits compromise impact |
| Sysmon config | Olaf Hartong modular | MITRE ATT&CK coverage, noise reduction |
| Log collection | Direct syslog to Splunk | Appropriate for current scale |
| Retention policy | Tiered by source | Compliance + cost optimization |
| Dashboard framework | Dashboard Studio | Current Splunk standard |

---

> **Confidentiality Note:** All organization-specific details 
> have been sanitized. Design decisions are presented as 
> methodology — not as a map of organizational infrastructure.
