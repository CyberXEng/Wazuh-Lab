## Wazuh Lab

> Proof‑of‑Concept for:
>
> 1. Installing a **Wazuh manager** (single‑node)
> 2. File Integrity Monitoring (FIM) on **Windows** and **Kali** agents
> 3. Detecting & **blocking a malicious actor** (Kali → CentOS web server) using Wazuh active response / IP reputation
> 4. Suricata IDS integration on **Ubuntu**
* Full Wazuh Lab Video : [https://drive.google.com/drive/folders/1RGNhs04WM-VEydVWw4FEqJCxEQBE6jW-](https://drive.google.com/drive/folders/1RGNhs04WM-VEydVWw4FEqJCxEQBE6jW-)

**References:**
* Instalation: [https://documentation.wazuh.com/current/quickstart.html](https://documentation.wazuh.com/current/quickstart.html)
* File integrity: [https://documentation.wazuh.com/current/proof-of-concept-guide/poc-file-integrity-monitoring.html](https://documentation.wazuh.com/current/proof-of-concept-guide/poc-file-integrity-monitoring.html)
* Block malicious actor: [https://documentation.wazuh.com/current/proof-of-concept-guide/block-malicious-actor-ip-reputation.html](https://documentation.wazuh.com/current/proof-of-concept-guide/block-malicious-actor-ip-reputation.html)
* Suricata IDS integration: [https://documentation.wazuh.com/current/proof-of-concept-guide/integrate-network-ids-suricata.html](https://documentation.wazuh.com/current/proof-of-concept-guide/integrate-network-ids-suricata.html)

![Network Architecture](Images/Network%20Arch.png)<br/>

---

## Table of contents

* [Prerequisites](#prerequisites)
* [Install Wazuh manager (single-node)](#install-wazuh-manager-single-node)
* [Register / deploy agents (Kali & Windows)](#register--deploy-agents-kali--windows)
* [File Integrity Monitoring (FIM / syscheck)](#file-integrity-monitoring-fim--syscheck)
* [Blocking a known malicious actor (active response)](#blocking-a-known-malicious-actor-active-response)
* [Troubleshooting & quick checks](#troubleshooting--quick-checks)
* [Notes, security & recommendations](#notes-security--recommendations)
* [Useful commands summary](#useful-commands-summary)

---

## Prerequisites

* Host for Wazuh manager (CentOS/Ubuntu/Debian).
* CentOS server running a web service (target machine).
* Kali Linux machine (attacker) and one Windows host (agents).
* Network connectivity: agents → manager on the required ports.
* Root / admin privileges on all machines.
* PoC documentation links are available (see References).

---
