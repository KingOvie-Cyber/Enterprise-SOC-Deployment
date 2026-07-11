# Detection Rules
## Production Detection Engineering — SPL Rules and MITRE ATT&CK Mapping

---

## Introduction

This document contains the complete detection rule library 
for the Enterprise SOC. Each rule is documented with its 
threat context, data source, detection logic, MITRE ATT&CK 
mapping, tuning notes, and deployment status.

All rules have been tested against real production data 
in this environment before deployment.

---

## Detection Rule Standard

Every detection rule follows this documentation standard:

```
Rule Name:          Human-readable name
MITRE Technique:    ATT&CK technique ID and name
Severity:           Critical / High / Medium / Low
Data Source:        Index and EventCode(s)
Status:             Active / Testing / Planned
Description:        What this rule detects and why it matters
SPL:                The actual search query
Threshold:          What triggers an alert
False Positives:    Known benign triggers and how to handle them
Response:           Initial triage steps when this fires
```

---

## RULE-001 — Brute Force Login Detection

```
Rule Name:       Brute Force Login Detection
MITRE Technique: T1110 — Brute Force
Severity:        High
Data Source:     index=windows, EventCode=4625
Status:          Active
```

### Description
Detects multiple consecutive failed authentication attempts 
against Windows accounts from the same source within a 
short time window. Consistent with password spraying, 
credential stuffing, and targeted brute force attacks.

This is one of the highest-value detections for this 
environment given that a spike of 64 failed logins 
was detected in the first weeks of SOC operation — 
validating the need for this rule.

### SPL
```spl
index=windows EventCode=4625
| bucket _time span=5m
| stats count by _time, src_ip, user
| where count > 5
| table _time, src_ip, user, count
| sort -count
```

### Threshold
More than 5 failed logins from the same source IP 
targeting the same user within a 5-minute window.

### False Positives
- Users who have forgotten their password
- Service accounts with incorrect stored credentials
- IT admin testing authentication mechanisms

### Response
1. Identify the source IP (`src_ip`) and username (`user`)
2. Determine if the source IP is internal or external
3. Check if the targeted account exists and is active
4. If external source — consider temporary IP block
5. If internal source — contact the device owner
6. Escalate if targeting privileged accounts (admin, service accounts)

---

## RULE-002 — Malicious Office Macro Execution

```
Rule Name:       Malicious Office Macro Execution
MITRE Technique: T1566.001 — Phishing: Spearphishing Attachment
Severity:        Critical
Data Source:     index=sysmon, EventCode=1
Status:          Active
```

### Description
Detects Microsoft Office applications spawning unusual 
child processes. This is the definitive indicator of a 
malicious macro executing a payload. Attackers embed 
macros in Office documents delivered via phishing email — 
when the victim opens the document and enables macros, 
the macro spawns cmd.exe, PowerShell, or another payload 
delivery mechanism.

### SPL
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
  "*\\werfault.exe",
  "*\\WerFaultSecure.exe"
)
| table _time, host, User, ParentImage, Image, CommandLine
| sort -_time
```

### Threshold
Any single occurrence — this pattern is rarely legitimate 
and always warrants investigation.

### False Positives
- Some legitimate Office add-ins spawn helper processes
- PDF converters or macro-based automation tools
- Always examine the full `CommandLine` of the spawned process

### Response
1. Immediately isolate the affected endpoint from the network
2. Identify the Office document that triggered the macro
3. Examine the full command line of the spawned child process
4. Check for network connections from the same host 
   around the same timestamp (EventCode 3)
5. Check for files created after the macro executed 
   (EventCode 11)
6. Escalate to Incident Response if payload confirmed

---

## RULE-003 — Outbound Connection on Unusual Port

```
Rule Name:       Outbound Connection on Unusual Port
MITRE Technique: T1071 — Application Layer Protocol (C2)
Severity:        Medium
Data Source:     index=sysmon, EventCode=3
Status:          Active
```

### Description
Detects outbound network connections to destination ports 
outside the standard business-use set. Malware communicating 
with command-and-control (C2) infrastructure frequently 
uses non-standard ports to evade detection by port-based 
firewall rules and signature-based network monitoring.

### SPL
```spl
index=sysmon EventCode=3 Initiated=true
| where NOT (DestinationPort IN (
    80, 443, 53, 22, 25, 587, 143, 993,
    8080, 8443, 3389, 445, 135, 5985,
    5228, 5223, 5229, 853
  ))
| stats count by host, SourceIp, DestinationIp, 
  DestinationPort, Image
