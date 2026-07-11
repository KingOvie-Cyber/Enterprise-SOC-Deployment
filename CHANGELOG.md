# Changelog
## Enterprise SOC Deployment — Version History

All notable changes to this project are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)

---

## [Unreleased] — In Progress

### Planned
- Scale Sysmon deployment to all 16-18 production endpoints
- Microsoft 365 / Entra ID sign-in log integration
- Automated offboarding alert (RULE-008)
- Email alerting for Critical and High severity detections
- Splunk Enterprise base license procurement
- Vulnerability scanner integration (OpenVAS/Nessus)
- Firewall per-rule logging enabled in Meraki dashboard
- DHCP lease identity mapping lookup table

---

## [1.3.0] — 2026-06

### Added
- ISO 27001 Technological Controls Evidence dashboard
  (8 Annex A controls mapped to Splunk monitoring evidence)
- Meraki Network Monitoring dashboard
  (7 panels: top domains, top talkers, event types, DoH detection)
- DNS-over-HTTPS visibility gap identified and documented
  (39 devices using DoH across 2-3 sites)
- Detection Rule RULE-008 added to roadmap
  (Offboarding gap identified via real incident)
- Complete GitHub portfolio documentation (this repository)

### Changed
- Sysmon configuration updated to Olaf Hartong modular config
  (restored missing NetworkConnect / EventCode 3 coverage)
- Splunk index retention policy documented and finalized
- Docs folder reorganized to follow deployment sequence (01-16)

### Fixed
- Sysmon NetworkConnect rule missing from initial config
- Universal Forwarder Deployment Client retry loop
  (deploymentclient.conf disabled)
- Dashboard Studio token wiring errors (ISO 27001 dashboard)
- Meraki field extraction (type vs signature field name)

---

## [1.2.0] — 2026-05

### Added
- SOC Overview Dashboard (Dashboard Studio)
  (Total events, active hosts, failed logins, event distribution,
  events over time, recent process creations)
- Endpoint Telemetry Dashboard (Sysmon EventCode panels)
- Detection rules RULE-001 through RULE-007 (active)
- Email alerting configuration (Outlook/M365 SMTP)
- Splunk index architecture (meraki, sysmon, windows)
- props.conf and transforms.conf for Meraki routing
- Sysmon ACL permission fix documented (wevtutil command)

### Changed
- Docker Compose updated with named persistent volumes
  (resolved data loss on container recreation)

### Fixed
- Sysmon errorCode=5 (Access Denied on event log channel)
- inputs.conf typo (disabled=fasle corrected to disabled=false)
- Docker container crash on Web UI restart
  (workaround: always restart via docker compose restart)
- Universal Forwarder fishbucket position loss after
  Sysmon config reload

---

## [1.1.0] — 2026-04

### Added
- Cisco Meraki syslog integration (UDP 514)
- Splunk Universal Forwarder deployed on pilot endpoint
- Sysmon deployed on pilot Windows endpoint
- Windows Event Log forwarding (Security/System/Application)
- Splunk receiving port 9997 configured
- End-to-end pipeline verification (all three indexes active)
- Windows Server 2025 RODC deployment

### Changed
- Splunk deployment moved to Docker Compose
  (from initial native install attempt)

### Fixed
- Docker port mapping error (8088 vs 8089)
- Deployment Client handshake loop
  (deploymentclient.conf created with disabled=true)

---

## [1.0.0] — 2026-03

### Added
- VMware ESXi deployed on bare metal (Lenovo ThinkStation)
- Rocky Linux 9.x VM provisioned (soc-core)
- Rocky Linux hardened (CIS-aligned controls)
  - SSH hardened (root login disabled, MaxAuthTries 3)
  - Firewalld configured (only required ports open)
  - Kernel hardening (sysctl 99-hardening.conf)
  - Unnecessary services disabled
  - Auditd enabled
- Docker Engine installed
- Docker Compose installed
- Splunk Enterprise deployed via Docker Compose
- Splunk Web UI accessible on port 8000
- Initial indexes created (meraki, sysmon, windows)

### Architecture Decisions Made
- RODC over full Domain Controller (least privilege)
- Docker over native install (clean lifecycle management)
- Rocky Linux over Ubuntu (RHEL-compatible, enterprise-grade)
- Direct Meraki-to-Splunk syslog (appropriate for current scale)
- Dashboard Studio over Classic dashboards (current standard)
- Tiered index retention (90 days sysmon/meraki, 365 days windows)

---

## Real Incidents Detected

| Date | Incident | Detection Method | Outcome |
|---|---|---|---|
| 2026-05 | 64 failed login attempts in 24h | Failed Login dashboard panel | Investigated, root cause determined |
| 2026-06 | Departed employee post-departure account access | Manual detection / dashboard review | Account disabled, offboarding process reviewed, RULE-008 planned |

---

> **Confidentiality Note:** Dates shown are approximate 
> and generalized. Specific incident details, system 
> names, and organizational identifiers have been 
> sanitized in accordance with ISO 27001 
> confidentiality principles.
