# Sysmon
## Windows Endpoint Telemetry — Deployment and Configuration

---

## Introduction

This document describes the deployment, configuration, and 
operational use of System Monitor (Sysmon) across Windows 
endpoints in the Enterprise SOC environment.

Sysmon is a Windows system service and device driver that 
logs detailed system activity to the Windows Event Log. 
It provides visibility into endpoint behavior that standard 
Windows Event Logs simply cannot capture — making it one 
of the most valuable tools available to a blue team operating 
without a commercial EDR solution.

---

## Why Sysmon

Standard Windows Event Logs capture authentication events, 
account changes, and policy modifications — but they miss 
critical attacker activity:

| Activity | Standard Windows Logs | Sysmon |
|---|---|---|
| Process creation with full command line | Partial | Full |
| Network connections with process context | No | Yes |
| File creation events | No | Yes |
| Registry modifications | No | Yes |
| DNS queries | No | Yes |
| DLL/image loads | No | Yes |
| Process injection indicators | No | Yes |
| Timestomping detection | No | Yes |

Sysmon closes these gaps without requiring a commercial 
EDR license, making it the foundational endpoint telemetry 
tool for this SOC environment.

---

## Sysmon Event Codes Reference

| EventCode | Event Type | Detection Value | Volume |
|---|---|---|---|
| 1 | Process Creation | Very High | High |
| 2 | File Creation Time Changed (timestomping) | High | Low |
| 3 | Network Connection | Very High | Very High |
| 5 | Process Terminated | Medium | High |
| 7 | Image/DLL Loaded | High | Very High |
| 8 | CreateRemoteThread (injection) | Very High | Low |
| 10 | Process Access (credential dumping) | Very High | Low |
| 11 | File Created | High | High |
| 12 | Registry Object Added/Deleted | High | Medium |
| 13 | Registry Value Set | High | Medium |
| 14 | Registry Object Renamed | High | Low |
| 15 | FileCreateStreamHash (ADS) | High | Low |
| 17 | Pipe Created | Medium | Low |
| 18 | Pipe Connected | Medium | Low |
| 22 | DNS Query | High | Very High |

---

## Configuration Approach

### Configuration Selection

The Olaf Hartong modular Sysmon configuration was selected 
as the baseline configuration. This community-maintained 
configuration provides:

- MITRE ATT&CK-aligned detection coverage
- Noise reduction rules for known-good system processes
- High-fidelity event capture across all critical event types
- Active community maintenance and updates

Source:
```
https://raw.githubusercontent.com/olafhartong/sysmon-modular/master/sysmonconfig.xml
```

### Why Not a Custom Configuration from Scratch

Building a custom Sysmon configuration from scratch is 
time-intensive and requires deep knowledge of Windows 
internals to avoid both coverage gaps (missing real threats) 
and excessive noise (drowning the SIEM with low-value events). 

The Olaf Hartong config represents years of community 
refinement and real-world threat hunting experience. It 
was used as the baseline with specific tuning applied 
for this environment's volume requirements.

---

## Installation

### Prerequisites

- Administrative rights on the Windows endpoint
- Sysmon binary (download from Microsoft Sysinternals)
- Sysmon configuration XML file

### Step 1 — Grant Sysmon Log Read Permission

This step is critical and is frequently missed. Without 
this permission, the Splunk Universal Forwarder cannot 
read the Sysmon Operational event log channel.

Run in Command Prompt or Powershell as Administrator, i recommend Powershell:

```cmd
wevtutil set-log "Microsoft-Windows-Sysmon/Operational" /ca:O:BAG:SYD:(A;;0x1;;;BA)(A;;0x1;;;SY)(A;;0x1;;;LS)(A;;0x1;;;NS)(A;;0x1;;;S-1-5-32-573)
```

This grants read access to:
- Built-in Administrators (BA)
- Local System (SY)
- Local Service (LS)
- Network Service (NS)
- Event Log Readers group (S-1-5-32-573)

### Step 2 — Install Sysmon with Configuration

```powershell
# Run as Administrator
sysmon64.exe -accepteula -i sysmonconfig.xml
```

