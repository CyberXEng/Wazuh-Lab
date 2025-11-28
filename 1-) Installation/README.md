## Wazuh Lab [Installation]
---

## Install Wazuh manager (single-node)

> **Note:** package names and repository commands evolve. Use the official Wazuh docs for exact repo URLs and packages for your distro/version. The examples below demonstrate the conceptual steps.
![Wazuh Groups](Images/Wazuh%20Groups.png)<br/>
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

```

---
