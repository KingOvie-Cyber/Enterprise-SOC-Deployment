# Incident Response Playbook
## Enterprise SOC — Structured Response Procedures

---

## Introduction

This document defines the incident response procedures 
for the Enterprise SOC. It provides structured, 
repeatable workflows for detecting, containing, 
investigating, and recovering from security incidents.

Incident response without a playbook is reactive and 
inconsistent. With a playbook, even a single-analyst 
SOC can respond systematically under pressure.

---

## Incident Severity Classification

| Severity | Definition | Examples | Response Time |
|---|---|---|---|
| Critical | Active compromise confirmed or highly likely | LSASS access, ransomware indicators, confirmed data exfiltration | Immediate |
| High | Strong indicators of malicious activity | Office macro execution, brute force success, new admin account | Within 1 hour |
| Medium | Suspicious activity requiring investigation | Unusual ports, registry persistence, timestomping | Within 4 hours |
| Low | Anomalous but likely benign activity | Single failed login, unusual process, new software install | Within 24 hours |

---

## Incident Response Phases

```
PHASE 1: DETECTION
Alert fires or suspicious activity identified

PHASE 2: TRIAGE
Determine severity and scope quickly

PHASE 3: CONTAINMENT
Stop the bleeding — prevent further damage

PHASE 4: INVESTIGATION
Understand what happened and how

PHASE 5: ERADICATION
Remove the threat from the environment

PHASE 6: RECOVERY
Restore normal operations safely

PHASE 7: POST-INCIDENT
Document, report, and improve
```

---

## PLAYBOOK-001 — Brute Force / Credential Attack

**Triggered by:** RULE-001 (Brute Force Login Detection)
**Severity:** High

### Phase 1 — Detection
```
Alert condition: >5 failed logins from same source IP 
within 5 minutes
```

### Phase 2 — Triage

```spl
-- Get full picture of the attack
index=windows EventCode=4625
src_ip="<ATTACKER_IP>"
earliest=-1h latest=now
| table _time, src_ip, user, host, Failure_Reason
| sort _time
```

Questions to answer:
- Is the source IP internal or external?
- Which accounts are being targeted?
- Are any logins succeeding (EventCode 4624)?
- How many hosts are being targeted?

```spl
-- Check for successful logins from same source
index=windows EventCode=4624
src_ip="<ATTACKER_IP>"
earliest=-1h latest=now
| table _time, src_ip, user, host, Logon_Type
```

### Phase 3 — Containment

**If source is external:**
- Block the source IP at the Meraki firewall
- Document: IP address, time blocked, reason

**If source is internal:**
- Identify the device (check DHCP logs for IP-to-device mapping)
- Disconnect the device from the network if compromise suspected
- Contact the device owner

**If any login succeeded:**
- Immediately lock the compromised account
- Reset the account password
- Review all activity from that account in the last 24 hours

### Phase 4 — Investigation

```spl
-- Check all activity from successfully compromised account
index=windows user="<COMPROMISED_USER>"
earliest=-24h latest=now
| stats count by EventCode, host
| sort -count
```

```spl
-- Check for lateral movement from affected host
index=sysmon EventCode=3
host="<AFFECTED_HOST>"
earliest=-2h latest=now
| table _time, Image, DestinationIp, DestinationPort
```

### Phase 5 — Eradication
- Remove any unauthorized sessions
- Reset compromised credentials
- Review and remove any persistence mechanisms created

### Phase 6 — Recovery
- Restore account access with new credentials
- Confirm MFA is enabled (if available)
- Monitor the account for 48 hours post-incident

### Phase 7 — Post-Incident
- Document the incident in the incident log
- Review whether rate limiting or account lockout 
  policy changes are warranted
- Update detection thresholds if false positive rate was high

---

## PLAYBOOK-002 — Malware / Malicious Code Execution

**Triggered by:** RULE-002 (Office Macro Execution) or 
suspicious process activity
**Severity:** Critical

### Phase 1 — Detection
```
Alert condition: Office application spawning unusual 
child process
```

