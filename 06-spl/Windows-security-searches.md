# Windows Security Event SPL Searches

SPL queries against native Windows Security event log telemetry (Event Codes 4624/4625/4672/4688/4720 etc.), collected from domain-joined hosts and the identity infrastructure, forwarded through the syslog pipeline into Splunk.

---

## Event Code 4625 — Failed Logon (Brute Force Detection)

```spl
index=windows sourcetype="WinEventLog:Security" EventCode=4625
| stats count by Account_Name, src_ip
| where count > 5
| sort -count
```

**Why it matters:** Repeated failed logons from a single source against one or more accounts is the classic brute-force / password-spray signature.

---

## Event Code 4624 — Successful Logon After Multiple Failures

Correlate a successful logon that follows a burst of failures — a strong sign a brute-force attempt succeeded.

```spl
index=windows sourcetype="WinEventLog:Security" EventCode=4625
| stats count as failures by Account_Name, src_ip
| where failures > 5
| join Account_Name
    [ search index=windows sourcetype="WinEventLog:Security" EventCode=4624
      | table _time, Account_Name, src_ip, Logon_Type ]
| table _time, Account_Name, src_ip, Logon_Type, failures
| sort -_time
```

**Why it matters:** Failures followed immediately by success against the same account is a high-fidelity compromise indicator, far stronger than either signal alone.

---

## Event Code 4672 — Special Privileges Assigned to New Logon

Track when accounts are granted admin-equivalent privileges at logon.

```spl
index=windows sourcetype="WinEventLog:Security" EventCode=4672
| table _time, Account_Name, host, Privilege_List
| sort -_time
```

**Why it matters:** Unexpected privileged logons (especially for service accounts or non-admin users) can indicate privilege escalation or token abuse.

---

## Event Code 4688 — New Process Created

Same intent as Sysmon EventCode 1, useful when Sysmon isn't deployed on a given host and native auditing is the only source.

```spl
index=windows sourcetype="WinEventLog:Security" EventCode=4688
| eval parent=lower(Creator_Process_Name), child=lower(New_Process_Name)
| search child="*powershell.exe" OR child="*cmd.exe" OR child="*psexec.exe"
| table _time, host, Account_Name, Creator_Process_Name, New_Process_Name, Process_Command_Line
| sort -_time
```

**Why it matters:** Native process auditing gives coverage even on hosts where Sysmon isn't yet deployed — important during phased rollouts.

---

## Event Code 4720 / 4728 / 4732 — Account & Group Changes

Monitor account creation and additions to privileged groups (Domain Admins, local Administrators).

```spl
index=windows sourcetype="WinEventLog:Security" (EventCode=4720 OR EventCode=4728 OR EventCode=4732)
| eval action=case(EventCode=4720, "Account Created", EventCode=4728, "Added to Global Group", EventCode=4732, "Added to Local Group")
| table _time, action, Account_Name, Group_Name, host
| sort -_time
```

**Why it matters:** New accounts or unexpected additions to privileged groups are common post-compromise persistence steps — this is one of the highest-value alerts for a small SOC to tune well.

---

## Event Code 4768/4769 — Kerberos Ticket Requests (Golden/Silver Ticket Hunting)

Baseline TGT/TGS request patterns; useful on the RODC/domain controller logs specifically.

```spl
index=windows sourcetype="WinEventLog:Security" (EventCode=4768 OR EventCode=4769)
| stats count by Account_Name, Service_Name, host
| sort -count
```

**Why it matters:** Anomalous ticket request volume or unusual service ticket requests can indicate Kerberoasting or ticket-forging attacks against the domain.

---

## Notes

- These searches assume the Splunk Add-on for Microsoft Windows (or equivalent CIM-mapped TA) is applied for consistent field naming (`Account_Name`, `src_ip`, etc.). Adjust field names to match raw XML if the TA isn't installed.
- Recommend cross-referencing the 4625/4624 correlation search with your identity infrastructure logs for full-picture identity monitoring.
- Tune count thresholds (`> 5`, etc.) against your own environment's baseline before enabling as real-time alerts — thresholds here are starting points, not fixed rules.
