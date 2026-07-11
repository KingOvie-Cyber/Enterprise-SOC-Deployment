# Rocky Linux Hardening Checklist
## Applied to SOC Server — Pre-Production Security Configuration

---

## Overview

This checklist documents the security hardening applied
to the Rocky Linux SOC server before connecting to the
production network. Hardening is aligned with CIS
Benchmark recommendations for RHEL-compatible systems.

**Status:** All checked items [x] have been applied
and verified in this environment.

---

## 1. Access Control

- [x] Root login disabled via SSH (`PermitRootLogin no`)
- [x] SSH MaxAuthTries set to 3
- [x] Idle session timeout configured (`ClientAliveInterval 300`)
- [x] Sudo access restricted to named admin accounts
- [x] Umask set to 027

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

---

## 2. Firewall (firewalld)

- [x] Firewalld enabled and active at boot
- [x] Only required ports open (see table below)
- [x] All other inbound traffic dropped by default

| Port | Protocol | Purpose | Status |
|---|---|---|---|
| 22 | TCP | SSH | Open |
| 514 | UDP | Syslog (Meraki) | Open |
| 514 | TCP | Syslog TCP | Open |
| 8000 | TCP | Splunk Web UI | Open |
| 9997 | TCP | Forwarder input | Open |
| All others | - | - | Blocked |

```bash
firewall-cmd --permanent --add-port=22/tcp
firewall-cmd --permanent --add-port=514/udp
firewall-cmd --permanent --add-port=514/tcp
firewall-cmd --permanent --add-port=8000/tcp
firewall-cmd --permanent --add-port=9997/tcp
firewall-cmd --reload
```

---

## 3. Kernel Hardening (sysctl)

- [x] IP forwarding disabled
- [x] ICMP redirects disabled (accept and send)
- [x] TCP SYN cookies enabled
- [x] Reverse path filtering enabled
- [x] IPv6 disabled
- [x] dmesg access restricted

```bash
# /etc/sysctl.d/99-hardening.conf
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv6.conf.all.disable_ipv6 = 1
kernel.dmesg_restrict = 1
```

---

## 4. Service Hardening

- [x] Postfix disabled and masked
- [x] Bluetooth disabled and masked
- [x] CUPS disabled and masked
- [x] Avahi-daemon disabled and masked
- [x] Only required services verified running

```bash
systemctl disable --now postfix bluetooth cups avahi-daemon
```

---

## 5. Audit Logging

- [x] Auditd installed and enabled at boot
- [x] System activity logged for forensic capability
- [x] Log rotation configured

```bash
dnf install -y audit
systemctl enable --now auditd
```

---

## 6. Package Management

- [x] System fully patched before deployment
- [x] Only required packages installed

```bash
dnf update -y
```

---

## 7. File System

- [x] SUID/SGID files audited
- [x] World-writable files reviewed

```bash
find / -perm /4000 -o -perm /2000 2>/dev/null
find / -perm -002 -not -type l 2>/dev/null
```

---

## Verification Commands

```bash
# Verify SSH
sshd -T | grep -E "permitrootlogin|maxauthtries"

# Verify firewall
firewall-cmd --list-all

# Verify sysctl
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.disable_ipv6

# Verify auditd
systemctl status auditd

# Verify running services
systemctl list-units --type=service --state=running
```

---

> **Confidentiality Note:** System-specific identifiers
> have been sanitized. This checklist documents
> methodology only.
