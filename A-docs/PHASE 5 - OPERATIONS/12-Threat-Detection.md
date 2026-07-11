# Threat Detection
## Detection Engineering — Rules, Logic, and MITRE ATT&CK Mapping

---

## Introduction

This document describes the threat detection capability 
built into the Enterprise SOC. It covers the detection 
engineering methodology, the detection rules implemented, 
their mapping to the MITRE ATT&CK framework, and the 
rationale behind each detection decision.

Detection engineering is the practice of building 
systematic, repeatable mechanisms to identify adversary 
behavior within an environment. It is the difference 
between a SIEM that stores logs and a SOC that 
actively defends the organization.

---

## Detection Engineering Methodology

### The Detection Development Process

Each detection rule in this SOC was developed through 
the following process:

```
1. IDENTIFY THREAT
   What adversary technique are we trying to detect?
   Reference: MITRE ATT&CK framework

2. IDENTIFY DATA SOURCE
   What log source captures evidence of this technique?
   (Sysmon EventCode, Windows Event Log, Meraki)

3. IDENTIFY INDICATOR
   What specific field values, patterns, or behaviors
   indicate this technique is occurring?

4. WRITE AND TEST SPL
   Build the search in Splunk Search & Reporting
   Verify it returns expected results against real data
   Tune to reduce false positives

5. DOCUMENT AND DEPLOY
   Document the detection logic, data source, 
   MITRE mapping, and false positive considerations
   Deploy as a saved search or alert
```

### Detection Philosophy

**Signal over noise:** A detection that generates 
constant false positives will be ignored. Every 
detection rule was tested against real production 
data and tuned to minimize noise while preserving 
coverage of genuine threats.

**Context-aware detection:** A single event rarely 
tells the whole story. Detection rules are designed 
to capture behavioral patterns — not just individual 
events — where possible.

**Documented rationale:** Every detection rule has 
documented reasoning. "We alert on this because..." 
matters as much as the SPL itself.

---

## Detection Rules

---

### Detection 1 — Brute Force Login Detection

**MITRE ATT&CK:** T1110 — Brute Force

**Description:**
Detects multiple failed authentication attempts 
against Windows accounts within a short time window. 
This pattern is consistent with password spraying, 
credential stuffing, or targeted brute force attacks.

**Data Source:** Windows Security Event Log
**EventCode:** 4625 (An account failed to log on)

**Detection Logic:**
```spl
index=windows EventCode=4625
| bucket _time span=5m
| stats count by _time, src_ip, user
| where count > 5
| table _time, src_ip, user, count
| sort -count
```

**Threshold:** More than 5 failed logins from the 
same source IP within a 5-minute window.

**False Positive Considerations:**
- A user who has forgotten their password may 
  trigger this legitimately
- Service accounts with incorrect stored credentials 
  may generate continuous failures
- Review the `user` field — service accounts failing 
  repeatedly are lower priority than interactive 
  user accounts being targeted

**Severity:** High

---

### Detection 2 — Malicious Office Macro Execution

**MITRE ATT&CK:** T1566.001 — Phishing: Spearphishing Attachment

**Description:**
Detects Office applications (Word, Excel, PowerPoint, 
Outlook) spawning unusual child processes. This is 
the classic indicator of a malicious macro executing 
a payload — one of the most common initial access 
techniques used by threat actors.

**Data Source:** Sysmon
**EventCode:** 1 (Process Creation)

**Detection Logic:**
```spl
index=sysmon EventCode=1
ParentImage IN (
  "*\\winword.exe",
  "*\\excel.exe",
  "*\\powerpnt.exe",
  "*\\outlook.exe"
)
NOT Image IN (
  "*\\splwow64.exe",
  "*\\csc.exe",
  "*\\werfault.exe"
)
| table _time, host, User, ParentImage, Image, CommandLine
| sort -_time
```

**What to look for:**
- Office application spawning `cmd.exe`, `powershell.exe`, 
  `wscript.exe`, `mshta.exe`, or any executable in 
  temp/AppData directories
- Unusual command line arguments in the child process

**False Positive Considerations:**
- Some legitimate Office add-ins spawn helper processes
- Review the full command line of the spawned process 
  before escalating

**Severity:** Critical

---

### Detection 3 — Outbound Connection on Unusual Port

