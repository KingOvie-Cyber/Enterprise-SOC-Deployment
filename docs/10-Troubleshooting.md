# Troubleshooting
## Issues Encountered, Diagnosed, and Resolved

---

## Introduction

This document is one of the most valuable in this repository. 
It documents every significant technical issue encountered 
during the deployment and operation of the Enterprise SOC — 
including the exact symptoms, the diagnostic process used 
to isolate the root cause, and the resolution applied.

Real-world SOC deployments encounter problems that 
documentation, tutorials, and vendor guides do not cover. 
This document captures that lived experience.

Every issue documented here was a real obstacle encountered 
in this production environment.

---

## How to Use This Document

If you encounter an issue, work through the relevant 
section in this order:

```
1. Match your symptoms to an issue below
2. Follow the diagnostic steps to confirm root cause
3. Apply the resolution
4. Verify the fix using the verification command provided
```

---

## Issue 1 — Splunk Data Lost After Container Recreation

### Symptoms
- All indexed data disappeared from Splunk
- All searches returned zero results
- Splunk appeared healthy but had no history

### Root Cause
The Splunk Docker container was recreated (during 
troubleshooting) without persistent volume mounts 
configured. Splunk writes indexed data inside the 
container filesystem by default. When the container 
is removed, all data is permanently lost.

### Resolution
Configure named Docker volumes in docker-compose.yml:

```yaml
volumes:
  - splunk-var:/opt/splunk/var
  - splunk-etc:/opt/splunk/etc
```

With these volumes, data persists independently of 
the container lifecycle.

### Prevention
Never recreate the Splunk container without confirming 
persistent volumes are configured. Verify with:

```bash
docker inspect splunk | grep -A 20 "Mounts"
```

Both `splunk-var` and `splunk-etc` should appear 
as mounted volumes.

---

## Issue 2 — Splunk Web UI Restart Crashes Container

### Symptoms
- Clicking "Server Controls → Restart Splunk" in 
  Splunk Web caused the container to exit
- `docker ps` showed the container as stopped
- No error in Splunk — it simply went offline

### Root Cause
The official Splunk Docker image uses an Ansible-based 
entrypoint script to manage the container startup 
sequence. When Splunk Web triggers a restart internally, 
the Ansible entrypoint does not correctly re-supervise 
the restarted Splunk process — the container exits 
instead of restarting cleanly.

### Resolution
Always restart Splunk from the host using:

```bash
docker compose restart splunk
```

Never use Splunk Web's "Server Controls → Restart" 
in this containerized deployment.

### Verification
```bash
docker ps
# Confirm status shows "Up"
```

---

## Issue 3 — Sysmon Events Not Appearing in Splunk (errorCode=5)

### Symptoms
- Sysmon events visible in Windows Event Viewer locally
- Zero Sysmon events in Splunk
- `errorCode=5` appearing in splunkd.log

### Root Cause
The Splunk Universal Forwarder service did not have 
read permission on the Sysmon Operational event log 
channel. Windows Event Log channels have individual 
Access Control Lists (ACLs). The Sysmon channel, 
unlike standard Windows Event Log channels, requires 
explicit read permission to be granted to service accounts.

`errorCode=5` is Windows' "Access Denied" error.

### Resolution
Run in Command Prompt as Administrator on the endpoint:

```cmd
wevtutil set-log "Microsoft-Windows-Sysmon/Operational" /ca:O:BAG:SYD:(A;;0x1;;;BA)(A;;0x1;;;SY)(A;;0x1;;;LS)(A;;0x1;;;NS)(A;;0x1;;;S-1-5-32-573)
```

Restart the forwarder:

```powershell
Restart-Service SplunkForwarder
```

### Verification
```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 30
# Confirm errorCode=5 no longer appears for Sysmon channel
```

```spl
index=sysmon EventCode=1
| head 5
# Confirm events appear in Splunk
```

---

## Issue 4 — Repeating DeploymentClient Handshake Errors

### Symptoms
- splunkd.log flooded with repeating messages:
  `DC:DeploymentClient - channel=tenantService/handshake 
  Will retry sending handshake message to DS; err=not_connected`
- Winsock error 10060 appearing alongside
- Log noise making real errors difficult to identify

