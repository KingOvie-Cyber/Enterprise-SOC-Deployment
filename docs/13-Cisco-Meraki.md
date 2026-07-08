# Cisco Meraki
## Network Security Monitoring — Syslog Integration and Log Analysis

---

## Introduction

This document describes the Cisco Meraki integration with 
the Enterprise SOC. It covers the syslog configuration, 
log types collected, field extraction methodology, and 
the monitoring capability delivered through this integration.

Cisco Meraki provides the network security layer across 
multiple physical sites. By forwarding Meraki logs to 
Splunk, the SOC gains visibility into network traffic, 
firewall events, VPN activity, and web browsing behavior 
across the entire organization's network perimeter.

---

## Why Network Log Integration Matters

Endpoint telemetry (Sysmon) shows what happens on 
individual devices. Network logs show what happens 
between devices and the outside world — and between 
devices on the same network.

Without network logs, the SOC cannot:
- Detect devices communicating with known-bad infrastructure
- Identify data exfiltration attempts
- Monitor web browsing behavior for policy compliance
- Detect VPN anomalies or unauthorized remote access
- Establish a network traffic baseline for anomaly detection

Meraki syslog integration closes these gaps.

---

## Architecture

```
Cisco Meraki Appliance
(Cloud-managed, each physical site)
         |
         | UDP Syslog
         | Port 514
         |
         v
Splunk Enterprise
(Rocky Linux host, Docker container)
Port 514 exposed via Docker port mapping
         |
         v
meraki index
sourcetype: meraki_syslog
```

---

## Meraki Dashboard Configuration

### Syslog Server Configuration

```
Location in Meraki Dashboard:
Network-wide → General → Reporting → Syslog servers
```

### Settings Applied

| Setting | Value |
|---|---|
| Server IP | SOC server IP address |
| Port | 514 |
| Protocol | UDP |
| Encrypted TLS | Disabled (plain UDP) |

### Roles Enabled

| Role | Description | Enabled |
|---|---|---|
| Appliance Event log | Firewall events, VPN events, security events | Yes |
| Appliance URLs | All web requests made by network clients | Yes |
| Appliance Flows | Full traffic flow data (all connections) | No |

### Why Appliance Flows is Disabled

Appliance Flows logs every single network connection 
across all clients — not just security-relevant events 
and web requests. At the organization's current network 
scale (16-18 active endpoints across 2-3 sites), 
enabling Flows would generate significantly higher 
log volume than Event log and URLs combined, increasing 
storage requirements and approaching Splunk license 
limits faster.

Flows will be reconsidered if specific investigation 
requirements arise that cannot be met with Event log 
and URL data alone.

---

## Log Types and Format

### Appliance Event Log Format

Meraki event log entries follow a syslog format with 
structured key-value pairs:

```
Jun 29 09:32:08 <device-ip> 1 <timestamp> <network-id> 
<event-type> <key=value pairs>
```

**Event types confirmed in this environment:**

| Type | Description | Count (24h sample) |
|---|---|---|
| `fips_event` | FIPS cryptographic compliance events | 23 |
| `vpn_connectivity_change` | VPN tunnel state changes | 8 |
| `route_connection_change` | Network routing path changes | 6 |
| `vpn_registry_change` | VPN configuration changes | 2 |

### Appliance URLs Format

URL log entries capture web requests from network clients:

```
Jun 29 09:32:08 <device-ip> 1 <timestamp> <network-id> 
urls src=<src-ip>:<src-port> dst=<dst-ip>:<dst-port> 
mac=<mac-address> request: <method> <url>
```

**Note on field extraction:** Meraki URL logs use 
space-delimited format rather than consistent key=value 
pairs throughout. Standard Splunk field extraction does 
not automatically parse all fields. Custom regex 
extraction is required — see Field Extraction section below.

---

## Splunk Configuration

### props.conf

```ini
[meraki_syslog]
SHOULD_LINEMERGE = false
TIME_PREFIX = ^
TIME_FORMAT = %b %d %H:%M:%S
MAX_TIMESTAMP_LOOKAHEAD = 32

[source::udp:514]
TRANSFORMS-routing = route_meraki
sourcetype = meraki_syslog
```

### transforms.conf

```ini
[route_meraki]
REGEX = .
DEST_KEY = _MetaData:Index
FORMAT = meraki
```

---

## Field Extraction

Since Meraki URL logs require regex-based extraction, 
the following patterns are used in SPL searches.

### Full URL Log Extraction

```spl
index=meraki sourcetype=meraki_syslog "urls"
| rex field=_raw "src=(?<src_ip>[^:]+):(?<src_port>\d+)\s+dst=(?<dst_ip>[^:]+):(?<dst_port>\d+)\s+mac=(?<mac>\S+)\s+request:\s+\S+\s+(?<url>https?://\S+)"
| table _time, src_ip, src_port, dst_ip, dst_port, mac, url
```

### Domain-Only Extraction

```spl
index=meraki sourcetype=meraki_syslog "urls"
| rex field=_raw "request:\s+\S+\s+(?<url>https?://[^\s/]+)"
| top limit=10 url
```

### Source IP Extraction

```spl
index=meraki sourcetype=meraki_syslog "urls"
| rex field=_raw "src=(?<src_ip>[^:]+):"
| stats count by src_ip
| sort -count
```

### Destination Port Extraction

```spl
index=meraki sourcetype=meraki_syslog "urls"
| rex field=_raw "dst=[^:]+:(?<dst_port>\d+)"
| top limit=10 dst_port
```

