# DNS Sinkhole Lab — Threat Detection with Pi-hole & Wazuh SIEM

## Overview

This lab demonstrates the deployment of a DNS sinkhole using Pi-hole integrated with Wazuh SIEM in a virtualized home lab environment. The objective was to intercept and log DNS queries to known malicious and tracking domains, redirect them to a null address (0.0.0.0), and forward those events to a SIEM for centralized visibility.

This simulates a real-world defensive security control used by SOC teams and network defenders to detect compromised hosts, block C2 communication, and generate threat intelligence from DNS telemetry.

---

## Lab Environment

| Component | Details |
|---|---|
| Hypervisor | Proxmox VE 9.1.1 |
| DNS Sinkhole | Pi-hole v6.4.1 on Ubuntu 22.04 |
| SIEM | Wazuh v4.7.5 |
| Client VM | Windows 11 WIN11-CLIENT |
| Network | 192.168.1.0/24 |
| Pi-hole / Wazuh IP | 192.168.1.233 |
| Client IP | 192.168.1.189 |

---

## Tools Used

- Pi-hole — Network-wide DNS sinkhole and query logger
- Wazuh SIEM — Log ingestion, alerting, and security event management
- Proxmox — Type 1 hypervisor hosting all VMs
- StevenBlack Hosts List — Consolidated malware and tracking domain blocklist (92,277 domains)
- nslookup — DNS query tool used to simulate and verify sinkholing
- nano — Text editor used to configure Wazuh ossec.conf

---

## Objectives

- Deploy Pi-hole as a DNS sinkhole on an existing Ubuntu VM
- Load a threat intelligence blocklist containing known malicious domains
- Configure a Windows 11 client to use Pi-hole as its DNS resolver
- Simulate DNS queries to blocked domains and confirm sinkholing
- Forward Pi-hole logs to Wazuh for centralized SIEM monitoring
- Verify end-to-end detection chain from client query to SIEM ingestion

---

## MITRE ATT&CK Mapping

| Technique | ID | Description |
|---|---|---|
| Command and Control — Application Layer Protocol: DNS | T1071.004 | Adversaries use DNS to communicate with C2 infrastructure. The sinkhole intercepts these queries before they reach the attacker. |
| Exfiltration Over Alternative Protocol | T1048 | DNS-based exfiltration attempts are intercepted and logged by Pi-hole. |
| Indicator Removal — DNS Cache Poisoning Defense | T1565 | Sinkholing redirects malicious DNS responses to null, preventing resolution of attacker-controlled domains. |
| Network Traffic Content Inspection Detection | T1040 | Pi-hole logs provide full DNS query visibility, enabling detection of beaconing or suspicious lookup patterns. |
---

## Phase 1 — Pi-hole Installation

Pi-hole was installed directly on the Wazuh Ubuntu VM (192.168.1.233) using the official installer script.

Steps:
1. Updated system packages: sudo apt update && sudo apt upgrade -y
2. Resolved interrupted dpkg state: sudo dpkg --configure -a
3. Ran Pi-hole installer: curl -sSL https://install.pi-hole.net | bash
4. Selected ens18 as the network interface
5. Selected Cloudflare (DNSSEC) as the upstream DNS provider
6. Enabled query logging — Show everything privacy mode
7. Installation completed successfully

Result: Pi-hole installed and active at http://192.168.1.233/admin with 92,277 default blocklist domains loaded.

---

## Phase 2 — Blocklist Configuration

A consolidated threat intelligence blocklist was added to Pi-hole to expand coverage beyond the default list.

Blocklist added: https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts

Gravity update command: sudo pihole -g

Result: 92,277 unique domains confirmed in gravity database. Blocklist status: Enabled.

---

## Phase 3 — Client DNS Configuration

The Windows 11 client VM (192.168.1.189) was configured to use Pi-hole as its primary DNS resolver.

Settings applied:
- Preferred DNS: 192.168.1.233
- Alternate DNS: 8.8.8.8

Verification: DNS resolver confirmed as pi.hole via nslookup.
---

## Phase 4 — Sinkhole Verification

A DNS query was made from the Windows 11 client to a known blocked domain to confirm the sinkhole was functioning.

Command: nslookup doubleclick.net

Result: Server returned pi.hole at 192.168.1.233 and resolved doubleclick.net to 0.0.0.0 confirming the sinkhole intercepted the query and returned a null address instead of the real IP. The Pi-hole query log confirmed the block with client IP 192.168.1.189 recorded.

---

## Phase 5 — Wazuh Log Integration

Wazuh was configured to ingest Pi-hole DNS logs for centralized SIEM monitoring.

The following localfile block was added to /var/ossec/etc/ossec.conf to point Wazuh at the Pi-hole log file. Wazuh manager was restarted and the Pi-hole log was verified streaming in real time using sudo tail -f /var/log/pihole/pihole.log.

Result: Wazuh successfully ingesting Pi-hole dnsmasq log entries. DNS queries from the Win11 client visible in real-time log stream including cached, forwarded, and blocked domain lookups.

---

## Findings

- DNS sinkhole successfully intercepted queries to known malicious and tracking domains
- 92,277 domains blocked across ad, malware, and tracking categories
- Windows 11 client DNS traffic fully visible through Pi-hole including Microsoft telemetry domains, tracking domains, and the simulated doubleclick.net query
- Pi-hole query log confirmed blocked queries with client IP, timestamp, domain, and action
- Wazuh configured to monitor Pi-hole log file providing SIEM-level visibility into DNS activity across the lab network

## Screenshots

| Screenshot | Description |
|---|---|
| screenshot-wazuh-dashboard.png | Wazuh dashboard showing 2 active agents |
| screenshot-pihole-install-complete.png | Pi-hole installation complete screen |
| screenshot-pihole-dashboard-baseline.png | Pi-hole dashboard Active with 92,277 domains |
| screenshot-blocklist-added.png | StevenBlack blocklist added and Enabled |
| screenshot-gravity-update.png | Gravity update terminal showing 92,277 domains loaded |
| screenshot-win11-dns-settings.png | Win11 DNS configured to 192.168.1.233 |
| screenshot-nslookup-sinkholed.png | nslookup doubleclick.net returning 0.0.0.0 |
| screenshot-pihole-dashboard-active.png | Pi-hole dashboard showing blocked queries at 6.3% |
| screenshot-pihole-query-log.png | Query log showing blocked domains from Win11 client |
| screenshot-wazuh-ossec-config.png | ossec.conf with Pi-hole localfile block added |
| screenshot-pihole-log-streaming.png | Live Pi-hole log stream in terminal |

---

## Key Takeaways

- DNS sinkholes are a low-cost high-visibility defensive control that can be deployed quickly in any network environment
- Integrating Pi-hole with a SIEM like Wazuh creates a full detection pipeline from DNS query to security alert
- Even a default blocklist catches significant telemetry and tracking traffic demonstrating the volume of outbound DNS activity on a standard Windows endpoint
- This technique directly maps to real SOC workflows where DNS logs are a primary source of threat hunting data

---

## Author

Akel Swinton — IT Systems Administrator | Cybersecurity
