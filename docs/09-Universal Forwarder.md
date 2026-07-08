# Universal Forwarder
## Splunk Universal Forwarder — Deployment and Configuration

---

## Introduction

This document describes the deployment, configuration, and 
troubleshooting of the Splunk Universal Forwarder on Windows 
endpoints in the Enterprise SOC environment.

The Splunk Universal Forwarder is a lightweight agent that 
reads from configured data sources on a Windows endpoint 
and ships that data to Splunk Enterprise over TCP port 9997. 
It is the bridge between endpoint telemetry (Sysmon, Windows 
Event Logs) and the SIEM.

---

## Architecture Role

```
Windows Endpoint
      |
      ├── Sysmon → Windows Event Log (Sysmon/Operational)
      ├── Windows → Windows Event Log (Security/System/App)
      |
      └── Splunk Universal Forwarder
          ├── Reads: Sysmon/Operational channel
          ├── Reads: Security Event Log
          ├── Reads: System Event Log
          ├── Reads: Application Event Log
          └── Ships: TCP 9997 → Splunk Enterprise
```

The Universal Forwarder is free regardless of the Splunk 
Enterprise license tier. There is no per-endpoint cost 
for the forwarder itself.

---

## Key Concepts

### Universal Forwarder vs Heavy Forwarder

The Universal Forwarder is a lightweight shipper — it 
reads data and sends it to Splunk with minimal processing. 
It does not perform index-time field extraction or run 
full Splunk searches. This is appropriate for endpoint 
deployment where resource consumption must be minimal.

### Configuration File Precedence

Splunk applies configuration from multiple locations in 
a defined precedence order:

```
Higher precedence (wins conflicts):
  etc/system/local/          ← where we configure
  etc/apps/<app>/local/
  etc/apps/<app>/default/
  etc/system/default/        ← Splunk defaults (lower precedence)
```

Always place custom configuration in `etc/system/local/` 
to ensure it takes precedence over defaults.

---

## Installation

### Prerequisites

- Windows endpoint with administrative access
- Splunk Universal Forwarder MSI installer
- Network connectivity to Splunk server on TCP 9997
- Sysmon already installed (see [Sysmon.md](Sysmon.md))

### Installation Command

```powershell
# Run as Administrator
msiexec.exe /i splunkforwarder-x64.msi `
  AGREETOLICENSE=Yes `
  RECEIVING_INDEXER="<splunk-server-ip>:9997" `
  WINEVENTLOG_APP_ENABLE=1 `
  WINEVENTLOG_SEC_ENABLE=1 `
  WINEVENTLOG_SYS_ENABLE=1 /quiet
```

### Verify Installation

```powershell
Get-Service SplunkForwarder
```

Expected output: `Status: Running`

---

## Configuration Files

All configuration files are located at:
```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\
```

Three files require configuration:
1. `inputs.conf` — what data to collect and where to route it
2. `outputs.conf` — where to send collected data
3. `deploymentclient.conf` — disable deployment server (important)

---

## inputs.conf

This file defines what data sources the forwarder reads 
and which Splunk index each source is routed to.

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = sysmon
disabled = false
renderXml = false
sourcetype = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational

[WinEventLog://Security]
index = windows
disabled = false
sourcetype = WinEventLog:Security

[WinEventLog://System]
index = windows
disabled = false
sourcetype = WinEventLog:System

[WinEventLog://Application]
index = windows
disabled = false
sourcetype = WinEventLog:Application
```

### Configuration Explained

