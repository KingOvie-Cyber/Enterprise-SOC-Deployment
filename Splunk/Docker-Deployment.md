# Splunk Enterprise — Docker Deployment

## Why Docker?

Splunk was deployed as a Docker container to:

- Separate application lifecycle from OS
- Enable clean upgrades without OS-level changes
- Use volume mounts for persistent index data
- Simplify backup and recovery

## Docker Compose Configuration

```yaml
version: "3.8"
services:
  splunk:
    image: splunk/splunk:latest
    container_name: splunk
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_PASSWORD=<REDACTED>
    ports:
      - "8000:8000"    # Web UI
      - "9997:9997"    # Universal Forwarder input
      - "514:514/udp"  # Syslog input
    volumes:
      - splunk-var:/opt/splunk/var
      - splunk-etc:/opt/splunk/etc
    restart: unless-stopped

volumes:
  splunk-var:
  splunk-etc:
```

## Key Configuration Decisions

**Persistent volumes** — splunk-var and splunk-etc are mounted
as named Docker volumes. This means index data and configuration
survive container restarts and upgrades.

**Port mapping** — Only required ports are exposed.
Splunk management port (8089) is not exposed externally — 
deliberate security decision to reduce attack surface.

**restart: unless-stopped** — Container restarts automatically
after server reboot without manual intervention.

## Indexes Created

| Index Name | Data Source           | Retention |
| ---------- | --------------------- | --------- |
| meraki     | Cisco Meraki syslog   | 90 days   |
| sysmon     | Windows Sysmon events | 90 days   |
| windows    | Windows Event Logs    | 90 days   |
| main       | Default/misc          | 30 days   |

## Issue Encountered & Resolved

**Problem:** Index data was lost when container was recreated
**Cause:** Data was being written inside the container filesystem which is ephemeral
**Solution:** Added named volume mounts for /opt/splunk/var and /opt/splunk/etc
ensuring all index data and configuration persist independently of container lifecycle

## Useful Docker Commands

```bash
# Start Splunk
docker compose up -d

# Check container status
docker ps

# View Splunk logs
docker logs splunk

# Follow Splunk logs live
docker logs -f splunk

# Restart Splunk
docker compose restart splunk

# Stop Splunk
docker compose down

# Access Splunk CLI inside container
docker exec -it splunk bash

# Check Splunk version inside container
docker exec splunk /opt/splunk/bin/splunk version

# Backup Splunk config volume
docker run --rm -v splunk-etc:/data -v $(pwd):/backup \
  alpine tar czf /backup/splunk-etc-backup.tar.gz /data
```