### Phase 2 — Triage

```spl
-- Get full process creation context
index=sysmon EventCode=1
host="<AFFECTED_HOST>"
earliest=-30m latest=now
| table _time, User, ParentImage, Image, CommandLine, Hashes
| sort _time
```

Questions to answer:
- What Office document triggered this?
- What child process was spawned?
- What did the command line contain?
- Did the process make network connections?
- Did the process create any files?

```spl
-- Check for network connections from affected host
index=sysmon EventCode=3
host="<AFFECTED_HOST>"
earliest=-30m latest=now
| table _time, Image, DestinationIp, DestinationPort
| sort _time
```

```spl
-- Check for files dropped
index=sysmon EventCode=11
host="<AFFECTED_HOST>"
earliest=-30m latest=now
| table _time, Image, TargetFilename
| sort _time
```

### Phase 3 — Containment

**Immediate actions (do these first):**
1. Isolate the affected endpoint from the network 
   (disconnect network cable or disable network adapter)
2. Do NOT power off the machine — volatile memory 
   may contain forensic evidence
3. Preserve the current state for investigation

**In Meraki:**
- Block the affected device's MAC address at the 
  network level as a secondary containment measure

### Phase 4 — Investigation

```spl
-- Full timeline of activity on affected host
index=sysmon OR index=windows
host="<AFFECTED_HOST>"
earliest=-2h latest=now
| table _time, EventCode, Image, CommandLine, 
  TargetFilename, TargetObject, DestinationIp
| sort _time
```

Investigation questions:
- What was the initial delivery mechanism 
  (email attachment, download, USB)?
- What persistence mechanisms were established?
- What data was accessed or exfiltrated?
- Did the malware spread to other hosts?

```spl
-- Check if other hosts connected to same C2 IP
index=sysmon EventCode=3
DestinationIp="<C2_IP>"
earliest=-24h latest=now
| stats count by host
```

### Phase 5 — Eradication
- Remove identified malicious files
- Remove registry persistence entries
- Run antivirus/antimalware scan on the endpoint
- Reimaging may be required for confirmed compromise

### Phase 6 — Recovery
- Only reconnect to network after confirmed clean
- Monitor the endpoint for 72 hours post-recovery
- Reset credentials for any accounts used on the 
  affected endpoint

### Phase 7 — Post-Incident
- Document the attack chain (initial access → execution 
  → persistence → C2 → impact)
- Submit malware samples to analysis if available
- Review phishing awareness training needs

---

## PLAYBOOK-003 — Unauthorized Account Access (Offboarding Gap)

**Triggered by:** RULE-008 or manual detection
**Severity:** Critical
**Context:** Based on a real incident in this environment 
where a departed employee accessed company systems 
after their departure

### Phase 1 — Detection
```
Alert condition: Login from account flagged as 
offboarded/departed, or login from unexpected device 
after departure
```

### Phase 2 — Triage

```spl
-- Get full login activity for the account
index=windows EventCode=4624
user="<DEPARTED_EMPLOYEE_ACCOUNT>"
earliest=-7d latest=now
| table _time, host, src_ip, Logon_Type, Workstation_Name
| sort _time
```

Questions to answer:
- When did the unauthorized access occur?
- From which device/IP?
- What resources were accessed?
- Was the departure date correctly recorded?
- Why was the account still active?

### Phase 3 — Containment
**Immediate:**
1. Disable the account immediately
2. Revoke all active sessions (force sign-out all devices)
3. Reset the account password (prevents re-access 
   if password was shared)
4. Revoke any API tokens or application passwords 
   associated with the account

### Phase 4 — Investigation

```spl
-- Review everything accessed during unauthorized session
index=windows user="<DEPARTED_EMPLOYEE_ACCOUNT>"
earliest="<DEPARTURE_DATE>" latest=now
| stats count by EventCode, host
| sort -count
```

- Review email access logs (if M365 integrated)
- Review file access logs for sensitive documents
- Determine if any data was exported or forwarded

