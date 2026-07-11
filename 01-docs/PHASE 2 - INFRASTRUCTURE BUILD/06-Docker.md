# Docker
## Splunk Enterprise Containerization — Deployment and Management

---

## Introduction

This document describes the Docker-based deployment of Splunk 
Enterprise on the SOC server. It covers the rationale for 
containerization, the Docker Compose configuration, persistent 
storage design, port mapping decisions, and operational 
management procedures.

---

## Why Docker for Splunk

The decision to containerize Splunk rather than install it 
natively on Rocky Linux was deliberate. The key reasons are 
documented in [03-Design-Decisions.md](03-Design-Decisions.md), 
but are summarized here:

**Application/OS separation:** Splunk configuration and data 
live independently of the operating system. OS-level changes 
(patches, updates, reconfigurations) do not affect Splunk's 
operational state.

**Clean upgrade path:** Upgrading Splunk is a matter of 
pulling a new image and recreating the container. No RPM 
conflicts, no manual file migrations.

**Data persistence:** Named Docker volumes ensure all indexed 
data and Splunk configuration survive container restarts, 
recreations, and upgrades.

**Consistent deployment:** The `docker-compose.yml` file 
fully describes the Splunk deployment. It can be version-controlled, 
reviewed, and reproduced exactly.

---

## Prerequisites

Before deploying Splunk via Docker, ensure:

- Docker Engine is installed and running
- Docker Compose plugin is installed
- The deploying user is in the `docker` group (or using sudo)
- Required firewall ports are open (8000, 9997, 514)

Verify:
```bash
docker --version
docker compose version
sudo firewall-cmd --list-ports
```

---

## Docker Compose Configuration

### File Location
```
~/soc-deployment/splunk/docker-compose.yml
```

### Full Configuration

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
      - "8000:8000"
      - "9997:9997"
      - "514:514/udp"
    volumes:
      - splunk-var:/opt/splunk/var
      - splunk-etc:/opt/splunk/etc
    restart: unless-stopped

volumes:
  splunk-var:
  splunk-etc:
```

---

## Configuration Decisions Explained

### Image Selection
```yaml
image: splunk/splunk:latest
```
The official Splunk Docker image from Docker Hub is used. 
In a production environment with stricter change control, 
pinning to a specific version tag (e.g. `splunk/splunk:9.3.0`) 
would be preferred to prevent unexpected upgrades during 
container recreation.

### Port Mappings
```yaml
ports:
  - "8000:8000"    # Splunk Web UI
  - "9997:9997"    # Universal Forwarder data input
  - "514:514/udp"  # Syslog input (Cisco Meraki)
```

Only ports actively required for SOC operation are exposed. 
Notable omissions:

| Port | Purpose | Exposed? | Reason |
|---|---|---|---|
| 8000 | Splunk Web UI | Yes | Analyst access required |
| 9997 | Forwarder input | Yes | Endpoint log shipping required |
| 514 | Syslog (UDP) | Yes | Meraki log receive required |
| 8089 | Management API | No | Not required externally — reduces attack surface |
| 8088 | HEC (HTTP Event Collector) | No | Not used in this deployment |

### Persistent Volumes
```yaml
volumes:
  - splunk-var:/opt/splunk/var
  - splunk-etc:/opt/splunk/etc
```

Two named volumes are critical:

**splunk-var:** Contains all indexed event data, search 
artifacts, and internal Splunk state. Without this volume, 
all indexed data is lost every time the container is recreated.

**splunk-etc:** Contains all Splunk configuration files — 
indexes.conf, props.conf, transforms.conf, saved searches, 
dashboards, and user settings. Without this volume, all 
configuration is lost on container recreation.

This was learned through experience early in the deployment 
when data was lost before volumes were correctly configured.

### Restart Policy
```yaml
restart: unless-stopped
```

The container automatically restarts after server reboots 
and after most failure conditions — without requiring manual 
intervention. `unless-stopped` (rather than `always`) means 
if the container is deliberately stopped (e.g. for maintenance), 
it stays stopped after a reboot until manually started again.

---

## Deployment Commands

### Start Splunk
```bash
cd ~/soc-deployment/splunk
docker compose up -d
```

### Monitor Initialization (First Start)
```bash
docker logs -f splunk
```
Wait for the initialization sequence to complete before 
accessing the Web UI. First start typically takes 2-4 minutes.

### Check Container Status
```bash
docker ps
```

### Restart Splunk (Recommended Method)
```bash
docker compose restart splunk
```

**Important:** Always restart Splunk using this command 
from the host, not via Splunk Web's "Server Controls" menu. 
The official Splunk Docker image uses an Ansible-based 
entrypoint that can cause the container to exit rather 
than restart when the restart is triggered internally 
via the Web UI.

### Stop Splunk
```bash
docker compose down
```

### View Splunk Logs
```bash
docker logs splunk
docker logs -f splunk    # Follow/live view
```

### Access Splunk CLI Inside Container
```bash
docker exec -it splunk bash
```

### Run Splunk CLI Commands
```bash
docker exec splunk /opt/splunk/bin/splunk version
docker exec splunk /opt/splunk/bin/splunk list index \
  -auth admin:<password>