| where count > 3
| sort -count
```

### Threshold
More than 3 connections to the same unusual port from 
the same process — reduces single-occurrence noise while 
catching sustained beaconing patterns.

### False Positives
- Legitimate applications using non-standard ports
- Development tools and test environments
- VPN clients using proprietary ports
- Always examine the `Image` (process) making the connection

### Response
1. Identify the process (`Image`) making the connection
2. Determine if the destination IP is known/expected
3. Check if other hosts are connecting to the same destination
4. Search for other suspicious activity on the same host
   around the same time
5. If C2 suspected — isolate endpoint and escalate

---

## RULE-004 — Persistence via Run Registry Key

```
Rule Name:       Persistence via Run Registry Key
MITRE Technique: T1547.001 — Boot/Logon Autostart: Registry Run Keys
Severity:        High
Data Source:     index=sysmon, EventCode=13
Status:          Active
```

### Description
Detects modifications to Windows autostart registry keys. 
Any executable added to Run or RunOnce keys launches 
automatically at user logon — this is one of the most 
common and simple persistence mechanisms used by malware. 
Legitimate software also uses these keys, so context 
is critical.

### SPL
```spl
index=sysmon EventCode=13
TargetObject IN (
  "*\\Software\\Microsoft\\Windows\\CurrentVersion\\Run\\*",
  "*\\Software\\Microsoft\\Windows\\CurrentVersion\\RunOnce\\*",
  "*\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon\\*",
  "*\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\RunServices\\*"
)
| table _time, host, User, Image, TargetObject, Details
| sort -_time
```

### Threshold
Any modification to these keys warrants review — 
filter false positives contextually, not by threshold.

### False Positives
- Software installers writing legitimate autostart entries
- Browser auto-launch entries (Edge, Chrome)
- Security software registering autostart components
- Always check `Details` (the value written) and `Image` 
  (the process that wrote it)

### Response
1. Examine the `Details` field — what executable path 
   was written to the Run key?
2. Examine the `Image` field — what process wrote the entry?
3. Is the executable path in a suspicious location 
   (Temp, AppData, random folder)?
4. Is the writing process a known legitimate application?
5. If suspicious — check if the executable file exists 
   and examine it
6. Remove the Run key entry if confirmed malicious

---

## RULE-005 — LSASS Memory Access (Credential Dumping)

```
Rule Name:       LSASS Memory Access
MITRE Technique: T1003.001 — OS Credential Dumping: LSASS Memory
Severity:        Critical
Data Source:     index=sysmon, EventCode=10
Status:          Active
```

### Description
Detects processes reading LSASS (Local Security Authority 
Subsystem Service) memory. LSASS stores credential material 
in memory — tools like Mimikatz access it to extract 
plaintext passwords, NTLM hashes, and Kerberos tickets. 
This technique is used in almost every significant 
Windows credential theft operation.

### SPL
```spl
index=sysmon EventCode=10
TargetImage="*\\lsass.exe"
NOT SourceImage IN (
  "*\\MsMpEng.exe",
  "*\\svchost.exe",
  "*\\wininit.exe",
  "*\\csrss.exe",
  "*\\lsass.exe",
  "*\\SenseIR.exe",
  "*\\SecurityHealthService.exe"
)
| table _time, host, SourceImage, TargetImage, GrantedAccess
| sort -_time
```

### Threshold
Any non-excluded process accessing lsass.exe — 
this is a critical pattern with very few legitimate 
non-system causes.

### False Positives
- Security/AV software (Windows Defender, EDR agents) 
  legitimately access LSASS for monitoring
- Add confirmed legitimate security tools to the 
  exclusion list
- Specific `GrantedAccess` values like 0x1010, 0x1410, 
  0x143A indicate credential-extraction intent

### Response
1. This is a Critical severity finding — treat as 
   a confirmed incident until proven otherwise
2. Identify the `SourceImage` accessing LSASS
3. Is it a known security tool or an unknown/suspicious process?
4. Check for other suspicious activity on this host 
   in the preceding 30 minutes
5. Immediately isolate the endpoint
6. Escalate to full incident response

---

## RULE-006 — Timestomping Detection

```
Rule Name:       Timestomping Detection
MITRE Technique: T1070.006 — Indicator Removal: Timestomp
Severity:        Medium
Data Source:     index=sysmon, EventCode=2
Status:          Active
```

### Description
Detects changes to file creation timestamps. Attackers 
modify timestamps (timestomping) to make malicious files 
appear older than they are, complicating forensic timeline 
reconstruction. Legitimate installer activity occasionally 
triggers this, but large timestamp changes on files in 
user-writable locations are suspicious.

### SPL
```spl
index=sysmon EventCode=2
earliest=-24h latest=now
| eval time_diff=abs(strptime(CreationUtcTime,"%Y-%m-%d %H:%M:%S") - strptime(PreviousCreationUtcTime,"%Y-%m-%d %H:%M:%S"))
| where time_diff > 86400
| table _time, host, Image, TargetFilename, 
  PreviousCreationUtcTime, CreationUtcTime, time_diff
