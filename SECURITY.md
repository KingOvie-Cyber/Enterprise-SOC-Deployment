# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in the
documentation, configuration examples, or code
in this repository, please report it responsibly.

**Do NOT open a public GitHub issue for security vulnerabilities.**

### How to Report

Contact directly via LinkedIn: [Ogheneovie Emezana](https://www.linkedin.com/in/ogheneovie-emezana)

Please include:
- Description of the vulnerability
- Location in the repository (file and line if applicable)
- Potential impact
- Suggested fix (if known)

---

## Scope

This repository contains:
- Documentation of a SOC deployment methodology
- Sanitized configuration file examples
- SPL query examples
- Detection rule logic

### In Scope
- Incorrect security guidance that could lead to
  misconfiguration if followed
- SPL queries or detection logic with significant
  security flaws
- Configuration examples that expose sensitive
  architecture details inadvertently

### Out of Scope
- The live production environment itself
  (not publicly accessible)
- Third-party tools referenced in this repository
  (Splunk, Sysmon, Cisco Meraki — report to vendors)

---

## Confidentiality Commitment

This repository has been reviewed to ensure:

- No IP addresses from the production environment
  are disclosed
- No hostnames or system identifiers are disclosed
- No credentials, tokens, or secrets are present
- No network topology details that could aid
  an attacker are disclosed
- All sensitive values are replaced with
  clearly-marked placeholders (e.g. `<REDACTED>`,
  `<soc-server-ip>`)

This repository complies with ISO 27001
confidentiality principles — it documents
methodology, not the organization's
security posture.

---

## Supported Versions

This repository documents a point-in-time deployment.
Configuration examples and SPL queries are provided
as reference material — always validate against
current vendor documentation before use in production.

| Component | Version Documented |
|---|---|
| Splunk Enterprise | 10.4.0 |
| Rocky Linux | 10.x |
| Sysmon | Latest (Olaf Hartong config) |
| Docker | Current stable |
| VMware ESXi | Current |