**MITRE ATT&CK:** T1071 — Application Layer Protocol (C2)

**Description:**
Detects outbound network connections to destination 
ports outside the standard business-use set. Malware 
communicating with command-and-control infrastructure 
often uses non-standard ports to avoid detection.

**Data Source:** Sysmon
**EventCode:** 3 (Network Connection)

**Detection Logic:**
```spl
index=sysmon EventCode=3 Initiated=true
| where NOT (DestinationPort IN (
    80, 443, 53, 22, 25, 587, 143, 993, 
    8080, 8443, 3389, 445, 135
  ))
| stats count by src_ip, DestinationIp, DestinationPort, Image
| where count > 3
| sort -count
```

**What to look for:**
- Connections on unusual high ports (>8000) from 
  non-server processes
- Repeated connections to the same external IP on 
  the same unusual port (beaconing pattern)
- Connections from unusual initiating processes 
  (browsers, documents, scripts)

**False Positive Considerations:**
- Many legitimate applications use non-standard ports
- Always review the initiating process (`Image`) 
  and destination IP before escalating
- Known-good applications using specific ports 
  can be added to the exclusion list

**Severity:** Medium

---

### Detection 4 — Persistence via Run Registry Key

**MITRE ATT&CK:** T1547.001 — Boot/Logon Autostart: Registry Run Keys

**Description:**
Detects modifications to Windows Run and RunOnce 
registry keys — one of the most common persistence 
mechanisms used by malware. Any executable added 
to these keys will launch automatically at logon.

**Data Source:** Sysmon
**EventCode:** 13 (Registry Value Set)

**Detection Logic:**
```spl
index=sysmon EventCode=13
TargetObject IN (
  "*\\Software\\Microsoft\\Windows\\CurrentVersion\\Run\\*",
  "*\\Software\\Microsoft\\Windows\\CurrentVersion\\RunOnce\\*",
  "*\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon\\*"
)
| table _time, host, User, Image, TargetObject, Details
| sort -_time
```

**What to look for:**
- Entries pointing to executables in temp, AppData, 
  or other non-standard locations
- Entries created by unusual processes (not installers 
  or legitimate software update mechanisms)
- RunOnce entries that persist beyond the expected 
  single-run lifecycle

**False Positive Considerations:**
- Legitimate software installers frequently write 
  to Run keys
- Browser auto-launch entries (e.g., Edge) are benign
- Review the `Details` field (the value being written) 
  and `Image` (the process writing it)

**Severity:** High

---

### Detection 5 — LSASS Memory Access (Credential Dumping)

**MITRE ATT&CK:** T1003.001 — OS Credential Dumping: LSASS Memory

**Description:**
Detects processes accessing LSASS (Local Security 
Authority Subsystem Service) memory. LSASS holds 
credential material in memory — accessing it is 
a primary technique used by tools like Mimikatz 
to extract plaintext passwords and hashes.

**Data Source:** Sysmon
**EventCode:** 10 (Process Access)

**Detection Logic:**
```spl
index=sysmon EventCode=10
TargetImage="*\\lsass.exe"
NOT SourceImage IN (
  "*\\MsMpEng.exe",
  "*\\svchost.exe",
  "*\\wininit.exe",
  "*\\csrss.exe",
  "*\\lsass.exe"
)
| table _time, host, SourceImage, TargetImage, GrantedAccess
| sort -_time
```

**What to look for:**
- Access from non-system processes to lsass.exe
- Specific `GrantedAccess` values associated with 
  credential extraction (e.g., 0x1010, 0x1410)
- Access from PowerShell, cmd.exe, or unusual executables

**False Positive Considerations:**
- Security software (AV, EDR) legitimately accesses 
  LSASS for monitoring purposes
- Always verify the `SourceImage` before escalating
- System processes in the exclusion list cover 
  the most common legitimate access patterns

**Severity:** Critical

---

### Detection 6 — Timestomping Detection

**MITRE ATT&CK:** T1070.006 — Indicator Removal: Timestomp

**Description:**
Detects changes to file creation timestamps. 
Attackers modify file timestamps (timestomping) 
to make malicious files appear older than they 
are, hindering forensic timeline analysis.

**Data Source:** Sysmon
**EventCode:** 2 (File Creation Time Changed)

