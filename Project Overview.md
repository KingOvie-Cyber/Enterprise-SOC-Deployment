# Project Overview
## Enterprise Security Operations Centre — Production Deployment

---

## Introduction

This document provides a comprehensive overview of the Enterprise 
Security Operations Centre (SOC) project. It describes the business 
context, objectives, scope, methodology, and outcomes of a production 
SOC built entirely from the ground up for a live organization.

This is not a theoretical exercise. The environment described here 
is currently operational, actively monitoring production assets, and 
generating real security telemetry.

---

## Business Context

### The Problem

The organization had no centralized visibility into its security 
posture. There was no mechanism to detect threats, monitor user 
activity, analyze network traffic, or respond to security incidents 
in a structured, repeatable way.

Specific gaps identified prior to this project:

- No SIEM or log aggregation capability
- No endpoint telemetry beyond basic Windows Event Logs
- No network traffic visibility or firewall log analysis
- No detection rules or alerting mechanisms
- No compliance evidence generation capability
- No documented incident response process

### The Objective

Design, deploy, and operate a production-grade Security Operations 
Centre capable of:

- Centralizing logs from all monitored assets
- Detecting threats in real time using custom detection logic
- Providing continuous visibility across endpoints and network
- Generating evidence for ISO 27001 Annex A compliance
- Establishing a repeatable, documented SOC operational process

### The Scope

```
In Scope:
├── Windows workstation endpoints (16-18 hosts)
├── Cisco Meraki network infrastructure (2-3 sites)
├── SOC server infrastructure (VMware ESXi)
└── Splunk SIEM platform

Out of Scope (Current Phase):
├── Cloud workload monitoring (planned future phase)
├── Microsoft 365 / Entra ID log integration (pending access)
├── Mobile device monitoring
└── OT/ICS environments
```

---

## Project Methodology

### Approach

The project followed a phased deployment methodology:

```
Phase 1 — Infrastructure
Design and deploy the physical and virtual infrastructure 
that underpins the SOC.

Phase 2 — Hardening
Secure the SOC server before connecting it to the production 
network or receiving any log data.

Phase 3 — SIEM Deployment
Deploy and configure Splunk Enterprise as the central 
log management and detection platform.

Phase 4 — Log Pipeline
Configure all log sources to forward data into Splunk 
and verify end-to-end data flow.

Phase 5 — Detection Engineering
Build detection rules, correlation searches, and alerting 
based on real data flowing through the pipeline.

Phase 6 — Dashboard Development
Build operational and compliance dashboards for SOC 
visibility and audit evidence.

Phase 7 — Operationalization
Document processes, playbooks, and procedures for 
ongoing SOC operation.
```

### Design Philosophy

Every architectural decision in this project was made against 
three criteria:

**Security first:** The SOC server was hardened before any 
production data was connected. Security configuration was 
not an afterthought — it was a prerequisite.

**Production grade:** This environment needed to work reliably, 
not just demonstrate a concept. Decisions around containerization, 
persistent storage, index retention, and network segmentation 
all reflect production operational requirements.

**Documented deliberately:** Architecture decisions, challenges, 
and lessons learned are documented throughout. A SOC that cannot 
be understood, maintained, or handed over is an operational risk, 
not an asset.

---

## What Was Built

### Infrastructure Layer

A dedicated SOC server was provisioned on VMware ESXi bare metal, 
running Rocky Linux as the host operating system. Rocky Linux was 
selected for its enterprise-grade stability, RHEL compatibility, 
and long support lifecycle — mirroring what most enterprise 
environments run in production.

### Security Layer

The Rocky Linux server was hardened prior to deployment using 
CIS Benchmark-aligned controls. SSH access was restricted, 
unnecessary services were disabled, firewall rules were tightened 
to only permit required ports, and kernel parameters were tuned 
to reduce the attack surface.

### SIEM Layer

Splunk Enterprise was deployed as a Docker container using Docker 
Compose, providing clean separation between the application and 
the operating system. Persistent Docker volumes ensure indexed 
data survives container restarts, upgrades, and recreation.

