# SOC Architecture Overview

## Network & Data Flow Diagram

```
                    COMPANY NETWORK
                          |
              +-----------+-----------+
              |                       |
        [Cisco Meraki]          [Endpoints]
        Network Firewall         Sysmon
        & Switch Logs            Telemetry
              |                       |
              +----------+------------+
                         |
                  [SYSLOG COLLECTOR]
                  Rocky Linux VM
                  (CIS Hardened)
                  Rsyslog
                         |
                         v
                   [SPLUNK SIEM]
                   Rocky Linux VM
                   Docker Container
                   Splunk Enterprise
                         |
                   +-----+-----+
                   |           |
             [Dashboards]  [Alerts]
                       
               
```

## Design Decisions

**Why two separate physical servers?**
Separating the syslog collector from the Splunk host ensures 
that log collection continues even if the SIEM goes down for 
maintenance. It also isolates the attack surface — 
the collector only accepts inbound syslog, the SIEM 
only receives from the collector.

**Why Rocky Linux?**
Rocky Linux is RHEL-compatible, enterprise-grade, and 
community-supported with a long lifecycle. It mirrors 
what most enterprises run in production, making the 
skills directly transferable.

**Why Docker for Splunk?**
Containerizing Splunk allows version-controlled deployments, 
easy upgrades, and clean separation of the application 
from the OS. Volume mounts ensure index data persists 
independently of the container lifecycle.