### Step 3 — Verify Installation

```powershell
# Confirm service is running
Get-Service Sysmon64

# Confirm events are being generated
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10
```

### Step 4 — Verify Configuration Loaded

```powershell
sysmon64.exe -c
```

This displays the currently active configuration. Verify 
that key sections (NetworkConnect, ProcessCreate, FileCreate, 
RegistryEvent, DnsQuery) are present.

---

## Configuration Management

### Reload Configuration Without Reinstalling

When the configuration XML is updated, reload without 
uninstalling and reinstalling Sysmon:

```powershell
sysmon64.exe -c sysmonconfig.xml
```

### Uninstall Sysmon

```powershell
sysmon64.exe -u force
```

### Check Active Configuration (Filtered)

```powershell
# Check NetworkConnect rules specifically
sysmon64.exe -c | Select-String -Pattern "NetworkConnect" -Context 0,10

# Check ProcessCreate rules
sysmon64.exe -c | Select-String -Pattern "ProcessCreate" -Context 0,10
```

---

## Key Configuration Sections

### ProcessCreate (EventCode 1)

```xml
<ProcessCreate onmatch="exclude">
  <!-- Exclude known-good noisy processes -->
  <Image condition="end with">svchost.exe</Image>
</ProcessCreate>
```

Process creation is the highest-value Sysmon event type. 
Every process execution is captured with:
- Full executable path
- Complete command line arguments
- Parent process information
- User context
- Process and parent process hashes

### NetworkConnect (EventCode 3)

```xml
<NetworkConnect onmatch="exclude">
  <!-- Exclude high-volume known-good connections -->
  <Image condition="end with">svchost.exe</Image>
</NetworkConnect>
```

Network connection logging was initially missing from 
the configuration. After identifying the gap (the 
NetworkConnect section was absent entirely from the 
initial deployment), the Olaf Hartong config was 
applied to restore coverage.

### FileCreate (EventCode 11)

```xml
<FileCreate onmatch="include">
  <!-- Focus on high-value locations -->
  <TargetFilename condition="contains">\Startup\</TargetFilename>
  <TargetFilename condition="contains">\Temp\</TargetFilename>
  <TargetFilename condition="end with">.exe</TargetFilename>
  <TargetFilename condition="end with">.dll</TargetFilename>
</FileCreate>
```

### RegistryEvent (EventCodes 12, 13, 14)

Registry monitoring focuses on persistence-related keys:

```xml
<RegistryEvent onmatch="include">
  <TargetObject condition="contains">Run</TargetObject>
  <TargetObject condition="contains">RunOnce</TargetObject>
  <TargetObject condition="contains">Services</TargetObject>
</RegistryEvent>
```

### DnsQuery (EventCode 22)

```xml
<DnsQuery onmatch="exclude">
  <!-- Exclude high-volume known-good domains -->
  <QueryName condition="end with">microsoft.com</QueryName>
  <QueryName condition="end with">windows.com</QueryName>
</DnsQuery>
```

---

## Splunk Integration

Sysmon events are shipped to Splunk via the Universal 
Forwarder. Full forwarder configuration is in 
[Universal-Forwarder.md](Universal-Forwarder.md).