| sort -time_diff
```

### Threshold
Timestamp changes greater than 24 hours (86400 seconds).

### False Positives
- Microsoft Edge and other browsers modify installer 
  temp file timestamps during update processes
- File synchronization tools may normalize timestamps
- Software installers occasionally set files to 
  specific historical dates

### Response
1. Examine the `TargetFilename` — is it in a 
   suspicious location?
2. Examine the `Image` — what process changed the timestamp?
3. What was the file's previous creation date vs new date?
4. Does the change make the file appear significantly older?
5. Check if the file was created around the same time 
   as other suspicious activity

---

## RULE-007 — New Local Administrator Account Created

```
Rule Name:       New Local Administrator Account Created
MITRE Technique: T1136.001 — Create Account: Local Account
Severity:        High
Data Source:     index=windows, EventCode=4720 + 4732
Status:          Active
```

### Description
Detects creation of new local user accounts combined with 
addition to the Administrators group. Attackers create 
privileged local accounts for persistent access and 
lateral movement. The combination of both events 
(account created AND added to admin group) provides 
higher confidence than either event alone.

### SPL
```spl
index=windows (EventCode=4720 OR EventCode=4732)
| stats values(EventCode) as events,
        values(host) as hosts by user
| where mvcount(events) >= 2
| table _time, hosts, user, events
| sort -_time
```

### Threshold
Both EventCode 4720 (account created) AND 4732 
(added to group) occurring for the same username.

### False Positives
- IT administrators creating legitimate service accounts
- Software installers that create local accounts
- Correlate with change management records

### Response
1. Identify the new account name (`user`)
2. Which host was this account created on?
3. What process or user created it?
4. Is this account documented in change management?
5. If unauthorized — disable the account immediately
6. Investigate what other activity occurred on that 
   host around the same time

---

## RULE-008 — Offboarding Verification Alert

```
Rule Name:       Departed Employee Account Activity
MITRE Technique: T1078 — Valid Accounts
Severity:        Critical
Data Source:     index=windows, EventCode=4624
Status:          Planned
```

### Description
This rule was added to the roadmap following a real 
incident in this environment where a departed employee 
accessed company systems from an unauthorized device 
after their departure. The rule will alert on 
successful logins from accounts that have been 
flagged for deprovisioning.

### Implementation Requirement
Requires a lookup table (`offboarded_accounts.csv`) 
maintained by HR/IT containing accounts of departed 
employees pending full deprovisioning.

### SPL (Planned)
```spl
index=windows EventCode=4624
| lookup offboarded_accounts.csv username AS user 
  OUTPUT status
| where status="offboarded"
| table _time, host, user, src_ip, Logon_Type
| sort -_time
```

### Response
1. This is a Critical finding — any login from an 
   offboarded account is an unauthorized access event
2. Immediately disable the account
3. Review what the account accessed during this session
4. Determine how the account was still active 
   (offboarding process gap)
5. Report as a security incident

---

## Detection Coverage Matrix

| Rule ID | Name | Technique | Severity | Status | Data Source |
|---|---|---|---|---|---|
| RULE-001 | Brute Force Login | T1110 | High | Active | Windows 4625 |
| RULE-002 | Office Macro Execution | T1566.001 | Critical | Active | Sysmon 1 |
| RULE-003 | Unusual Port Outbound | T1071 | Medium | Active | Sysmon 3 |
| RULE-004 | Run Key Persistence | T1547.001 | High | Active | Sysmon 13 |
| RULE-005 | LSASS Access | T1003.001 | Critical | Active | Sysmon 10 |
| RULE-006 | Timestomping | T1070.006 | Medium | Active | Sysmon 2 |
| RULE-007 | Local Admin Created | T1136.001 | High | Active | Windows 4720/4732 |
| RULE-008 | Offboarded Account Login | T1078 | Critical | Planned | Windows 4624 |

---

## Known Detection Gaps

| Technique | Gap | Priority | Planned Approach |
|---|---|---|---|
| T1021 Remote Services | No network flow data | Medium | Enable Meraki Flows |
| T1053 Scheduled Tasks | No dedicated rule yet | Medium | Sysmon EventCode 1 parent analysis |
| T1055 Process Injection | EventCode 8 rule not yet deployed | High | Planned next sprint |
| T1078 Valid Accounts | Requires behavioral baseline | High | RULE-008 (Planned) |
| T1190 Exploit Public-Facing App | No application logs | Low | Future phase |

---

> **Confidentiality Note:** Detection thresholds, 
> exclusion lists, and specific organizational 
> context have been generalized. Actual production 
> tuning values are not documented here.