### Root Cause
The Universal Forwarder's Deployment Client feature 
was active and configured with the Splunk Enterprise 
server's data port (9997) as the Deployment Server 
address. The forwarder was attempting to initiate 
a deployment management handshake on a port that 
only accepts raw event data — causing continuous 
connection failures and retry loops.

### Resolution
Create or update deploymentclient.conf:

```ini
[deployment-client]
disabled = true
```

Restart the forwarder:

```powershell
Restart-Service SplunkForwarder
```

### Verification
```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 30
# Confirm DC:DeploymentClient errors no longer appear
```

---

## Issue 5 — inputs.conf Typo Causing Silent Input Failure

### Symptoms
- Windows Event Log inputs configured in inputs.conf
- Zero Security/System/Application events in Splunk
- No errors in splunkd.log

### Root Cause
A typo in inputs.conf: `disabled = fasle` instead 
of `disabled = false`. Splunk does not recognize 
`fasle` as a valid boolean value and silently 
ignores (or disables) the input without logging 
an error. This is a classic silent failure.

### Resolution
Correct the typo in inputs.conf:

```ini
# Wrong
disabled = fasle

# Correct
disabled = false
```

Restart the forwarder:

```powershell
Restart-Service SplunkForwarder
```

### Prevention
Always verify inputs.conf content visually after editing:

```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"
```

Read every line. Typos in boolean values cause silent 
failures that are very difficult to diagnose.

---

## Issue 6 — Sysmon EventCode 3 (Network) Not Appearing

### Symptoms
- EventCode 1 (Process Creation) events flowing normally
- Zero EventCode 3 (Network Connection) events in Splunk
- `Get-WinEvent` for Sysmon EventCode 3 also returned nothing locally

### Root Cause
The Sysmon configuration had no `<NetworkConnect>` 
section at all. Sysmon does not log network connections 
unless explicitly configured to do so. The initial 
configuration only included ProcessCreate and FileCreate 
rules — network monitoring was simply never enabled.

**Diagnostic insight:** The fact that EventCode 3 
was absent locally (not just in Splunk) pointed 
conclusively to a Sysmon configuration issue, 
not a forwarding or Splunk issue. This distinction 
saved significant troubleshooting time.

### Resolution
Apply the Olaf Hartong modular Sysmon configuration, 
which includes NetworkConnect rules:

```powershell
sysmon64.exe -c sysmonconfig.xml
```

Verify the NetworkConnect section is now present:

```powershell
sysmon64.exe -c | Select-String -Pattern "NetworkConnect" -Context 0,5
```

### Verification
```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 20 | Where-Object {$_.Id -eq 3}
# Should now return network connection events
```

```spl
index=sysmon EventCode=3
| head 10
# Should return events in Splunk
```

---

## Issue 7 — All Sysmon Forwarding Stopped After Config Update

### Symptoms
- All Sysmon events disappeared from Splunk (including EventCode 1)
- EventCode 1 had been working correctly before the config update
- Sysmon service was running
- splunkd.log showed TailReader only reading tracker.log, not the Sysmon channel

### Root Cause
When `sysmon64.exe -c sysmonconfig.xml` was run to 
apply a new configuration, the Sysmon service 
briefly restarted. The Universal Forwarder's internal 
fishbucket (read position bookmark) for the Sysmon 
Operational event log channel lost its position 
during the restart. The forwarder did not automatically 
resume reading from the Sysmon channel after the 
service came back up.

### Resolution
Restart the Universal Forwarder service after any 
Sysmon configuration update:

```powershell
Restart-Service SplunkForwarder
```

Wait 60-90 seconds, then verify events are flowing.

### Prevention
Whenever `sysmon64.exe -c` is run to apply a 
configuration change, always follow with a 
forwarder restart as a standard procedure.

---

## Issue 8 — "Set Token Value to Render Visualization" in Dashboard

### Symptoms
- Dashboard panels showing placeholder icon instead of data
- Error message: "Set token value to render visualization"
- The underlying search works correctly in Search & Reporting

### Root Cause
The panel's visualization was bound to a time range 
token (e.g., `$global_time.earliest$`) but either:
- The SPL query did not reference that token, so 
  Splunk had no time range to apply, OR
- A second, duplicate time range input was created 
  with a different token name, and the panel was 
  bound to the wrong token

