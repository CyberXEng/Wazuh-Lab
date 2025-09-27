## Wazuh Lab [File Integrity, Blocking Malicious Actor]

> Proof‑of‑Concept for:
>
> 1. Installing a **Wazuh manager** (single‑node)
> 2. File Integrity Monitoring (FIM) on **Windows** and **Kali** agents
> 3. Detecting & **blocking a malicious actor** (Kali → CentOS web server) using Wazuh active response / IP reputation

**References:**

* File integrity PoC: [https://documentation.wazuh.com/current/proof-of-concept-guide/poc-file-integrity-monitoring.html](https://documentation.wazuh.com/current/proof-of-concept-guide/poc-file-integrity-monitoring.html)
* Block malicious actor PoC: [https://documentation.wazuh.com/current/proof-of-concept-guide/block-malicious-actor-ip-reputation.html](https://documentation.wazuh.com/current/proof-of-concept-guide/block-malicious-actor-ip-reputation.html)

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

* Host for Wazuh manager (CentOS/Ubuntu/Debian — adjust commands to your distro).
* CentOS server running a web service (target machine).
* Kali Linux machine (attacker) and one Windows host (agents).
* Network connectivity: agents → manager on the required ports.
* Root / admin privileges on all machines.
* PoC documentation links are available (see References).

---

## Install Wazuh manager (single-node)

> **Note:** package names and repository commands evolve. Use the official Wazuh docs for exact repo URLs and packages for your distro/version. The examples below demonstrate the conceptual steps.

### 1. Update & prerequisites

**Debian / Ubuntu**

```bash
sudo apt update
sudo apt install -y curl apt-transport-https gnupg
```

**CentOS / RHEL**

```bash
sudo yum update -y
sudo yum install -y curl
```

### 2. Add Wazuh repo & install (example)

```bash
# Example (replace with current Wazuh repo commands from official docs)
curl -sSL https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo apt install -y wazuh-manager wazuh-api
```

### 3. Start & enable services

```bash
sudo systemctl enable --now wazuh-manager
sudo systemctl status wazuh-manager

sudo systemctl enable --now wazuh-api
sudo systemctl status wazuh-api
```

> Optional: install Elasticsearch/OpenSearch + Wazuh dashboard if you want visualization. For PoC, API/logs are sufficient.

---

## Register / deploy agents (Kali & Windows)

### Create agent entry on the manager

```bash
sudo /var/ossec/bin/manage_agents
```

* Use `A` to add an agent — give a name (e.g., `kali-attacker`, `windows-host`).
* Use `E` to extract the key for enrollment.

Non-interactive example:

```bash
sudo /var/ossec/bin/manage_agents -a -n kali-attacker
sudo /var/ossec/bin/manage_agents -e <agent-id> > /tmp/kali.key
```

### Install agent on Kali (Linux)

```bash
sudo apt update
# install wazuh-agent per official repo instructions
sudo apt install -y wazuh-agent

# enroll agent to manager (example)
sudo /var/ossec/bin/agent-auth -m <MANAGER_IP> -p 1515 -A kali-attacker

sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent
```

### Install agent on Windows

* Download Wazuh agent MSI from official Wazuh downloads and run installer, or install silently:

```powershell
msiexec.exe /i wazuh-agent-4.x.msi /qn WAZUH_MANAGER="<MANAGER_IP>" WAZUH_AGENT_NAME="windows-host"
```

* Start agent service:

```powershell
Start-Service WazuhSvc
Get-Service WazuhSvc
```

> Verify agent connection on manager:

```bash
sudo /var/ossec/bin/agent_control -l
```

---

## File Integrity Monitoring (FIM / syscheck)

FIM is handled by Wazuh's `syscheck` module. Edit agent `ossec.conf` to configure directories and exclusions.

### Linux agent example (`/var/ossec/etc/ossec.conf`)

```xml
<syscheck>
  <!-- Frequency in seconds (3600 = 1 hour) -->
  <frequency>3600</frequency>

  <!-- Directories to monitor (comma-separated) -->
  <directories check_all="yes">/etc,/usr/bin,/var/www</directories>

  <!-- Exclude patterns -->
  <ignore>/var/www/tmp/*</ignore>
</syscheck>
```

### Windows agent example (`C:\Program Files\ossec-agent\ossec.conf`)

```xml
<syscheck>
  <frequency>3600</frequency>
  <!-- Windows paths; separate appropriately (semicolon or comma) -->
  <directories>c:\windows;c:\program files;c:\inetpub\wwwroot</directories>
  <ignore>c:\windows\temp\*;c:\program files\*\temp\*</ignore>
</syscheck>
```

**Notes**

* `check_all="yes"` enables hashing.
* Use `<ignore>` to skip volatile or large directories (e.g., `/proc`, `/sys`).
* Ensure the agent service account can read files you monitor (Windows ACLs).

### Force an immediate syscheck

```bash
sudo systemctl restart wazuh-agent
```

### Verify FIM alerts (manager)

```bash
sudo tail -n 200 /var/ossec/logs/alerts/alerts.log
# or
sudo tail -n 200 /var/ossec/logs/alerts/alerts.json
```

Search for `syscheck` or the rule IDs that relate to file-integrity alerts.

---

## Blocking a known malicious actor (active response)

Goal: detect attacker IP from alerts and block it on the CentOS web server using active response (iptables/firewalld or hosts.deny).

### 1) Enable active-response in manager (`/var/ossec/etc/ossec.conf`)

```xml
<active-response>
  <disabled>no</disabled>

  <!-- Example commands (confirm availability in your install): -->
  <command>host-deny</command>
  <command>firewalld-drop</command>
</active-response>
```

### 2) Ensure active-response scripts exist on the agent

On CentOS agent:

```bash
ls -l /var/ossec/active-response/bin/
# Expect scripts like host-deny, iptables, firewalld-drop
```

If using `firewalld`:

```bash
sudo systemctl enable --now firewalld
```

### 3) Create a detection rule that triggers the active response

Create `/var/ossec/etc/rules/local_rules.xml` on the manager:

```xml
<group name="local,">
  <rule id="100100" level="10">
    <!-- Replace with actual matching logic: use <if_matched_sid> or <match> to tie to the alert -->
    <description>Suspicious connection pattern: potential attacker — block IP</description>
    <options>no_full_log</options>

    <active-response>
      <command>firewalld-drop</command>
      <timeout>3600</timeout>
    </active-response>
  </rule>
</group>
```

**Important:** Replace matching conditions with the exact SID or regex that identifies the malicious activity (e.g., multiple 404s, ModSecurity alerts, payload signatures). Use the PoC documentation for exact detection patterns.

### 4) Test the full flow

1. From Kali, simulate the malicious behavior (repeated requests or test exploit) against CentOS web server.
2. Verify manager receives alerts and the custom rule matched.
3. Check active response logs:

```bash
sudo tail -n 200 /var/ossec/logs/active-responses.log
```

4. Verify IP is blocked on CentOS:

```bash
# iptables
sudo iptables -L -n | grep <ATTACKER_IP>

# firewalld
sudo firewall-cmd --list-rich-rules | grep <ATTACKER_IP>

# hosts.deny
grep <ATTACKER_IP> /etc/hosts.deny
```

---

## Troubleshooting & quick checks

* List agents (manager):

```bash
sudo /var/ossec/bin/agent_control -l
```

* Recent alerts:

```bash
sudo tail -n 200 /var/ossec/logs/alerts/alerts.log
```

* Active response logs:

```bash
sudo tail -n 200 /var/ossec/logs/active-responses.log
```

* Active-response scripts:

```bash
ls -l /var/ossec/active-response/bin/
```

* Restart agent (Linux):

```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

* Windows agent service:

```powershell
Get-Service WazuhSvc
```

---

## Notes, security & recommendations

* **Caution:** Active responses can block legitimate traffic. Tune thresholds and maintain whitelists. Test thoroughly.
* Secure manager ↔ agent communications (TLS, proper auth) for production.
* Avoid hashing very large or volatile directories—use `<ignore>` to reduce noise.
* Audit any active-response scripts before enabling them.
* Keep PoC and production environments separate.

---

## Useful commands summary

```bash
# Manager: list agents
sudo /var/ossec/bin/agent_control -l

# Manager: check alerts
sudo tail -n 200 /var/ossec/logs/alerts/alerts.log

# Manager: active responses log
sudo tail -n 200 /var/ossec/logs/active-responses.log

# Agent (Linux): restart
sudo systemctl restart wazuh-agent

 CentOS: check iptables/firewalld for blocked IP
sudo iptables -L -n | grep <IP>
sudo #firewall-cmd --list-rich-rules | grep <IP>

# Check active-response scripts
ls -l /var/ossec/active-response/bin/
```

---
