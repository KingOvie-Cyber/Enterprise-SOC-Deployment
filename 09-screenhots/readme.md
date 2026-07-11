# Screenshots

Visual evidence of the SOC build in operation — live Splunk dashboards covering ISO 27001 control evidence, network monitoring, and endpoint/Sysmon telemetry. These support the architecture and SPL documentation elsewhere in this repo; the configs and searches are the primary source of truth, but these confirm the environment actually runs and produces real data.

## Redaction Notice

Screenshots in this folder have been reviewed and redacted prior to publishing — internal hostnames, usernames, and identifying file paths have been removed or blurred from all table columns (`host`, `SourceImage`/`TargetImage` paths, `C:\Users\<name>\...` entries) before upload.

## Naming Convention

```
<category>-<component>-<short-description>.png
```

## Index

| File | Dashboard / Panel | Shows | Related Doc |
|---|---|---|---|
| `soc-dashboard-overview.png` | SOC Overview → **SOC Dashboard** tab | Top-line KPIs (total events today, failed logins today, active hosts 24h), event-over-time trend split by sourcetype (meraki/sysmon/windows), event source pie chart, recent Sysmon process creations table | `dashboards/SOC-Overview.md` |
| `soc-dashboard-overview2.png` | SOC Overview → **SUMMARY / OVERVIEW PANELS** tab | Event volume by EventCode (horizontal bar), event volume broken out by EventCode + host, and a "which event codes have never appeared" gap-check table | `dashboards/SOC-Overview.md` |
| `sysmon_process_creation_event1_5_.png` | SOC Overview → **Sysmon-Process Creation (EventCode 1,5)** tab | Top processes executed (cmd.exe, tasklist.exe, updater processes), process creation volume over time, suspicious parent→child process relationships (Office spawning children), recent process creation table | `spl/Sysmon-EventCode-Searches.md` (Event Code 1 section) |
| `Screenshot_2026-07-07_120534.png` | SOC Overview → **Sysmon-Useful Event Codes (EventCode 2,8,10,15,17,18)** tab | File creation time changed (EventCode 2), CreateRemoteThread/process injection indicators (EventCode 8), Process Access (EventCode 10), File Create Stream Hash (EventCode 15), Pipe Created/Connected (EventCode 17/18) | `spl/Sysmon-EventCode-Searches.md` (Event Code 8 section) |
| `sysmon_process_creation_event2_8_10_15_17_18_.png` | SOC Overview → **Sysmon-Useful Event Codes** tab (scrolled view) | Same panel set as above — file creation time changed, remote thread/injection indicators, process access table showing lsass.exe access from csrss.exe/wininit.exe | `spl/Sysmon-EventCode-Searches.md` (Event Code 8 section) |
| `sysmon_process_creation_event3_22_.png` | SOC Overview → **Sysmon-Network Activity (Event Code 3,22)** tab | Network connection volume over time, top destination IPs and ports, outbound connections on unusual ports, DNS query panels (Event 22) | `spl/Sysmon-EventCode-Searches.md` (Event Code 3 and 22 sections) |
| `sysmon_process_creation_event11_12_13_14_7_.png` | SOC Overview → **Sysmon-File&Persistence (EventCode 11,12,13,14,7)** tab | Recent files created, file creation volume over time, files created in sensitive locations, registry events (12/13/14), registry persistence keys (Run/RunOnce), Image/DLL Loaded and Unsigned DLLs panels | `spl/Sysmon-EventCode-Searches.md` (Event Code 11, 13, 7 sections) |
| `ISO_27001_Technological_Controls_Evidence.png` | **ISO 27001 Technological Controls Evidence** dashboard (top half) | Active log sources mapped to A.8.15, log retention policy per index, monitoring source health (A.8.16), total monitored log source count, 30-day failed login trend (A.5.15) | `dashboards/ISO-27001-Controls.md` |
| `ISO_27001_Technological_Controls_Evidence2.png` | **ISO 27001 Technological Controls Evidence** dashboard (bottom half) | Privileged group changes (A.5.18), registry/config change volume (A.8.9), endpoint detection activity count (A.8.7), Splunk storage utilization (A.8.6), and the active detection rule inventory mapped to MITRE ATT&CK technique IDs (A.5.7) | `dashboards/ISO-27001-Controls.md` |
| `Meraki_Network_Monitoring.png` | **Meraki Network Monitoring** dashboard | Total event count, VPN/network event breakdown (pie chart), top visited domains, top talkers by web request volume, Meraki event volume over time, top destination ports, DNS-over-HTTPS usage detection | `spl/Meraki-Searches.md` |

## Guidelines for Adding New Screenshots

1. Redact hostnames, usernames, and any company-identifying domains before committing.
2. Prefer PNG over JPG for UI screenshots — text stays sharp.
3. Crop to the relevant panel/tab rather than uploading full-page captures with excess whitespace.
4. Keep the index table updated — one row per image, pointing to the doc or SPL search it evidences.
5. Name detection-specific screenshots starting with `detection-` if you add ones that show an alert actually firing.
