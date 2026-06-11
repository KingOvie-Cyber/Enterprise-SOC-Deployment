# Rocky Linux Hardening Checklist
### Applied to: syslog-collector VM and splunk-host VM

## Access Control
- [ ] Root login disabled in /etc/ssh/sshd_config (PermitRootLogin no)
- [ ] SSH password authentication disabled (PasswordAuthentication no)
- [ ] SSH key-based authentication configured
- [ ] Sudo access restricted to named admin accounts
- [ ] Umask set to 027 in /etc/profile and /etc/bashrc
- [ ] Idle session timeout configured (TMOUT=600)

## Network Hardening
- [ ] Firewalld enabled and active (systemctl enable firewalld)
- [ ] Only required ports open:
  - syslog-collector: UDP 514 (syslog), TCP 22 (SSH)
  - splunk-host: TCP 8000 (web UI), TCP 9997 (forwarder), TCP 22 (SSH)
- [ ] IPv6 disabled in /etc/sysctl.conf
- [ ] net.ipv4.ip_forward = 0
- [ ] net.ipv4.conf.all.accept_redirects = 0
- [ ] net.ipv4.conf.all.send_redirects = 0
- [ ] net.ipv4.tcp_syncookies = 1
- [ ] net.ipv4.conf.all.rp_filter = 1

## Services
- [ ] Postfix disabled and masked
- [ ] Bluetooth disabled and masked
- [ ] CUPS disabled and masked
- [ ] Avahi-daemon disabled
- [ ] Only required services running (verified with: systemctl list-units --type=service --state=running)

## Logging & Auditing
- [ ] Auditd installed and enabled
- [ ] Audit rules configured for:
  - Privileged command execution
  - User/group modifications
  - File permission changes
  - Authentication events
- [ ] Rsyslog configured for remote forwarding (syslog-collector only)
- [ ] Log rotation configured (/etc/logrotate.d/)

## File System
- [ ] /tmp mounted with noexec, nosuid, nodev options
- [ ] /var/log on separate partition
- [ ] Sticky bit verified on /tmp (chmod +t /tmp)
- [ ] SUID/SGID files audited (find / -perm /4000 -o -perm /2000)
- [ ] World-writable files audited and reviewed

## Package Management
- [ ] System fully patched (dnf update -y)
- [ ] Only required packages installed
- [ ] RPM GPG keys verified
- [ ] DNF automatic security updates configured

## Verification Commands Used
```bash
# Verify SSH hardening
sshd -T | grep -E "permitrootlogin|passwordauthentication"

# Verify open ports
ss -tulnp

# Verify firewall rules  
firewall-cmd --list-all

# Verify running services
systemctl list-units --type=service --state=running

# Verify sysctl settings
sysctl -a | grep -E "ip_forward|accept_redirects|syncookies|rp_filter"
```
