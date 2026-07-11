# Meraki Network Monitoring Dashboard
## Dashboard Studio Build Guide

---

## Purpose

Network traffic visibility across all monitored sites 
via Cisco Meraki syslog data. Covers URL/web activity, 
event types, top talkers, and encrypted DNS detection.

---

## Dashboard Settings
```
Title:       Meraki Network Monitoring
Framework:   Dashboard Studio (Grid layout)
Time Token:  global_time (Last 24 hours default)
```

---

## Panel Layout

```
Row 1: [Total Meraki Events] [Security Event Types - Pie]
Row 2: [Event Volume Over Time — Line Chart — full width]
Row 3: [Top Visited Domains] [Top Talkers]
Row 4: [Top Destination Ports] [DNS-over-HTTPS Usage]
```

---

## Panel SPL Queries

### Total Meraki Events
```spl
index=meraki sourcetype=meraki_syslog
earliest=$global_time.earliest$ latest=$global_time.latest$
| stats count
```

### Security Event Types
```spl
index=meraki sourcetype=meraki_syslog
earliest=$global_time.earliest$ latest=$global_time.latest$
| stats count by type
| sort -count
```

### Event Volume Over Time
```spl
index=meraki sourcetype=meraki_syslog
earliest=$global_time.earliest$ latest=$global_time.latest$
| timechart span=1h count
```

### Top Visited Domains
```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=$global_time.earliest$ latest=$global_time.latest$
| rex field=_raw "request:\s+\S+\s+(?<url>https?://[^\s/]+)"
| top limit=10 url
```

### Top Talkers
```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=$global_time.earliest$ latest=$global_time.latest$
| rex field=_raw "src=(?<src_ip>[^:]+):"
| top limit=10 src_ip
```

### Top Destination Ports
```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=$global_time.earliest$ latest=$global_time.latest$
| rex field=_raw "dst=[^:]+:(?<dst_port>\d+)"
| top limit=10 dst_port
```

### DNS-over-HTTPS Usage
```spl
index=meraki sourcetype=meraki_syslog "urls"
earliest=$global_time.earliest$ latest=$global_time.latest$
("dns.google" OR "cloudflare-dns.com" OR "doh.opendns.com")
| rex field=_raw "src=(?<src_ip>[^:]+):"
| stats count by src_ip
| sort -count
```

---

## Known Gaps (Documented)

```
⬜ Firewall allow/deny — requires per-rule logging in Meraki
⬜ DHCP identity mapping — regex refinement needed
⬜ Client connect/disconnect — not yet configured
```

---

> See [Cisco-Meraki.md](../docs/10-Cisco-Meraki.md) for 
> full integration and field extraction documentation.
