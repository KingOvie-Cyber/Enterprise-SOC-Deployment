# Lessons Learned
## Reflections, Insights, and Recommendations

---

## Introduction

This document captures the most important lessons learned 
during the design, deployment, and operation of the 
Enterprise SOC. It is written for two audiences:

**For future reference:** Anyone maintaining or extending 
this environment should read this document before making 
significant changes. These lessons represent hard-won 
operational knowledge that is not captured anywhere else.

**For professional development:** This document demonstrates 
the kind of reflective engineering practice that separates 
someone who deployed a system from someone who truly 
understands it.

---

## Lesson 1 — Security Hardening Must Precede Connectivity

### What Happened
The instinct during deployment was to get Splunk running 
and receiving logs quickly — and then address hardening 
afterward. This approach was deliberately reversed: 
hardening was completed before the server was connected 
to the production network.

### Why It Matters
A SIEM server that is not hardened before it begins 
receiving data is a liability, not an asset. If it is 
compromised before hardening is complete, an attacker 
gains visibility into the organization's monitoring 
capability. The hardening window is also the window 
of highest risk — once connected and receiving data, 
changes become more disruptive.

### The Principle
Security configuration is a prerequisite. Not an 
afterthought. Not something to do "when there is time."

---

## Lesson 2 — Silent Failures Are the Most Dangerous

### What Happened
Multiple failures during this deployment produced no 
error messages — data simply stopped flowing, or never 
started flowing, with no obvious indication of why. 
The `disabled = fasle` typo. The missing NetworkConnect 
Sysmon rule. The Deployment Client retry loop masking 
other issues. The fishbucket losing its read position.

### Why It Matters
Silent failures are harder to diagnose than noisy failures 
precisely because there is no error to search for. A SOC 
that appears healthy but has a silent data gap is worse 
than one that is obviously broken — the analyst may make 
decisions based on the false assumption that the absence 
of alerts means the absence of threats.

### The Principle
Build verification into every stage of deployment. Never 
assume something is working — prove it. The diagnostic 
flowchart in [15-Troubleshooting.md](15-Troubleshooting.md) 
was developed directly from the experience of hunting 
silent failures.

---

## Lesson 3 — Work Backward When Diagnosing Pipeline Failures

### What Happened
When events were not appearing in Splunk, the initial 
instinct was to look at Splunk configuration first. 
The more reliable approach, developed through experience, 
was to work backward from the source:

```
1. Does the event exist locally on the endpoint?
2. Is the network path open between endpoint and SIEM?
3. Is the forwarder service running?
4. Does splunkd.log show errors?
5. Are the configuration files syntactically correct?
6. Does the index exist in Splunk?
```

Starting from the source and working toward Splunk 
isolates the failure layer precisely, rather than 
guessing and trying random fixes.

### The Principle
Diagnose systematically. Every pipeline has discrete 
stages. Find the stage where the data stops. Fix 
that stage. Do not skip steps.

---

## Lesson 4 — Inspect Raw Events Before Building Query Logic

### What Happened
Initial Meraki queries failed because the assumed 
field name (`signature`) did not match the actual 
field name (`type`). This is a common mistake when 
working with a new data source — assuming that field 
names match what other platforms use.

The fix was simple: inspect a raw event first.

```spl
index=meraki sourcetype=meraki_syslog
| head 1
| table _raw
```

Once the actual log format was visible, the correct 
field names became immediately apparent.

### The Principle
Never write production queries based on assumed field 
names. Always inspect raw events first. This applies 
to every new data source onboarded into any SIEM.

---

## Lesson 5 — Containerization Introduces a New Category of Operational Concerns

### What Happened
Deploying Splunk in Docker provided real benefits — 
clean lifecycle management, persistent volumes, 
portable configuration. But it also introduced 
a category of issues that native installations 
do not have:

- The Ansible entrypoint causing container exits 
  on internal restarts
- The `ansible` user appearing in the container 
  rather than the expected `splunk` user
- ESXi console clipboard not working (not Docker-specific, 
  but compounded the container management challenge)
- Port mapping requiring explicit verification after 
  every container recreation

None of these issues appeared in tutorials or vendor 
documentation. They only surface in production.

### The Principle
Containerization is not a simplification — it is a 
trade of one set of operational concerns for another. 
The container layer must be understood, not treated 
as a black box. Always verify port mappings, volume 
mounts, and restart behavior explicitly.

---

## Lesson 6 — The Deployment Client Is Not Optional to Disable

### What Happened
The Splunk Universal Forwarder's Deployment Client 
feature generated continuous handshake retry errors 
that flooded splunkd.log. This noise made it 
significantly harder to identify the real causes 
of data forwarding failures. The Deployment Client 
was configured with the wrong address (the indexer's 
data port rather than a deployment server), causing 
constant protocol mismatch errors.

Disabling the Deployment Client via `deploymentclient.conf` 
eliminated the noise entirely and immediately made 
log troubleshooting more tractable.

### The Principle
In environments without a Deployment Server, always 
disable the Deployment Client. It is not a default-safe 
feature — leaving it enabled when there is no 
Deployment Server creates operational noise that 
masks real problems.

