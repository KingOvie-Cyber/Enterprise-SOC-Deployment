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


Infrastructure        ████████████████████  100% ✅

OS Hardening          ████████████████████  100% ✅

Log Pipeline          ████████████████████  100% ✅

SIEM Deployment       ████████████████████  100% ✅

Endpoint Telemetry    ████████░░░░░░░░░░░░   40% ⚙️  (scaling in progress)

Detection Rules       ██████░░░░░░░░░░░░░░   30% ⚙️  (building)

Dashboards            ████░░░░░░░░░░░░░░░░   20% ⚙️  (building)

IR Playbooks          ██░░░░░░░░░░░░░░░░░░   10% ⬜ (planned)

Full Asset Coverage   ████░░░░░░░░░░░░░░░░   20% ⚙️  (scaling)



*Repository maintained by Ogheneovie Emezana — 
Security Operations Engineer*  
*[LinkedIn](https://www.linkedin.com/in/ogheneovie-emezana)*
