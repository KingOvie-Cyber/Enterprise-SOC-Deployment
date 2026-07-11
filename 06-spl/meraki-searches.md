# Meraki Network Monitoring SPL Searches

SPL queries built against Cisco Meraki event logs (MX security appliance / MR access points / MS switches) forwarded into Splunk via syslog. These support the "Meraki Network Monitoring" dashboard in the SOC build.

---

## IDS/IPS Alerts (MX Security Events)

Surface signature-based intrusion alerts from the Meraki MX.

```spl
index=network sourcetype="meraki:ids-alerts"
| table _time, srcIp, destIp, destPort, protocol, message, priority
| sort -_time
```

**Why it matters:** First place to look for known-exploit or scanning activity hitting the perimeter.

---

## Firewall Deny Events

Track blocked connections to spot recon/scanning attempts or misconfigured internal traffic.

```spl
index=network sourcetype="meraki:firewall" action="deny"
| stats count by src_ip, dest_ip, dest_port, protocol
| sort -count
| head 50
```

**Why it matters:** A spike in denies from a single internal host is often a sign of infection (worm-like scanning) or a misbehaving device.

---

## VPN Client Connect/Disconnect

Monitor client VPN session activity for anomalies (odd hours, unusual geo, repeated failures).

```spl
index=network sourcetype="meraki:vpn" (event="connect" OR event="disconnect")
| eval hour=strftime(_time, "%H")
| where hour < 6 OR hour > 22
| table _time, user, src_ip, event, remote_ip
| sort -_time
```

**Why it matters:** Off-hours VPN logins are a common indicator of compromised credentials being used outside normal working patterns.

---

## Rogue/Unauthorized AP Detection

Flag wireless access points detected by Meraki MR that aren't part of the managed network.

```spl
index=network sourcetype="meraki:air-marshal" 
| search containment="unauthorized" OR classification="rogue"
| table _time, ssid, bssid, channel, rssi
| sort -_time
```

**Why it matters:** Rogue APs can be used for evil-twin attacks or unauthorized network extension — a physical security concern that log-only monitoring often misses.

---

## Client Bandwidth Anomalies

Identify hosts suddenly consuming abnormal amounts of bandwidth (possible exfiltration or C2 beacon with large payloads).

```spl
index=network sourcetype="meraki:netflow"
| stats sum(bytes) as total_bytes by src_ip, dest_ip
| where total_bytes > 500000000
| sort -total_bytes
```

**Why it matters:** Large, sustained data transfers to unfamiliar external IPs is a top exfiltration indicator.

---

## Switch Port Flapping / Unexpected Device Changes

Detect ports going up/down repeatedly, which can indicate a rogue device being plugged/unplugged or a failing NIC.

```spl
index=network sourcetype="meraki:switch-events" (event="port-up" OR event="port-down")
| stats count by switch_name, port, event
| where count > 10
| sort -count
```

**Why it matters:** Repeated port state changes can indicate unauthorized physical access attempts (e.g., someone plugging in a rogue device repeatedly) or hardware failure worth investigating either way.

---

## Notes

- Sourcetypes assume Meraki syslog export configured via the dashboard (Network-wide > General > Reporting) pointed at the Rocky Linux syslog collector, which forwards to Splunk.
- Field names (`src_ip`, `dest_ip`, etc.) may need adjusting depending on whether you're using raw Meraki syslog format or a normalized CIM-compliant TA.
- These pair well with the ISO 27001 controls dashboard for evidencing network monitoring controls (A.8.16 / A.13.1).
