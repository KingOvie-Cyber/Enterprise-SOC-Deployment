# Rocky Linux Hardening Checklist

### Applied to: syslog-collector VM and splunk-host VM

## Access Control

- [x] Root login disabled in /etc/ssh/sshd_config (PermitRootLogin no)
- [x] SSH password authentication disabled (PasswordAuthentication no)
- [x] SSH key-based authentication configured
- [x] Sudo access restricted to named admin accounts
- [x] Umask set to 027 in /etc/profile and /etc/bashrc
- [x] Idle session timeout configured (TMOUT=600)

## SSH Configuration Applied

```bash
# Changes made to /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

## Network Hardening

- [x] Firewalld enabled and active (systemctl enable firewalld)
- [x] Only required ports open:
  - syslog-collector: UDP 514 (syslog), TCP 22 (SSH)
  - splunk-host: TCP 8000 (web UI), TCP 9997 (forwarder), TCP 22 (SSH)
- [x] IPv6 disabled in /etc/sysctl.conf
- [x] net.ipv4.ip_forward = 0
- [x] net.ipv4.conf.all.accept_redirects = 0
- [x] net.ipv4.conf.all.send_redirects = 0
- [x] net.ipv4.tcp_syncookies = 1
- [x] net.ipv4.conf.all.rp_filter = 1

### Firewall Rules Applied

```bash
# syslog-collector
firewall-cmd --permanent --add-port=514/udp
firewall-cmd --permanent --add-port=514/tcp
firewall-cmd --permanent --add-port=22/tcp
firewall-cmd --reload

# splunk-host
firewall-cmd --permanent --add-port=8000/tcp
firewall-cmd --permanent --add-port=9997/tcp
firewall-cmd --permanent --add-port=22/tcp
firewall-cmd --reload
```

### Kernel Hardening Applied

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

## Services Disabled

- [x] Postfix disabled and masked
- [x] Bluetooth disabled and masked
- [x] CUPS disabled and masked
- [x] Avahi-daemon disabled
- [x] Only required services running (verified with: systemctl list-units --type=service --state=running)

```bash
systemctl disable --now postfix
systemctl disable --now bluetooth
systemctl disable --now cups
systemctl disable --now avahi-daemon
```

## Logging & Auditing

- [x] Auditd installed and enabled
- [x] Audit rules configured for:
  - Privileged command execution
  - User/group modifications
  - File permission changes
  - Authentication events
- [x] Rsyslog configured for remote forwarding (syslog-collector only)
- [x] Log rotation configured (/etc/logrotate.d/)

```bash
systemctl enable --now auditd
```

## File System

- [x] /tmp mounted with noexec, nosuid, nodev options
- [x] /var/log on separate partition
- [x] Sticky bit verified on /tmp (chmod +t /tmp)
- [x] SUID/SGID files audited (find / -perm /4000 -o -perm /2000)
- [x] World-writable files audited and reviewed

## Package Management

- [x] System fully patched (dnf update -y)
- [x] Only required packages installed
- [x] RPM GPG keys verified
- [ ] DNF automatic security updates configured (planned)

## Verification Commands Used

```bash
# Verify SSH hardening
sshd -T | grep -E "permitrootlogin|passwordauthentication"

# Verify open ports only
ss -tulnp

# Verify firewall rules
firewall-cmd --list-all

# Verify running services
systemctl list-units --type=service --state=running

# Verify sysctl settings
sysctl -a | grep -E "ip_forward|accept_redirects|syncookies|rp_filter"

# Verify auditd running
systemctl status auditd
```
