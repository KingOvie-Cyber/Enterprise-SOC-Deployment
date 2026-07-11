# transforms.conf — Event Routing Configuration

**Location:** `/opt/splunk/etc/system/local/transforms.conf`

## Purpose

Defines routing rules that direct incoming events to the correct Splunk index based on their source. Works in conjunction with `props.conf`.

## Configuration

```ini
[route_meraki]
REGEX = .
# Match any non-empty event (routes all Meraki syslog events)

DEST_KEY = _MetaData:Index
# Set the index metadata field

FORMAT = meraki
# Route to the 'meraki' index
```

## How This Works With props.conf

1. Event arrives on UDP port 514
2. `props.conf` `[source::udp:514]` assigns:
   - `sourcetype = meraki_syslog`
   - `TRANSFORMS-routing = route_meraki`
3. `transforms.conf` `[route_meraki]` routes the event to:
   - `index = meraki`

**Result:** All UDP 514 events land in the `meraki` index with sourcetype `meraki_syslog`.

## Endpoint Routing Notes

Windows endpoint events (Sysmon + Windows Event Logs) are routed to their indexes via `inputs.conf` on the Universal Forwarder, not via `transforms.conf`:

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = sysmon

[WinEventLog://Security]
index = windows
```

`transforms.conf` routing is used for syslog sources where the index cannot be set at the source.
