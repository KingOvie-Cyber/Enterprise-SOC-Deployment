# Hardening
## Rocky Linux Production Security Hardening

---

## Introduction

This document describes the security hardening applied to the 
Rocky Linux SOC server prior to production deployment. Hardening 
was completed before the server was connected to the production 
network or began receiving any log data.

Security configuration is a prerequisite — not an afterthought.

The hardening approach is aligned with CIS (Center for Internet 
Security) Benchmark recommendations for RHEL-compatible Linux 
distributions, adapted for the specific SOC server role and 
operational requirements of this environment.

---

## Purpose

The SOC server occupies a position of elevated trust within the 
organization's network. It receives security logs from all 
monitored assets. If the SOC server itself were compromised, 
an attacker would gain visibility into the organization's entire 
detection capability.

For this reason, the attack surface of the SOC server was 
reduced aggressively:

- Only ports required for SOC operation are open
- Only services required for SOC operation are running
- All unnecessary software, services, and access paths 
  were removed or disabled
- System activity is logged via auditd for forensic capability

---

## Hardening Checklist

### Access Control

- [x] Root login disabled via SSH (`PermitRootLogin no`)
- [x] SSH password authentication configured appropriately
- [x] Maximum authentication attempts limited (`MaxAuthTries 3`)
- [x] Idle session timeout configured (`ClientAliveInterval 300`)
- [x] Sudo access restricted to named admin accounts only
- [x] Umask set to 027 in `/etc/profile` and `/etc/bashrc`

### SSH Configuration Applied

```bash
# /etc/ssh/sshd_config — changes applied
PermitRootLogin no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

---

### Network Hardening

- [x] Firewalld enabled and active at boot
- [x] Only required ports open (see port table below)
- [x] IPv6 disabled (not required in this environment)
- [x] IP forwarding disabled
- [x] ICMP redirects disabled (accept and send)
- [x] TCP SYN cookies enabled
- [x] Reverse path filtering enabled

#### Open Ports (firewalld)

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH administrative access |
| 514 | UDP | Syslog receive (Cisco Meraki) |
| 514 | TCP | Syslog receive (TCP fallback) |
| 8000 | TCP | Splunk Web UI |
| 9997 | TCP | Splunk Universal Forwarder input |

All other inbound traffic is dropped by default.

#### Firewall Commands Applied
#### Use sudo for Admin Privilidge
```bash
firewall-cmd --permanent --add-port=22/tcp
firewall-cmd --permanent --add-port=514/udp
firewall-cmd --permanent --add-port=514/tcp
firewall-cmd --permanent --add-port=8000/tcp
firewall-cmd --permanent --add-port=9997/tcp
firewall-cmd --permenent --add-port=8089/tcp
firewall-cmd --reload
```

---

### Kernel Hardening (sysctl)

Applied via `/etc/sysctl.d/99-hardening.conf`:

```bash
# Disable IP forwarding
net.ipv4.ip_forward = 0

# Disable ICMP redirect acceptance
net.ipv4.conf.all.accept_redirects = 0

# Disable ICMP redirect sending
net.ipv4.conf.all.send_redirects = 0

# Enable TCP SYN cookies
net.ipv4.tcp_syncookies = 1

# Enable reverse path filtering
net.ipv4.conf.all.rp_filter = 1

# Disable IPv6
net.ipv6.conf.all.disable_ipv6 = 1

# Restrict dmesg access to root
kernel.dmesg_restrict = 1
```

Applied with:
```bash
sudo sysctl -p /etc/sysctl.d/99-hardening.conf
```

---

### Service Hardening

- [x] Postfix disabled and masked
- [x] Bluetooth disabled and masked
- [x] CUPS (print service) disabled and masked
- [x] Avahi-daemon (mDNS) disabled and masked
- [x] Only required services verified running

```bash
sudo systemctl disable --now postfix 2>/dev/null
sudo systemctl disable --now bluetooth 2>/dev/null
sudo systemctl disable --now cups 2>/dev/null
sudo systemctl disable --now avahi-daemon 2>/dev/null
```

Verify only required services are running:
```bash
systemctl list-units --type=service --state=running
```

---

### Logging and Auditing

- [x] Auditd installed and enabled at boot
- [x] System-level activity logged for forensic capability
- [x] Log rotation configured

```bash
sudo dnf install -y audit
sudo systemctl enable --now auditd
```

Auditd captures:
- Privileged command execution
- User and group modifications
- File permission changes
- Authentication events
- System call activity

---

### Package Management

- [x] System fully patched before deployment (`dnf update -y`)
- [x] Only required packages installed
- [x] RPM GPG key verification enabled

```bash
sudo dnf update -y
```

---

### File System

- [x] SUID/SGID files audited
- [x] World-writable files reviewed
- [x] Sticky bit verified on /tmp

```bash
# Audit SUID/SGID files
find / -perm /4000 -o -perm /2000 2>/dev/null

# Find world-writable files
find / -perm -002 -not -type l 2>/dev/null

# Verify sticky bit on /tmp
ls -la / | grep tmp
```

---

## Verification Commands

Run these after applying hardening to confirm configuration:

```bash
# Verify SSH hardening
sshd -T | grep -E "permitrootlogin|maxauthtries|clientaliveinterval"

# Verify only required ports are open
sudo ss -tulnp
sudo firewall-cmd --list-all

# Verify sysctl settings applied
sysctl net.ipv4.ip_forward
sysctl net.ipv4.conf.all.accept_redirects
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv6.conf.all.disable_ipv6

# Verify auditd is running
systemctl status auditd

# Verify only required services are running
systemctl list-units --type=service --state=running
```

---

## Docker-Specific Security Considerations

Since Splunk runs inside Docker, additional security 
considerations apply at the container layer:

**Port exposure discipline:**
Only ports explicitly mapped in `docker-compose.yml` are 
accessible from the network. The Splunk management port 
8089 is NOT mapped — it is only accessible from within 
the container itself.

**Non-root container execution:**
The official Splunk Docker image runs Splunk under a 
non-root user internally, limiting the impact of any 
application-level vulnerability within the container.

**Volume isolation:**
Persistent data lives in named Docker volumes, not in 
directories accessible to the host OS directly.

---

## Hardening Trade-offs and Decisions

| Control | Applied | Reasoning |
|---|---|---|
| SSH root login disabled | Yes | Eliminates direct root access path |
| IPv6 disabled | Yes | Not required in this environment |
| Firewalld (not iptables directly) | Yes | Easier to manage, persistent across reboots |
| GUI enabled on server | Yes | Operational convenience; additional services hardened |
| SELinux enforcing | Evaluated | Default Rocky Linux SELinux policy maintained |

---

## Ongoing Hardening Maintenance

Hardening is not a one-time event. The following should 
be reviewed regularly:

- Apply security patches as released (`dnf update`)
- Review open ports periodically (`firewall-cmd --list-all`)
- Review running services periodically
- Review auditd logs for unexpected privileged activity
- Re-run verification commands after any significant 
  system change

---

## Best Practices Reference

The hardening approach in this document draws from:

- CIS Rocky Linux 10 Benchmark
- NIST SP 800-123 (Guide to General Server Security)
- NSA/CISA Linux Hardening Guide

---

> **Confidentiality Note:** Specific system identifiers, 
> network addresses, and organization-specific configuration 
> values have been sanitized. This document describes 
> hardening methodology and applied controls only.
