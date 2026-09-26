# Azure Honeypot — T-Pot Threat Intelligence Lab

## Overview

This project deploys a [T-Pot](https://github.com/telekom-security/tpotce) multi-honeypot platform on a dedicated Azure Virtual Machine to observe real-world attacker behavior against internet-facing systems. The VM is intentionally exposed to the internet on all ports to attract traffic, while network-level controls isolate it from any other resources so that a compromise cannot pivot laterally or escalate beyond the honeypot itself.

**Goal:** Collect live attack telemetry (scan patterns, exploit attempts, credential stuffing, malware drops, etc.) in a safe, contained environment, and use that data to build detection and analysis skills.

## Architecture

```mermaid
flowchart LR
    Internet([Public Internet])
    Admin([Security Analyst / Admin])

    subgraph VNET["Virtual Network (VNet)"]
        subgraph SUBNET["Subnet"]
            subgraph NSG["Network Security Group (NSG)"]
                direction LR
                RuleIn["Inbound Rule<br/>ALLOW ALL (*)"]
                subgraph VM["Virtual Machine (Linux Host)"]
                    subgraph TPOT["T-Pot Honeypot Platform"]
                        direction TB
                        Auth{"Auth Gate<br/>User + Password"}
                        UI["T-Pot Web Interface<br/>Kibana / Cockpit"]
                        Traps["Honeypot Daemons<br/>Cowrie, Dionaea, etc."]
                        Auth -->|Authorized| UI
                    end
                end
                RuleOut["Outbound Rule<br/>DENY ALL (*)"]
            end
        end
    end

    Drop([Egress Blocked])

    %% 1. Inbound Attack Flow to Traps
    Internet ==>|1. Attack Probes / Exploits| RuleIn
    RuleIn ==>|Forwarded| Traps

    %% 2. Admin Ingress with Auth
    Admin ==>|2. HTTPS Access :64297| RuleIn
    RuleIn ==>|Login Request| Auth

    %% 3. Outbound Containment Flow
    VM -.->|3. Egress Attempt| RuleOut
    RuleOut -.-x|Drop at Perimeter| Drop

    %% High-Contrast Theme Styles
    style Internet fill:#f1f5f9,stroke:#475569,stroke-width:2px,color:#0f172a
    style Admin fill:#e0f2fe,stroke:#0284c7,stroke-width:2px,color:#0369a1
    style VNET fill:#ffffff,stroke:#0284c7,stroke-width:2px,stroke-dasharray: 4 4,color:#0369a1
    style SUBNET fill:#f8fafc,stroke:#64748b,stroke-width:1.5px,stroke-dasharray: 2 2,color:#334155
    style NSG fill:#f1f5f9,stroke:#0f172a,stroke-width:2px,color:#0f172a
    style RuleIn fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d
    style VM fill:#ffffff,stroke:#0284c7,stroke-width:2px,color:#0f172a
    style TPOT fill:#f8fafc,stroke:#7c3aed,stroke-width:2px,color:#5b21b6
    style Auth fill:#fef3c7,stroke:#d97706,stroke-width:1.5px,color:#92400e
    style UI fill:#ede9fe,stroke:#7c3aed,stroke-width:1.5px,color:#5b21b6
    style Traps fill:#fee2e2,stroke:#ef4444,stroke-width:1.5px,color:#991b1b
    style RuleOut fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#7f1d1d
    style Drop fill:#fee2e2,stroke:#ef4444,stroke-width:2px,stroke-dasharray: 4 4,color:#991b1b

    %% Colored Links
    linkStyle 0,1 stroke:#dc2626,stroke-width:2px
    linkStyle 2,3 stroke:#16a34a,stroke-width:2px
    linkStyle 4 stroke:#7c3aed,stroke-width:2px
    linkStyle 5,6 stroke:#b91c1c,stroke-width:2px
```

- **VNet / Subnet:** A dedicated virtual network and subnet, used exclusively for this honeypot — not peered or connected to any other network, home lab, or production resources.
- **NSG (Network Security Group):** Configured to isolate the honeypot at the network layer:
  - No outbound rules permitting traffic into other subnets, VNets, or peered networks (prevents horizontal/lateral movement).
  - No rules allowing the honeypot to reach management or admin planes (prevents vertical/privilege escalation paths).
  - Inbound traffic from the internet is allowed broadly (by design, to attract attackers), while management access (SSH) is the only channel used by the operator.
- **VM:** Runs T-Pot, which fronts a suite of honeypot daemons (e.g., Cowrie, Dionaea, Suricata, and others bundled with T-Pot) behind its own internal Docker network, plus an ELK-based dashboard (Kibana) for visualizing captured activity.

## Isolation & Safety Design

The core design principle: **assume the honeypot VM will be compromised, and make sure that doesn't matter.**

- The VNet/subnet exists solely for this VM — no shared resources, no routes to other environments.
- NSG rules are scoped to prevent the honeypot from initiating connections outward to anything other than what's required for its own operation (e.g., OS/T-Pot updates), blocking any attempt to use it as a pivot point.
- SSH access is retained only for administration and is treated as a sensitive, monitored channel.
- No credentials, keys, or data of value are stored on the VM beyond what T-Pot itself requires.

## Data Collection

Data collection began as soon as the VM was deployed and exposed. T-Pot captures:

- Connection attempts and port scans across exposed services
- Interaction logs from individual honeypot daemons (login attempts, commands run, files dropped)
- Network-layer intrusion detection alerts (Suricata)
- Aggregated views and dashboards via T-Pot's built-in Kibana interface

## Objectives / What This Demonstrates

- Practical Azure networking: VNets, subnets, and NSGs used for security segmentation, not just connectivity
- Understanding of lateral movement and privilege escalation risks, and how to design against them at the network layer
- Hands-on experience deploying and operating a honeypot platform (T-Pot)
- Analysis of real-world attacker behavior using captured telemetry

## Findings / Observations

**Top source countries (by attacker IP volume):**
1. Pakistan
2. Bulgaria
3. Netherlands

**Most-targeted port:** Telnet

*(To be expanded as more data accumulates — e.g., common exploit signatures, notable payloads, credential patterns.)*

## Lessons Learned

*(To be filled in — e.g., NSG rule surprises, T-Pot resource usage, volume of noise vs. signal.)*

## Future Improvements

- Forward T-Pot data into Microsoft Sentinel for centralized detection/alerting
- Add automated alerting on specific attack patterns
- Compare attacker behavior across multiple honeypot flavors/regions

## Disclaimer

This honeypot is deployed in an isolated environment for educational and research purposes only. It is not connected to any production systems.