### Resolution A — Static Time Range
For panels whose SPL does not need a time bound 
(e.g., `eventcount` or `rest` queries):

In the panel data source settings, change 
**Time range** from `Default` to `Static` and 
set a fixed time range.

### Resolution B — Token Reference in SPL
For time-bounded panels, ensure the SPL explicitly 
references the token:

```spl
index=sysmon EventCode=1
earliest=$global_time.earliest$ latest=$global_time.latest$
| stats count
```

### Resolution C — Remove Duplicate Input
If a duplicate time range picker was accidentally 
created:
1. Click the duplicate input
2. Delete it from the dashboard
3. Verify all panels reference the single 
   `global_time` token

---

## Issue 9 — Meraki Field "signature" Returning No Results

### Symptoms
- `stats count by signature` returned zero statistics
- Raw events confirmed present (Events tab showed 73,000+ events)
- No error message — just empty statistics

### Root Cause
Meraki logs use `type` as the event category field, 
not `signature`. The assumption that Meraki would 
use the same field name as other firewall/SIEM 
integrations was incorrect.

### Resolution
Use the correct field name:

```spl
index=meraki sourcetype=meraki_syslog
| stats count by type
| sort -count
```

### Lesson
Always inspect raw events before building query 
logic based on assumed field names. The `| head 1 | table _raw` 
pattern shows the actual raw event content, 
revealing the real field structure.

---

## Issue 10 — ESXi Console Clipboard Not Working

### Symptoms
- Unable to paste text into the Rocky Linux VM console
- Copy-paste worked nowhere inside the VM
- VMware Tools (`vmtoolsd`) was confirmed running

### Root Cause
ESXi's remote console (web-based HTML5 console or 
VMware Remote Console) does not support clipboard 
sharing between the host and guest VM — regardless 
of VMware Tools status. This is a fundamental 
limitation of the ESXi remote console delivery 
mechanism, not a Tools configuration issue.

This differs from VMware Workstation, which uses 
a different local display protocol that supports 
clipboard sharing.

### Resolution
Access Splunk Web and other services directly 
from the host machine browser rather than through 
the ESXi console:

```
http://<soc-server-ip>:8000
```

Use SSH for terminal access to the Rocky Linux VM — 
SSH clients on the host machine support 
full copy-paste functionality:

```bash
ssh socadmin@<soc-server-ip>
```

Reserve the ESXi web console for situations where 
the network is unavailable or SSH is not accessible.

---

## Diagnostic Flowchart — Events Not in Splunk

```
Events not appearing in Splunk
         |
         ├── Does the event exist locally?
         │   (Get-WinEvent / Event Viewer on endpoint)
         │   |
         │   ├── NO → Sysmon config issue
         │   │         (missing event type rule)
         │   │         Fix: apply correct sysmonconfig.xml
         │   |
         │   └── YES → Continue...
         |
         ├── Is the network path open?
         │   (Test-NetConnection <splunk-ip> -Port 9997)
         │   |
         │   ├── FALSE → Network/firewall issue
         │   │           Fix: check firewall rules,
         │   │           Docker port mapping
         │   |
         │   └── TRUE → Continue...
         |
         ├── Is the forwarder service running?
         │   (Get-Service SplunkForwarder)
         │   |
         │   ├── STOPPED → Start the service
         │   │   Restart-Service SplunkForwarder
         │   |
         │   └── RUNNING → Continue...
         |
         ├── Check splunkd.log for errors
         │   (Get-Content ...splunkd.log -Tail 50)
         │   |
         │   ├── errorCode=5 → ACL permission issue
         │   │   Fix: wevtutil command (Issue 3)
         │   |
         │   ├── DeploymentClient errors → Disable it
         │   │   Fix: deploymentclient.conf (Issue 4)
         │   |
         │   └── No errors, TailReader normal → Continue...
         |
         └── Check inputs.conf for typos
             (disabled = false, not fasle)
             (index = sysmon, must match existing index)
             (renderXml = true for Sysmon)
```

---

> **Confidentiality Note:** All specific IP addresses, 
> hostnames, usernames, and organizational identifiers 
> have been sanitized. Issues are documented as 
> methodology and root cause analysis — not as 
> a disclosure of the organization's infrastructure.
