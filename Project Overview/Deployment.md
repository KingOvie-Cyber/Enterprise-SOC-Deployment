# Deployment
## Step-by-Step SOC Deployment Process

---

## Introduction

This document describes the complete deployment process for the 
Enterprise SOC environment. It covers every phase from initial 
infrastructure provisioning through to operational verification.

The deployment follows a deliberate sequence — each phase 
is verified before the next begins. This prevents the common 
mistake of configuring everything at once and debugging blind 
when something doesn't work.

---

## Deployment Philosophy

> Verify before you proceed. Every phase ends with a 
> verification step. Do not move forward until that 
> verification passes.

This principle was followed throughout the deployment and is 
reflected in the structure of this document. When issues were 
encountered, they were resolved at the layer where they 
originated — not worked around at a higher layer.

---

## Phase 1 — Infrastructure Provisioning

### 1.1 VMware ESXi Installation

VMware ESXi was installed on bare metal hardware (Lenovo 
ThinkStation). Post-installation configuration included:

- Network configuration (management interface, IP assignment)
- Datastore provisioning for VM storage
- vSwitch configuration and port group creation
- SSH enabled for management access (disabled after initial config)

### 1.2 Virtual Machine Provisioning

The primary SOC VM was provisioned with the following 
specifications (adjust to match available hardware):

```
Name:     soc-core
CPU:      4 vCPU (minimum)
RAM:      20 GB 
Disk:     300 GB 
Network:  SOC management port group
```

### 1.3 Rocky Linux Installation

Rocky Linux 9.x was installed on the SOC VM with the 
following configuration choices:

- Installation type: Server with GUI
- Hostname: set during installation (use organizational 
  naming convention)
- Network: static IP configured during installation
- Storage: default partitioning (customizable based on 
  disk capacity)
- User: non-root admin account created with sudo access

**Verification — Phase 1:**
```bash
hostnamectl
ip a
ping -c 4 8.8.8.8
```
All three should return expected values before proceeding.

---

## Phase 2 — Operating System Hardening

Hardening was applied BEFORE connecting to the production 
network or receiving any log data. Security configuration 
is a prerequisite, not an afterthought.

Full hardening procedure: [Hardening.md](Hardening.md)

### 2.1 System Update

```bash
sudo dnf update -y
sudo dnf install -y vim curl wget net-tools firewalld \
  policycoreutils-python-utils audit
```

### 2.2 Firewall Configuration

```bash
sudo systemctl enable --now firewalld

# Open only required ports
sudo firewall-cmd --permanent --add-port=22/tcp    # SSH
sudo firewall-cmd --permanent --add-port=514/udp   # Syslog (Meraki)
sudo firewall-cmd --permanent --add-port=514/tcp   # Syslog TCP
sudo firewall-cmd --permanent --add-port=8000/tcp  # Splunk Web UI
sudo firewall-cmd --permanent --add-port=9997/tcp  # Forwarder input
sudo firewall-cmd --reload
```

### 2.3 SSH Hardening

```bash
sudo vi /etc/ssh/sshd_config
```

Applied configuration:
```
PermitRootLogin no
PasswordAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

```bash
sudo systemctl restart sshd
```

### 2.4 Kernel Hardening

```bash
sudo vi /etc/sysctl.d/99-hardening.conf
```

```
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv6.conf.all.disable_ipv6 = 1
kernel.dmesg_restrict = 1
```

```bash
sudo sysctl -p /etc/sysctl.d/99-hardening.conf
```

### 2.5 Disable Unnecessary Services

```bash
sudo systemctl disable --now bluetooth 2>/dev/null
sudo systemctl disable --now cups 2>/dev/null
sudo systemctl disable --now avahi-daemon 2>/dev/null
```

### 2.6 Enable Audit Logging

```bash
sudo dnf install -y audit
sudo systemctl enable --now auditd
```

**Verification — Phase 2:**
```bash
sudo firewall-cmd --list-all
sudo sshd -T | grep -E "permitrootlogin|maxauthtries"
sudo sysctl net.ipv4.ip_forward
sudo systemctl status auditd
```

---

## Phase 3 — Docker Installation

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager \
  --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io \
  docker-compose-plugin
sudo systemctl enable --now docker

# Add admin user to docker group (log out and back in after)
sudo usermod -aG docker $USER
```

**Verification — Phase 3:**
```bash
docker --version
docker compose version
docker run hello-world
```

The `hello-world` container should print a success message 
confirming Docker is fully operational.

---

## Phase 4 — Splunk Enterprise Deployment

Full Docker deployment details: [Docker.md](Docker.md)

### 4.1 Create Working Directory

```bash
mkdir -p ~/soc-deployment/splunk
cd ~/soc-deployment/splunk
```

### 4.2 Create Docker Compose File

```bash
vi docker-compose.yml
```

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

### 4.3 Start Splunk

```bash
docker compose up -d
```

### 4.4 Monitor Initialization

```bash
docker logs -f splunk
```

Wait for the initialization sequence to complete before 
proceeding. First start takes several minutes.

**Verification — Phase 4:**
```bash
docker ps
```

Confirm the `splunk` container shows `Up` status.

Open a browser and navigate to:
```
http://<soc-server-ip>:8000
```

Log in with `admin` and the password set in the compose file.
If the Splunk Web UI loads, the deployment is successful.

---

## Phase 5 — Splunk Index Configuration

Full index configuration details: [Splunk.md](Splunk.md)

### 5.1 Enter the Splunk Container

```bash
docker exec -it splunk bash
```

### 5.2 Configure Indexes

```bash
vi /opt/splunk/etc/system/local/indexes.conf
```