### inputs.conf (on endpoint)

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = sysmon
disabled = false
renderXml = false
sourcetype = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
```

The `renderXml = true` setting is important — it preserves 
Sysmon's XML event structure, enabling Splunk to 
automatically extract all fields (process names, paths, 
hashes, network details) without requiring manual 
field extraction.

---

## Detection Coverage by MITRE ATT&CK

| MITRE Technique | Description | Sysmon EventCode |
|---|---|---|
| T1059 | Command and Scripting Interpreter | 1 |
| T1055 | Process Injection | 8 |
| T1003 | OS Credential Dumping (LSASS) | 10 |
| T1547 | Boot/Logon Autostart (Run keys) | 13 |
| T1566 | Phishing — Macro execution | 1 (parent-child) |
| T1071 | C2 via Application Layer Protocol | 3, 22 |
| T1070.006 | Timestomping | 2 |
| T1574 | DLL Side-Loading | 7 |
| T1105 | Ingress Tool Transfer | 11, 3 |
| T1021 | Remote Services (lateral movement) | 3 |

---

## Operational Considerations

### Volume Management

Sysmon generates significant event volume, particularly 
EventCode 3 (network connections) and EventCode 22 
(DNS queries). On a normally-active workstation:

```
Estimated Sysmon volume: 50-150 MB/day per endpoint
At 16-18 endpoints:      800 MB - 2.7 GB/day
```

Tuning the configuration to exclude known-good processes 
and domains is necessary to keep volume manageable within 
the Splunk index sizing limits.

### Known Noise Sources

These legitimate processes generate high Sysmon event 
volume and are filtered in detection logic:

- `splunk-powershell.exe` — Splunk Universal Forwarder helper
- `splunk-regmon.exe` — Splunk Universal Forwarder registry monitor
- `splunk-netmon.exe` — Splunk Universal Forwarder network monitor
- `MsMpEng.exe` — Windows Defender
- `svchost.exe` — Windows service host (many instances)
- `MicrosoftEdgeUpdate.exe` — Edge browser auto-updater

Detection queries should explicitly exclude these to 
avoid alert fatigue.

### User Field Resolution

Processes running as `NT AUTHORITY\SYSTEM` will display 
the User field as `NOT_TRANSLATED NT AUTHORITY\SYSTEM` 
in Splunk. This is expected behavior — it indicates 
a service or scheduled task running under the system 
account rather than a named user. It is not a data 
quality issue.

---

## Troubleshooting

### Sysmon Events Not Appearing in Splunk

**Step 1 — Verify Sysmon is generating events locally:**
```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
```

If this returns nothing, Sysmon is not running or the 
configuration is empty/invalid.

**Step 2 — Verify the log channel permission:**
If events exist locally but not in Splunk, the most 
common cause is missing read permission on the Sysmon 
Operational log channel. Re-apply the `wevtutil` command 
from the Installation section.

**Step 3 — Verify the forwarder inputs.conf:**
```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"
```
Confirm `disabled = false` and `index = sysmon`.

**Step 4 — Check forwarder logs:**
```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 50 | Select-String -Pattern "Sysmon"
```

### Specific Event Code Not Appearing

If a specific event code (e.g. EventCode 3) is missing 
from Splunk despite other codes appearing:

```powershell
# Check if that event code is being generated locally
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 20 | Where-Object {$_.Id -eq 3}
```

If nothing is returned locally, the Sysmon configuration 
does not have that event type enabled. Check the active 
configuration:

```powershell
sysmon64.exe -c | Select-String -Pattern "NetworkConnect" -Context 0,5
```

If the section is empty, the event type is not configured. 
Apply the Olaf Hartong configuration to restore coverage.

---

## Lessons Learned

**The log channel permission step is non-negotiable:** 
Without the `wevtutil` ACL command, the Splunk Universal 
Forwarder receives `errorCode=5` (Access Denied) when 
attempting to read the Sysmon Operational channel. 
This fails silently — the forwarder appears healthy, 
but no Sysmon events arrive in Splunk. This was one 
of the most time-consuming issues in the initial 
deployment.

**NetworkConnect was missing from initial config:** 
The initial Sysmon deployment used a configuration 
that had no `<NetworkConnect>` section at all. This 
meant EventCode 3 was never generated — not a 
forwarding issue, but a configuration gap. Discovered 
through systematic troubleshooting: events weren't 
in Splunk, events weren't in the local Windows Event 
Log, therefore Sysmon itself wasn't generating them, 
therefore the issue was the configuration.

**Applying a new config can briefly disrupt forwarding:** 
When the Olaf Hartong configuration was applied via 
`sysmon64.exe -c`, the Sysmon service briefly restarts. 
This can cause the Universal Forwarder to lose its 
read position on the event log channel temporarily. 
Restarting the forwarder service after a config change 
resolves this.

---

> **Confidentiality Note:** Endpoint hostnames, 
> usernames, and organization-specific configuration 
> details have been sanitized throughout this document.