**Detection Logic:**
```spl
index=sysmon EventCode=2
| eval time_diff=abs(strptime(CreationUtcTime, "%Y-%m-%d %H:%M:%S") - strptime(PreviousCreationUtcTime, "%Y-%m-%d %H:%M:%S"))
| where time_diff > 86400
| table _time, host, Image, TargetFilename, PreviousCreationUtcTime, CreationUtcTime, time_diff
| sort -time_diff
```

**What to look for:**
- Files whose timestamps were changed by more 
  than 24 hours (significant manipulation)
- Changes made by unusual processes
- Files in user-writable locations (AppData, Temp)

**False Positive Considerations:**
- Software installers occasionally normalize 
  timestamps during installation
- File synchronization tools may adjust timestamps

**Severity:** Medium

---

### Detection 7 — New Local Administrator Account Created

**MITRE ATT&CK:** T1136.001 — Create Account: Local Account

**Description:**
Detects the creation of new local user accounts 
and their addition to the local Administrators 
group. Attackers create privileged local accounts 
for persistence and lateral movement.

**Data Source:** Windows Security Event Log
**EventCodes:** 4720 (Account Created) + 4732 
(Member Added to Security-Enabled Local Group)

**Detection Logic:**
```spl
index=windows (EventCode=4720 OR EventCode=4732)
| stats values(EventCode) as events, 
        values(host) as hosts by user
| where mvcount(events) >= 2
| table _time, hosts, user, events
```

**What to look for:**
- New accounts created outside of normal business 
  hours or by unexpected processes
- Accounts immediately added to Administrators group 
  after creation
- Account names that do not follow organizational 
  naming conventions

**False Positive Considerations:**
- IT administrators creating legitimate service 
  accounts will trigger this
- Correlate with change management records to 
  distinguish authorized from unauthorized creation

**Severity:** High

---

## Detection Coverage Summary

| Detection Rule | MITRE Technique | Data Source | EventCode(s) | Severity |
|---|---|---|---|---|
| Brute Force Login | T1110 | Windows Security | 4625 | High |
| Malicious Office Macro | T1566.001 | Sysmon | 1 | Critical |
| Outbound C2 Beaconing | T1071 | Sysmon | 3 | Medium |
| Run Key Persistence | T1547.001 | Sysmon | 13 | High |
| LSASS Memory Access | T1003.001 | Sysmon | 10 | Critical |
| Timestomping | T1070.006 | Sysmon | 2 | Medium |
| Local Admin Created | T1136.001 | Windows Security | 4720, 4732 | High |

---

## Detection Gaps

The following techniques are not currently covered 
by active detection rules. These represent known 
gaps for future development:

| Technique | Gap Reason | Priority |
|---|---|---|
| T1021 — Remote Services | Requires network flow data | Medium |
| T1078 — Valid Accounts | Requires baseline normal behavior | High |
| T1053 — Scheduled Task | Sysmon coverage planned | Medium |
| T1082 — System Discovery | High false positive risk, needs tuning | Low |
| T1016 — Network Discovery | Requires network flow data | Medium |

---

## Planned Detection Improvements

- Deploy detection rules as Splunk saved searches 
  with email alerting for Critical and High severity
- Develop baseline behavioral profiles per user 
  and host to enable anomaly-based detection
- Add detection coverage for scheduled task creation 
  (Sysmon EventCode 1 parent-child analysis)
- Integrate threat intelligence feeds for known-bad 
  IP and domain matching against Meraki URL logs
- Build correlation searches that combine multiple 
  indicators into a single high-confidence alert

---

## Real-World Finding

During active monitoring of this production 
environment, the Failed Login detection identified 
a spike of 64 failed login attempts within a single 
24-hour period across the monitored endpoints.

Investigation revealed this was concentrated 
activity requiring review — demonstrating that 
the detection logic was functioning correctly 
and providing actionable visibility the 
organization did not have before the SOC was built.

Additionally, the off-boarding of a staff member 
revealed a gap in account deprovisioning — the 
individual accessed their company email from 
an unauthorized device after departure. This 
incident reinforced the need for automated 
off-boarding alerting, which has been added 
to the detection development roadmap.

---

> **Confidentiality Note:** Specific investigation 
> outcomes, affected usernames, device identifiers, 
> and organizational security findings have been 
> generalized. This document describes detection 
> methodology only.