### Phase 5 — Eradication
- Confirm account is fully disabled across all systems
- Revoke access to all cloud services (M365, cloud platforms)
- Review and remove any forwarding rules set on the 
  email account

### Phase 6 — Recovery
- Reassign any critical responsibilities the account held
- Ensure business continuity is maintained

### Phase 7 — Post-Incident
- Review and update the offboarding checklist
- Establish a maximum time window for account 
  deprovisioning after departure (e.g. same-day)
- Consider implementing automated account disablement 
  triggered by HR system on departure date
- Document the gap and remediation in the incident log

---

## PLAYBOOK-004 — Suspicious Network Activity / C2 Beaconing

**Triggered by:** RULE-003 (Unusual Port Outbound) or 
Meraki traffic anomaly
**Severity:** Medium to High

### Phase 2 — Triage

```spl
-- Full outbound connection picture from affected host
index=sysmon EventCode=3
host="<AFFECTED_HOST>"
earliest=-24h latest=now
| stats count by DestinationIp, DestinationPort, Image
| sort -count
```

```spl
-- Check beaconing pattern (regular interval connections)
index=sysmon EventCode=3
host="<AFFECTED_HOST>"
DestinationIp="<SUSPICIOUS_IP>"
earliest=-24h latest=now
| timechart span=5m count
```

A regular, repeating connection pattern at fixed 
intervals (e.g. every 5 minutes) is a strong 
beaconing indicator.

### Phase 3 — Containment
- Block the destination IP at the Meraki firewall
- If beaconing confirmed — isolate the endpoint

---

## Incident Log Template

Every incident must be logged with the following 
information:

```
Incident ID:        INC-[YEAR]-[NUMBER] (e.g. INC-2026-001)
Date/Time Detected: 
Date/Time Resolved: 
Severity:           Critical / High / Medium / Low
Detection Method:   Alert name or manual discovery
Affected Systems:   
Incident Summary:   
Root Cause:         
Actions Taken:      
Lessons Learned:    
Recommendations:    
Analyst:            
```

---

## Splunk Investigation Quick Reference

### Build a Timeline of Any Host

```spl
index=sysmon OR index=windows
host="<HOSTNAME>"
earliest=-2h latest=now
| table _time, EventCode, Image, CommandLine, 
  TargetFilename, TargetObject, DestinationIp
| sort _time
```

### Find All Activity From a Specific User

```spl
index=sysmon OR index=windows
user="<USERNAME>"
earliest=-24h latest=now
| stats count by EventCode, host, Image
| sort -count
```

### Find All Connections to a Specific IP

```spl
index=sysmon EventCode=3
DestinationIp="<IP_ADDRESS>"
earliest=-24h latest=now
| stats count by host, Image, DestinationPort
| sort -count
```

### Find All Files Created on a Host in a Window

```spl
index=sysmon EventCode=11
host="<HOSTNAME>"
earliest="<START_TIME>" latest="<END_TIME>"
| table _time, Image, TargetFilename
| sort _time
```

---

## Communication Templates

### Internal Escalation (to IT Manager)

> "We have detected [SEVERITY] security incident 
> affecting [SYSTEM/USER]. Initial indicators suggest 
> [BRIEF DESCRIPTION]. We have taken the following 
> immediate containment actions: [ACTIONS]. 
> Investigation is ongoing. Next update in [TIMEFRAME]."

### Stakeholder Update

> "A security incident was detected on [DATE] at [TIME]. 
> Affected systems: [LIST]. Current status: [CONTAINED / 
> UNDER INVESTIGATION / RESOLVED]. Impact assessment: 
> [DESCRIPTION]. No further action required from your 
> team at this time / Please [ACTION REQUIRED]."

---

> **Confidentiality Note:** Specific incident details, 
> affected system names, usernames, and organizational 
> context referenced in this playbook have been 
> generalized. Placeholder values (shown in angle 
> brackets) should be replaced with actual values 
> during a real incident response.
