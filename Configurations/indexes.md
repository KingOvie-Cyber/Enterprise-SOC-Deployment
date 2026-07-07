# indexes.conf — Enterprise SOC Index Configuration

**Location:** `/opt/splunk/etc/system/local/indexes.conf`

## Usage Notes

- Only activate a stanza when you are actively ingesting that data source.
- Do not pre-create empty indexes — Splunk creates index storage on first event write.

### Retention (`frozenTimePeriodInSecs`, in seconds)

| Period   | Seconds   |
|----------|-----------|
| 30 days  | 2592000   |
| 90 days  | 7776000   |
| 180 days | 15552000  |
| 365 days | 31536000  |

`maxTotalDataSizeMB` is the hard size cap in megabytes. Whichever limit is reached first (time or size) triggers data aging to frozen.

## Tier 1 — Currently Active

```ini
[meraki]
homePath        = $SPLUNK_DB/meraki/db
coldPath        = $SPLUNK_DB/meraki/colddb
thawedPath      = $SPLUNK_DB/meraki/thaweddb
maxTotalDataSizeMB     = 50000
frozenTimePeriodInSecs = 7776000
# Source:    Cisco Meraki syslog (firewall, VPN, URL events)
# Retention: 90 days
# Volume:    High — ~1.5 GB/day across 2-3 sites
# Notes:     URL logging generates the majority of volume

[sysmon]
homePath        = $SPLUNK_DB/sysmon/db
coldPath        = $SPLUNK_DB/sysmon/colddb
thawedPath      = $SPLUNK_DB/sysmon/thaweddb
maxTotalDataSizeMB     = 100000
frozenTimePeriodInSecs = 7776000
# Source:    Windows Sysmon endpoint telemetry
# Retention: 90 days
# Volume:    Very high — ~100 MB/day per endpoint
# Notes:     Largest index by volume — monitor size closely

[windows]
homePath        = $SPLUNK_DB/windows/db
coldPath        = $SPLUNK_DB/windows/colddb
thawedPath      = $SPLUNK_DB/windows/thaweddb
maxTotalDataSizeMB     = 50000
frozenTimePeriodInSecs = 31536000
# Source:    Windows Event Logs (Security/System/Application)
# Retention: 365 days — compliance requirement
# Volume:    Medium — ~50 MB/day per endpoint
# Notes:     Longer retention for audit and investigation
#            support across annual cycles
```

## Tier 2 — Activate When Source Is Onboarded

```ini
#[auth]
#homePath        = $SPLUNK_DB/auth/db
#coldPath        = $SPLUNK_DB/auth/colddb
#thawedPath      = $SPLUNK_DB/auth/thaweddb
#maxTotalDataSizeMB     = 20000
#frozenTimePeriodInSecs = 31536000
# Source:    Entra ID / Active Directory sign-in logs
# Retention: 365 days — compliance and investigation
# Volume:    Low — identity events are infrequent
# Status:    Pending Microsoft 365 / Entra ID integration

#[dns]
#homePath        = $SPLUNK_DB/dns/db
#coldPath        = $SPLUNK_DB/dns/colddb
#thawedPath      = $SPLUNK_DB/dns/thaweddb
#maxTotalDataSizeMB     = 75000
#frozenTimePeriodInSecs = 2592000
# Source:    DNS query logs
# Retention: 30 days — high volume, shorter value window
# Volume:    Very high — every DNS lookup across all endpoints
# Status:    Planned — pending DNS logging infrastructure

#[dhcp]
#homePath        = $SPLUNK_DB/dhcp/db
#coldPath        = $SPLUNK_DB/dhcp/colddb
#thawedPath      = $SPLUNK_DB/dhcp/thaweddb
#maxTotalDataSizeMB     = 10000
#frozenTimePeriodInSecs = 2592000
# Source:    DHCP lease logs
# Retention: 30 days — IP-to-device mapping reference
# Volume:    Low — lease events are infrequent
# Status:    Planned — enables identity-aware network monitoring

#[linux]
#homePath        = $SPLUNK_DB/linux/db
#coldPath        = $SPLUNK_DB/linux/colddb
#thawedPath      = $SPLUNK_DB/linux/thaweddb
#maxTotalDataSizeMB     = 30000
#frozenTimePeriodInSecs = 7776000
# Source:    Linux server syslog / auth.log
# Retention: 90 days
# Volume:    Medium — depends on server activity
# Status:    Planned — for Linux servers beyond SOC infra
```

## Tier 3 — Future Expansion

```ini
#[email]
#homePath        = $SPLUNK_DB/email/db
#coldPath        = $SPLUNK_DB/email/colddb
#thawedPath      = $SPLUNK_DB/email/thaweddb
#maxTotalDataSizeMB     = 30000
#frozenTimePeriodInSecs = 15552000
# Source:    Email gateway / Microsoft 365 mail logs
# Retention: 180 days — phishing investigation window
# Volume:    Medium
# Status:    Planned — pending M365 access

#[cloud]
#homePath        = $SPLUNK_DB/cloud/db
#coldPath        = $SPLUNK_DB/cloud/colddb
#thawedPath      = $SPLUNK_DB/cloud/thaweddb
#maxTotalDataSizeMB     = 50000
#frozenTimePeriodInSecs = 31536000
# Source:    AWS CloudTrail / Azure Activity Logs
# Retention: 365 days — cloud audit requirement
# Volume:    Medium
# Status:    Planned — pending cloud integration phase

#[vuln]
#homePath        = $SPLUNK_DB/vuln/db
#coldPath        = $SPLUNK_DB/vuln/colddb
#thawedPath      = $SPLUNK_DB/vuln/thaweddb
#maxTotalDataSizeMB     = 20000
#frozenTimePeriodInSecs = 7776000
# Source:    Vulnerability scanner output (Nessus / OpenVAS)
# Retention: 90 days
# Volume:    Low-Medium — periodic scan results
# Status:    Planned — pending scanner deployment

#[edr]
#homePath        = $SPLUNK_DB/edr/db
#coldPath        = $SPLUNK_DB/edr/colddb
#thawedPath      = $SPLUNK_DB/edr/thaweddb
#maxTotalDataSizeMB     = 50000
#frozenTimePeriodInSecs = 7776000
# Source:    EDR alerts (Wazuh / Defender / CrowdStrike)
# Retention: 90 days
# Volume:    Medium
# Status:    Planned — pending EDR deployment decision
```