---

## Monitoring Use Cases

### 1. Top Visited Domains

Establishes the baseline of normal web destinations. 
Repeated review identifies whether new, unexpected 
domains are appearing.

```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=-24h latest=now
| rex field=_raw "request:\s+\S+\s+(?<url>https?://[^\s/]+)"
| top limit=10 url
```

**Baseline established in this environment:**
Normal top domains include Microsoft telemetry endpoints, 
Windows Update, Office 365, Azure monitoring, and 
Google services — all expected for a Windows-managed 
organization using Microsoft 365.

### 2. Top Talkers (Highest Traffic Devices)

Identifies which devices generate the most web requests. 
Devices significantly above baseline warrant investigation.

```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=-24h latest=now
| rex field=_raw "src=(?<src_ip>[^:]+):"
| top limit=10 src_ip
```

### 3. Network Event Type Breakdown

Provides visibility into security events (firewall 
blocks, VPN changes, routing changes).

```spl
index=meraki sourcetype=meraki_syslog
earliest=-24h latest=now
| stats count by type
| sort -count
```

### 4. Traffic Volume Over Time

Establishes normal traffic patterns. Significant 
deviations from baseline (especially late-night 
spikes) warrant investigation.

```spl
index=meraki sourcetype=meraki_syslog
earliest=-24h latest=now
| timechart span=1h count
```

### 5. DNS-over-HTTPS Detection

Identifies devices bypassing standard DNS visibility 
by using encrypted DNS resolvers. This creates a 
monitoring blind spot where DNS-based threat detection 
cannot see what these devices are resolving.

```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=-24h latest=now
("dns.google" OR "cloudflare-dns.com" OR "doh.opendns.com")
| rex field=_raw "src=(?<src_ip>[^:]+):"
| stats count by src_ip
| sort -count
```

**Finding in this environment:** 39 distinct devices 
were found using DNS-over-HTTPS, generating 1,441 
DoH requests in a 24-hour window. This is likely 
default browser behavior (Firefox and Chrome enable 
DoH by default in some regions) rather than 
deliberate policy evasion — but it represents a 
real DNS visibility gap worth monitoring and 
addressing through network policy.

### 6. Geographic Destination Analysis

Identifies connections to unexpected countries. 
Connections to high-risk or unusual geographic 
regions for this organization warrant investigation.

```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=-24h latest=now
| rex field=_raw "dst=(?<dst_ip>[^:]+):"
| iplocation dst_ip
| top limit=10 Country
```

### 7. Destination Port Analysis

Identifies non-standard ports in use. The vast 
majority of legitimate traffic uses ports 443 and 80.

```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=-24h latest=now
| rex field=_raw "dst=[^:]+:(?<dst_port>\d+)"
| top limit=10 dst_port
```

**Finding in this environment:** Port 853 
(DNS-over-TLS) was identified alongside the DoH 
traffic — representing a second encrypted DNS 
mechanism in use on the network.

---

## Known Gaps and Planned Improvements

### Firewall Rule Logging

Currently, firewall allow/deny events do not appear 
in the Meraki log stream. This is because Meraki 
logs firewall hits per-rule, and logging must be 
explicitly enabled on individual firewall rules 
within the Meraki dashboard.

**Current status:** Gap identified. Per-rule logging 
to be enabled on key security rules.

**Impact:** Without firewall rule logging, the SOC 
cannot see what traffic is being blocked at the 
network perimeter — one of the most valuable 
visibility points in any network security program.

### DHCP Lease Identity Mapping

Meraki DHCP logs are present but the field extraction 
regex requires refinement to correctly parse the 
actual log format. Currently, source IPs in URL 
and event logs cannot be automatically correlated 
to device identities.

**Planned:** Build a DHCP lease lookup table in 
Splunk to map IP addresses to device hostnames 
and MAC addresses, enabling identity-aware 
network monitoring.

### Appliance Flows

Full connection flow data is not currently collected. 
This limits the SOC's ability to investigate 
non-web traffic and perform complete connection 
timeline analysis.

**Planned:** Enable Flows logging once storage 
and license capacity supports the additional volume.

---

## Lessons Learned

**Field name mismatch:** Initial attempts to 
group Meraki events by `signature` field (a common 
field name used in other SIEM/firewall integrations) 
returned no results. Investigation of raw events 
revealed Meraki uses `type` as the event category 
field. Always inspect raw events before building 
search logic based on assumed field names.

**URL logs need custom regex:** The assumption 
that Splunk's automatic field extraction would 
parse all Meraki URL log fields correctly was 
wrong. The space-delimited format of URL log 
entries required custom regex extraction for 
each field. This is now documented in the 
Field Extraction section above.

**DoH is a significant blind spot:** The 
discovery that 39 devices were using 
DNS-over-HTTPS — generating over 1,400 
DoH requests in 24 hours — was a direct 
outcome of building the monitoring capability. 
Without Meraki URL log integration, this 
blind spot would have been invisible. 
This finding demonstrates the value of 
network log monitoring beyond just 
firewall event analysis.

**Volume matters:** Meraki URL logs generate 
significant volume (80,000+ events per 24 hours 
in this environment). Index sizing, retention 
policy, and Splunk license planning must 
account for this source specifically.

---

> **Confidentiality Note:** Server IP addresses, 
> network topology details, device identifiers, 
> and organization-specific findings have been 
> generalized or sanitized. Actual infrastructure 
> details are not documented in this repository.