---

## Lesson 7 — The Source of Truth for Sysmon Configuration Is the Running Process

### What Happened
After applying a new Sysmon configuration, verification 
was done by checking the XML file on disk rather than 
the running configuration. The file looked correct. 
But `sysmon64.exe -c` (which shows the actual loaded 
configuration) revealed a different state — the 
service had not fully applied the new configuration.

The lesson: configuration files on disk and the 
running service state are two different things. 
Always verify what is actually loaded.

### The Principle
`sysmon64.exe -c` is the source of truth for what 
Sysmon is actually doing. The XML file on disk is 
the intended state. They may not always match.

---

## Lesson 8 — Detection Value Requires Baseline Knowledge

### What Happened
The first time the Top Visited Domains panel was reviewed, 
the initial instinct was to investigate several 
destinations as potentially suspicious. After research:
- `westeurope.livediagnostics.monitor.azure.com` — Azure monitoring agent
- `905469987510.data-kinesis.us-east-1.amazonaws.com` — AWS Kinesis telemetry
- `f.c2r.ts.cdn.office.net` — Office 365 Click-to-Run updates

All legitimate. Without knowing what "normal" looks 
like in this environment, it is impossible to 
identify what is anomalous.

### The Principle
A SOC analyst needs to know the environment as deeply 
as an attacker would. Establishing and documenting 
baseline normal behavior — which domains, which ports, 
which processes, which users — is foundational work 
that must precede meaningful threat detection.

---

## Lesson 9 — Real Incidents Validate the Investment

### What Happened
Two real incidents occurred during the SOC's early 
operational period:

**Incident 1:** A spike of 64 failed login attempts 
was detected in the Failed Logins Today dashboard 
panel — the organization had no visibility into this 
before the SOC was built.

**Incident 2:** A departed staff member accessed 
company email from an unauthorized device after 
their departure. The detection capability built 
in the SOC identified this gap and informed a 
broader conversation about offboarding procedures 
and account lifecycle management.

Neither incident required a custom detection rule 
or complex investigation — the core monitoring 
capability surfaced both. This validated the entire 
deployment investment within weeks of going live.

### The Principle
Start with broad visibility before sophisticated 
detection. The most valuable security investment 
is often the first — simply knowing what is 
happening in the environment.

---

## Lesson 10 — Documentation Is Part of the Deployment

### What Happened
During the deployment, there was pressure to focus 
exclusively on getting the technical components 
working. Documentation was treated as something 
to do "after" the deployment was complete.

The problem: by the time a component was working, 
the specific commands run, the decisions made, and 
the problems encountered were already fading from 
memory. The troubleshooting section in particular 
required reconstructing events from notes and 
memory rather than real-time capture.

### The Principle
Documentation happens during deployment, not after. 
Commands that work should be captured immediately. 
Decisions made should be noted at the time they are 
made. Problems encountered and solved should be 
recorded as they happen.

This repository represents the documentation that 
should have been built in parallel with the 
infrastructure — not reconstructed afterward. 
Future projects will use a parallel documentation 
approach from day one.

---

## Summary of Key Principles

| # | Principle |
|---|---|
| 1 | Harden before connecting |
| 2 | Silent failures require active verification, not passive assumption |
| 3 | Diagnose pipelines backward from the source |
| 4 | Inspect raw events before writing query logic |
| 5 | Understand the container layer — don't treat it as a black box |
| 6 | Disable the Deployment Client in environments without a Deployment Server |
| 7 | Running process state is the truth — not the file on disk |
| 8 | Baseline knowledge precedes meaningful detection |
| 9 | Broad visibility delivers value before sophisticated detection |
| 10 | Document during deployment, not after |

---

## What Would Be Done Differently

If this deployment were repeated from the beginning:

**1. Parallel documentation from day one**
Every command, decision, and problem captured 
in real time rather than reconstructed afterward.

**2. Native Splunk install instead of Docker initially**
Docker adds operational value but also operational 
complexity. Starting with a native install to establish 
baseline functionality, then migrating to containerization, 
would have reduced early troubleshooting complexity.

**3. Intermediate syslog collector for Meraki**
The direct Meraki-to-Splunk syslog path has no 
delivery guarantee. An intermediate rsyslog collector 
with disk buffering would prevent log loss during 
any Splunk maintenance window.

**4. Asset inventory built first**
Deploying monitoring before building an inventory 
of what was being monitored meant IP addresses 
appeared in dashboards without immediate context 
about which device they represented. Asset inventory 
is a prerequisite for identity-aware monitoring.

**5. Phased onboarding with verification at each stage**
The phased approach used in the deployment was correct 
in principle but not always applied strictly enough. 
Some stages were rushed, leading to issues that 
took longer to diagnose because multiple changes 
had been made simultaneously.

---

> **Confidentiality Note:** Specific incidents 
> referenced in this document have been described 
> at a general level without including any 
> organizational identifiers, affected accounts, 
> or security posture details. This document 
> describes methodology and reflection only.