### Log Pipeline

Two primary log sources were integrated:

**Endpoint telemetry:** Sysmon was deployed on Windows endpoints 
with a tuned XML configuration based on the Olaf Hartong modular 
config. The Splunk Universal Forwarder ships Sysmon and Windows 
Event Log data to the SIEM over port 9997.

**Network telemetry:** Cisco Meraki was configured to forward 
syslog events to Splunk over UDP port 514, providing visibility 
into firewall events, VPN activity, URL requests, and network 
security events.

### Detection Layer

Custom detection rules were written in SPL (Splunk's Search 
Processing Language) and mapped to MITRE ATT&CK techniques. 
Detection coverage spans brute force attacks, persistence 
mechanisms, credential dumping indicators, lateral movement 
patterns, and network anomalies.

### Dashboard Layer

Four operational dashboards were built using Splunk Dashboard 
Studio (not legacy Classic/XML):

- SOC Overview — general visibility across all log sources
- Endpoint Telemetry — Sysmon event code monitoring
- Meraki Network Monitoring — network traffic and anomalies
- ISO 27001 Technological Controls Evidence — compliance

### Compliance Layer

An ISO 27001 Annex A control mapping was developed, linking 
Splunk data to specific Technological Controls. An audit-facing 
dashboard was built to generate exportable evidence for 
assessors reviewing the organization's information security posture.

---

## Current Operational State

```
Monitoring coverage:   16-18 Windows endpoints + Cisco Meraki
Daily log volume:      ~3-5 GB/day (estimated, across all sources)
Detection rules:       7 active, MITRE ATT&CK mapped
Dashboards:            4 operational (Dashboard Studio)
ISO controls covered:  8 Annex A Technological Controls
Retention policy:      90 days (Sysmon/Meraki) / 365 days (Windows)
```

---

## Project Outcomes

**Technical outcomes:**
- Full log pipeline operational from endpoint to indexed event
- Real-time detection alerts configured for high-priority threats
- Four production dashboards providing continuous SOC visibility
- Documented compliance evidence for ISO 27001 Technological Controls

**Operational outcomes:**
- Organization now has centralized visibility it previously lacked
- Security incidents can be detected, investigated, and documented
- Structured incident response process established and documented
- Compliance posture improved with auditable, exportable evidence

**Professional outcomes:**
- Demonstrated end-to-end SOC engineering capability
- Practical experience with production infrastructure deployment
- Real-world application of MITRE ATT&CK detection framework
- ISO 27001 audit evidence generation in a live environment

---

## Navigation

| Document | Description |
|---|---|
| [Architecture](Architecture.md) | Infrastructure and system design |
| [Design Decisions](Design-Decisions.md) | Why each technology was chosen |
| [Deployment](Deployment.md) | Step-by-step deployment process |
| [Hardening](Hardening.md) | Rocky Linux security hardening |
| [Docker](Docker.md) | Splunk containerization approach |
| [Splunk](Splunk.md) | SIEM configuration and index design |
| [Sysmon](Sysmon.md) | Endpoint telemetry configuration |
| [Universal Forwarder](Universal-Forwarder.md) | Log shipping setup |
| [Cisco Meraki](Cisco-Meraki.md) | Network log integration |
| [Log Flow](Log-Flow.md) | End-to-end data pipeline |
| [Threat Detection](Threat-Detection.md) | Detection rules and logic |
| [Dashboard Development](Dashboard-Development.md) | Dashboard build process |
| [SPL Queries](SPL-Queries.md) | Search query reference library |
| [Troubleshooting](Troubleshooting.md) | Issues encountered and resolved |
| [Lessons Learned](Lessons-Learned.md) | Reflections and improvements |

---

> **Confidentiality Note:** All organization-specific information 
> including IP addresses, hostnames, network topology, and security 
> posture details has been sanitized in accordance with ISO 27001 
> confidentiality principles. This document describes methodology 
> and approach, not organizational specifics. 