```ini
[meraki]
homePath = $SPLUNK_DB/meraki/db
coldPath = $SPLUNK_DB/meraki/colddb
thawedPath = $SPLUNK_DB/meraki/thaweddb
maxTotalDataSizeMB = 50000
frozenTimePeriodInSecs = 7776000

[sysmon]
homePath = $SPLUNK_DB/sysmon/db
coldPath = $SPLUNK_DB/sysmon/colddb
thawedPath = $SPLUNK_DB/sysmon/thaweddb
maxTotalDataSizeMB = 100000
frozenTimePeriodInSecs = 7776000

[windows]
homePath = $SPLUNK_DB/windows/db
coldPath = $SPLUNK_DB/windows/colddb
thawedPath = $SPLUNK_DB/windows/thaweddb
maxTotalDataSizeMB = 50000
frozenTimePeriodInSecs = 31536000
```

### 5.3 Configure Sourcetype Parsing

```bash
vi /opt/splunk/etc/system/local/props.conf
```

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

### 5.4 Configure Log Routing

```bash
vi /opt/splunk/etc/system/local/transforms.conf
```

```ini
[route_meraki]
REGEX = .
DEST_KEY = _MetaData:Index
FORMAT = meraki
```

### 5.5 Enable Receiving Port

In Splunk Web:
```
Settings → Forwarding and receiving → Configure receiving
→ New Receiving Port → 9997 → Save
```

### 5.6 Restart Splunk

```bash
exit
docker compose restart splunk
```

**Verification — Phase 5:**

In Splunk Web, navigate to:
```
Settings → Indexes
```

Confirm `meraki`, `sysmon`, and `windows` indexes are listed.

---

## Phase 6 — Cisco Meraki Syslog Configuration

### 6.1 Configure Syslog in Meraki Dashboard

```
1. Log into Meraki Dashboard
2. Network-wide → General → Reporting → Syslog servers
3. Add a syslog server:
   Server IP:  <soc-server-ip>
   Port:       514
   Roles:      Appliance Event log, Appliance URLs
4. Save
```

**Verification — Phase 6:**

Wait 2-3 minutes, then run in Splunk Search:
```spl
index=meraki
| head 10
```

Events should appear within a few minutes of saving the 
Meraki configuration.

---

## Phase 7 — Windows Endpoint Configuration

Full endpoint configuration details:
- [Sysmon.md](Sysmon.md)
- [Universal-Forwarder.md](Universal-Forwarder.md)

### 7.1 Grant Sysmon Log Read Permission

Run in Command Prompt as Administrator on the endpoint:

```cmd
wevtutil set-log "Microsoft-Windows-Sysmon/Operational" /ca:O:BAG:SYD:(A;;0x1;;;BA)(A;;0x1;;;SY)(A;;0x1;;;LS)(A;;0x1;;;NS)(A;;0x1;;;S-1-5-32-573)
```

### 7.2 Install Sysmon

```powershell
# Run as Administrator
sysmon64.exe -accepteula -i sysmonconfig.xml
```

### 7.3 Install Splunk Universal Forwarder

```powershell
msiexec.exe /i splunkforwarder-x64.msi `
  AGREETOLICENSE=Yes `
  RECEIVING_INDEXER="<soc-server-ip>:9997" `
  WINEVENTLOG_APP_ENABLE=1 `
  WINEVENTLOG_SEC_ENABLE=1 `
  WINEVENTLOG_SYS_ENABLE=1 /quiet
```

### 7.4 Configure inputs.conf

```
Location: C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = sysmon
disabled = false
renderXml = true
sourcetype = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational

[WinEventLog://Security]
index = windows
disabled = false
sourcetype = WinEventLog:Security

[WinEventLog://System]
index = windows
disabled = false
sourcetype = WinEventLog:System

[WinEventLog://Application]
index = windows
disabled = false
sourcetype = WinEventLog:Application
```

### 7.5 Disable Deployment Client

```
Location: C:\Program Files\SplunkUniversalForwarder\etc\system\local\deploymentclient.conf
```

```ini
[deployment-client]
disabled = true
```

This prevents the forwarder from attempting to connect to 
a non-existent Deployment Server, which causes retry loops 
that can interfere with normal log forwarding.

### 7.6 Restart the Forwarder

```powershell
Restart-Service SplunkForwarder
```

**Verification — Phase 7:**

```spl
index=sysmon EventCode=1
| head 10
```

```spl
index=windows EventCode=4624
| head 10
```

Both searches should return recent events from the endpoint.

---

## Phase 8 — End-to-End Verification

Run this final verification search in Splunk:

```spl
| eventcount summarize=false index=*
| where index!="_internal" AND index!="_audit"
| table index, count
```

Expected output:

| index | count |
|---|---|
| meraki | (non-zero) |
| sysmon | (non-zero) |
| windows | (non-zero) |

If all three show non-zero counts, the full pipeline is 
operational end-to-end.

---

## Deployment Sequence Summary

```
Phase 1: Infrastructure    VMware ESXi → Rocky Linux VM
Phase 2: Hardening         OS secured before production use
Phase 3: Docker            Container runtime installed
Phase 4: Splunk            SIEM deployed and accessible
Phase 5: Indexes           Data storage configured
Phase 6: Meraki            Network logs flowing
Phase 7: Endpoints         Endpoint telemetry flowing
Phase 8: Verification      Full pipeline confirmed end-to-end
```

---

> **Confidentiality Note:** IP addresses, hostnames, and 
> organization-specific values shown as placeholders 
> (e.g. `<soc-server-ip>`) throughout this document. 
> Actual values are not documented in this repository.
