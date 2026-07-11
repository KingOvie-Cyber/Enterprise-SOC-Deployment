# Sysmon Event Code SPL Searches

Reference set of Splunk searches built against Sysmon telemetry ingested through the syslog collector into the SOC's Splunk instance. Field names assume the Sysmon TA (Technology Add-on) is applied for CIM compliance; adjust `index`/`sourcetype` to match your environment.

---

## Event Code 1 — Process Creation

Detect suspicious parent/child process relationships (e.g., Office spawning a shell).

```spl
index=endpoint sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
| eval parent=lower(ParentImage), child=lower(Image)
| search parent="*winword.exe" OR parent="*excel.exe" OR parent="*outlook.exe"
| search child="*cmd.exe" OR child="*powershell.exe" OR child="*wscript.exe" OR child="*mshta.exe"
| table _time, host, User, ParentImage, Image, CommandLine
| sort -_time
```

**Why it matters:** Office applications spawning command interpreters is a classic macro-malware / initial-access signature.

---

## Event Code 3 — Network Connection

Surface processes making outbound connections that aren't typically network-facing.

```spl
index=endpoint sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=3
| eval proc=lower(Image)
| search proc="*powershell.exe" OR proc="*rundll32.exe" OR proc="*regsvr32.exe" OR proc="*mshta.exe"
| stats count values(DestinationIp) as dest_ips values(DestinationPort) as dest_ports by host, Image, User
| where count > 0
| sort -count
```

**Why it matters:** LOLBins (living-off-the-land binaries) reaching out to the network is a strong C2/beaconing indicator.

---

## Event Code 7 — Image/DLL Loaded

Hunt for DLL side-loading or loads from non-standard paths.

```spl
index=endpoint sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=7
| where NOT match(ImageLoaded, "(?i)^[a-z]:\\\\(windows|program files|program files \(x86\))\\\\")
| stats count by host, Image, ImageLoaded, Signed, Signature
| where Signed="false"
| sort -count
```

**Why it matters:** Unsigned DLLs loading from user-writable paths (Temp, AppData, Downloads) is a common persistence/evasion technique.

---

## Event Code 8 — CreateRemoteThread

Classic process injection detection.

```spl
index=endpoint sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=8
| table _time, host, SourceImage, TargetImage, User, StartFunction
| sort -_time
```

**Why it matters:** Remote thread creation into another process (e.g., lsass.exe, explorer.exe) is a hallmark of injection-based malware and credential dumping tools.

---

## Event Code 11 — File Create

Detect files dropped into commonly abused execution paths.

```spl
index=endpoint sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=11
| search TargetFilename="*\\AppData\\Roaming\\*" OR TargetFilename="*\\Temp\\*" OR TargetFilename="*\\Startup\\*"
| regex TargetFilename="\.(exe|dll|ps1|bat|vbs|scr)$"
| table _time, host, Image, TargetFilename, User
| sort -_time
```

**Why it matters:** Malware droppers and staged payloads frequently land in Temp, AppData, or Startup folders.

---

## Event Code 13 — Registry Value Set

Catch persistence via Run keys.

```spl
index=endpoint sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=13
| search TargetObject="*\\CurrentVersion\\Run*" OR TargetObject="*\\CurrentVersion\\RunOnce*"
| table _time, host, Image, TargetObject, Details, User
| sort -_time
```

**Why it matters:** Registry Run/RunOnce keys are one of the most common Windows persistence mechanisms (MITRE T1547.001).

---

## Event Code 22 — DNS Query

Flag queries to newly-observed or suspicious-looking domains.

```spl
index=endpoint sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=22
| eval domain_len=len(QueryName)
| where domain_len > 40 OR match(QueryName, "^[a-z0-9]{16,}\.")
| stats count by host, Image, QueryName
| sort -count
```

**Why it matters:** Long, high-entropy subdomains are a common indicator of DNS tunneling or DGA (domain generation algorithm) malware.

---

## Notes

- All searches were tuned against real telemetry from the production SOC's Rocky Linux + Sysmon endpoints feeding into Splunk via the syslog collector.
- Recommend saving these as scheduled searches with alerting thresholds once baseline noise is characterized in your environment (expect false positives from legitimate admin tooling on first run).
- Field extraction depends on Splunk's Sysmon TA — install it (`Sysmon TA for Splunk` / `Microsoft-Windows-Sysmon-Operational`) before deploying these as-is.