```

---

## Volume Management

### List Docker Volumes
```bash
docker volume ls
```

### Inspect a Volume
```bash
docker volume inspect splunk-var
docker volume inspect splunk-etc
```

### Backup Splunk Configuration Volume
```bash
docker run --rm \
  -v splunk-etc:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/splunk-etc-backup.tar.gz /data
```

### Backup Splunk Data Volume
```bash
docker run --rm \
  -v splunk-var:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/splunk-var-backup.tar.gz /data
```

---

## Editing Splunk Configuration Files

Configuration files live inside the `splunk-etc` volume, 
accessible at `/opt/splunk/etc/` inside the container.

### Enter the Container
```bash
docker exec -it splunk bash
```

### Navigate to Configuration Directory
```bash
cd /opt/splunk/etc/system/local/
```

### Edit Configuration Files
```bash
vi indexes.conf
vi props.conf
vi transforms.conf
```

### Apply Configuration Changes
Exit the container and restart:
```bash
exit
docker compose restart splunk
```

Configuration changes require a Splunk restart to take effect.

---

## Upgrading Splunk

To upgrade to a newer Splunk version:

```bash
cd ~/soc-deployment/splunk

# Pull the latest image
docker compose pull

# Recreate the container with the new image
docker compose up -d --force-recreate
```

Because data and configuration live in persistent volumes 
(not inside the container), the upgrade process does not 
affect existing indexed data or configuration.

---

## Troubleshooting

### Container Not Starting
```bash
docker logs splunk
docker inspect splunk
```

Check for port conflicts:
```bash
sudo ss -tulnp | grep -E "8000|9997|514"
```

### Data Not Appearing in Splunk
Verify the container's port mappings are correct:
```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

Confirm Splunk receiving is configured:
```
Splunk Web → Settings → Forwarding and receiving 
→ Configure receiving → Port 9997 should be listed
```

### Container Exits Unexpectedly
```bash
docker logs splunk --tail 50
```

Check for disk space issues (full disk will cause Splunk 
to stop indexing and may cause the container to exit):
```bash
df -h
docker system df
```

---

## Security Considerations

**Password management:** The `SPLUNK_PASSWORD` environment 
variable in the compose file should not be stored in plain 
text in a production environment with strict secrets 
management requirements. Docker Secrets or an external 
secrets manager (Vault, AWS Secrets Manager) would be 
appropriate for higher-security deployments.

**Image verification:** In environments with strict supply 
chain requirements, the Splunk Docker image digest should 
be pinned and verified before deployment.

**Container user:** The official Splunk Docker image runs 
Splunk under a non-root internal user, reducing the impact 
of any application-level vulnerability within the container.

---

## Lessons Learned

**Data loss without volumes:** Early in the deployment, 
the `splunk-var` volume was not configured. When the 
container was recreated during troubleshooting, all 
indexed data was lost. Persistent volumes are not 
optional — they are mandatory for any production 
Splunk Docker deployment.

**Web UI restart issue:** Triggering a restart from 
Splunk Web's Server Controls caused the container to 
exit rather than restart cleanly. The root cause is 
the Ansible-based entrypoint in the official Splunk 
Docker image, which does not correctly re-supervise 
the Splunk process after an internally-triggered restart. 
Workaround: always restart via `docker compose restart splunk`.

**Port mapping verification:** During initial deployment, 
a port mapping error (8088 vs 8089) caused log forwarding 
to fail silently. Always verify port mappings with 
`docker ps` after container creation.

---

> **Confidentiality Note:** Passwords, tokens, and 
> organization-specific configuration values are 
> redacted throughout this document. Placeholders 
> such as `<REDACTED>` indicate sanitized values.