**[WinEventLog://Microsoft-Windows-Sysmon/Operational]**
The exact channel name for Sysmon's event log. This must 
match precisely — any variation will cause the forwarder 
to silently fail to read from this channel.

**index = sysmon**
Routes events from this channel to the `sysmon` index 
in Splunk Enterprise. This must match an index that 
actually exists on the Splunk server.

**disabled = false**
Must be explicitly set to `false`. If this is missing 
or set to `true`, the input is inactive. A common error 
is a typo here — `fasle` instead of `false` — which 
Splunk treats as an unrecognized value and may default 
to disabled behavior.

**renderXml = true**
Preserves Sysmon's XML event structure when forwarding. 
This is essential for correct field extraction in Splunk. 
Without this setting, Sysmon events arrive as flat text 
and many fields (process names, hashes, network details) 
are not automatically extracted.In the configuration, 
it is set to **false** because when logs arrives in splunk 
they come in as **raw-xml** which can be difficult to serach for
using the **SPL** Hence the **false** in the configuration.



---

## outputs.conf

This file defines where the forwarder sends collected data.

```ini
[tcpout]
defaultGroup = splunk_indexer

[tcpout:splunk_indexer]
server = <splunk-server-ip>:9997
compressed = true
```

### Configuration Explained

**defaultGroup = splunk_indexer**
Names the output group. Must match the stanza name 
used below (`[tcpout:splunk_indexer]`).

**server = <splunk-server-ip>:9997**
The IP address and port of the Splunk Enterprise 
receiving endpoint. Replace with your actual server IP.

**compressed = true**
Compresses data in transit, reducing bandwidth 
consumption. Does not affect the data content or 
index destination.

---

## deploymentclient.conf

This file must be created to disable the Deployment 
Client functionality.

```ini
[deployment-client]
disabled = true
```

### Why This Is Critical

The Splunk Universal Forwarder includes a Deployment 
Client feature that connects to a Deployment Server 
for centralized configuration management. If this 
feature is not explicitly disabled and no Deployment 
Server is configured, the forwarder attempts to contact 
a non-existent server on a continuous retry loop.

This retry loop was observed to:
- Generate constant error messages in splunkd.log 
  (DC:DeploymentClient handshake errors)
- Consume connection resources that interfered with 
  actual log forwarding
- Mask the real cause of log forwarding failures 
  during troubleshooting

**Disabling the Deployment Client is mandatory in 
this environment where centralized deployment server 
management is not in use.**

---

## Applying Configuration Changes

After editing any configuration file:

```powershell
Restart-Service SplunkForwarder
```

Wait 60-90 seconds after restart before checking 
Splunk for new events. The forwarder needs time to 
re-establish its connection and resume reading from 
where it last left off.

---

## Verification

### Verify Forwarder Service Status

```powershell
Get-Service SplunkForwarder
```

### Verify Network Connectivity to Splunk

```powershell
Test-NetConnection <splunk-server-ip> -Port 9997
```

Expected: `TcpTestSucceeded : True`

If this returns `False`, the issue is network/firewall 
— not forwarder configuration. Check:
- Firewall rules on both the endpoint and Splunk server
- Docker port mapping on the Splunk host (9997 must be 
  explicitly mapped)
- Network routing between endpoint and SOC server VLAN

### Verify Configuration Files Are Correct

```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"
Get-Content "C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf"
Get-Content "C:\Program Files\SplunkUniversalForwarder\etc\system\local\deploymentclient.conf"
```

### Check Forwarder Internal Logs

```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 50
```

Look for:
- `TcpOutEloop ... Connected to idx=<ip>:9997` — successful connection
- `errorCode=5` — permission denied reading a log channel
- `DC:DeploymentClient ... handshake ... not_connected` — 
  Deployment Client not disabled (apply deploymentclient.conf fix)

### Verify in Splunk (End-to-End Confirmation)

```spl
index=sysmon
| head 10
```

```spl
index=windows EventCode=4624
| head 10
```

If both return recent events, the forwarder is fully 
operational end-to-end.

---

## Troubleshooting Guide

### Issue: Service Running but No Events in Splunk

**Diagnosis sequence:**

1. Confirm Sysmon events exist locally:
```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
```

2. Confirm network path is open:
```powershell
Test-NetConnection <splunk-server-ip> -Port 9997
```

3. Check splunkd.log for errors:
```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 100 | Select-String -Pattern "ERROR|Sysmon"
```

4. Confirm inputs.conf has no typos:
- `disabled = false` (not `fasle`)
- `index = sysmon` (must match existing index on Splunk)
- `renderXml = true` (not missing)

### Issue: errorCode=5 in splunkd.log

The forwarder does not have permission to read the 
Sysmon Operational event log channel.

**Fix:** Run in Command Prompt as Administrator:
```cmd
wevtutil set-log "Microsoft-Windows-Sysmon/Operational" /ca:O:BAG:SYD:(A;;0x1;;;BA)(A;;0x1;;;SY)(A;;0x1;;;LS)(A;;0x1;;;NS)(A;;0x1;;;S-1-5-32-573)
```

Restart the forwarder after applying:
```powershell
Restart-Service SplunkForwarder
```

### Issue: Repeating DeploymentClient Handshake Errors

The Deployment Client feature is active and attempting 
to connect to a non-existent server.

**Fix:** Create or update deploymentclient.conf:
```ini
[deployment-client]
disabled = true
```

Restart the forwarder after applying.

### Issue: Forwarder Login Failed When Running CLI Commands

The Universal Forwarder has its own local admin account 
separate from the Splunk Enterprise server admin account. 
These are different credentials and must not be confused.

The default Universal Forwarder local admin credentials 
may be `admin/changeme` if never changed.

**Note:** CLI authentication issues with the forwarder 
do not affect log forwarding. Forwarding uses TCP data 
channel (outputs.conf), not the management API.

### Issue: Multiple outputs.conf Files Causing Conflicts

Check all outputs.conf locations:
```powershell
Get-ChildItem -Path "C:\Program Files\SplunkUniversalForwarder\etc" -Recurse -Filter "outputs.conf"
```

The file in `etc\system\local\` takes precedence. 
If auto-generated files in `etc\apps\` or 
`etc\system\default\` conflict, the `system\local` 
file should win — but verify the active server 
address is correct in whichever file has precedence.

---

## Scaling to Multiple Endpoints

When deploying the Universal Forwarder across 16-18 
endpoints, the following approach was used:

1. Configure one pilot endpoint fully and verify 
   end-to-end before scaling
2. Apply the same three configuration files 
   (`inputs.conf`, `outputs.conf`, `deploymentclient.conf`) 
   to each endpoint
3. Verify each endpoint individually in Splunk before 
   moving to the next

A future improvement is to use a deployment automation 
tool (Group Policy, Ansible, or even Splunk's own 
Deployment Server with proper configuration) to 
push configuration at scale rather than manually 
configuring each endpoint.

---

## Lessons Learned

**The `disabled = fasle` typo:** During initial 
configuration, a typo (`fasle` instead of `false`) 
in `inputs.conf` caused the Sysmon input to remain 
inactive. Splunk did not error — it simply did 
not read from that channel. This class of silent 
failure is particularly difficult to diagnose. 
Always verify configuration files visually after 
editing.

**DeploymentClient interference:** The Deployment 
Client retry loop was initially suspected as the 
cause of log forwarding failures. After systematic 
isolation (network test passed, service running, 
config files correct, events exist locally), 
the DeploymentClient was identified as noise 
masking the real issues. Disabling it cleaned 
up logs and simplified troubleshooting 
significantly.

**Reinstall as a valid troubleshooting step:** 
When a previous Deployment Server configuration 
was baked into the forwarder's state in a way 
that survived config file changes, a clean 
reinstall resolved the issue faster than 
continued debugging. Sometimes the cleanest 
path forward is a fresh start.

**Test network connectivity before blaming config:** 
On one endpoint, events were not arriving in 
Splunk despite apparently correct configuration. 
`Test-NetConnection` to port 9997 returned 
`False` — the endpoint simply could not reach 
the Splunk server due to a network path issue. 
Network connectivity is the first thing to 
verify, not the last.

---

> **Confidentiality Note:** Server IP addresses, 
> hostnames, and endpoint-specific identifiers 
> are represented as placeholders throughout 
> this document. Actual values are not 
> documented in this repository.
