# Wazuh-Lab-Installation---File-Integrity---Blocking-Malicious-Actor-
A hands-on Wazuh lab covering installation, file integrity monitoring, and blocking malicious actors. Learn to deploy Wazuh, detect file changes, and configure active responses to mitigate threats. Ideal for SecOps, SIEM training, and security monitoring practice.
🛡️ Wazuh Lab — VMware-based: Installation, FIM & Blocking Malicious Actors

Lab goal: Build a 4‑device Wazuh lab on VMware to demonstrate installation, File Integrity Monitoring (FIM), and Active Response (blocking malicious actors).
Devices (VMs):

wazuh-server — Ubuntu (Wazuh Manager + Dashboard)

centos-web — CentOS (web server + Wazuh agent)

win-client — Windows 10/11 (normal client + agent)

kali-attacker — Kali Linux (attacker)

Table of Contents

Lab topology & network

Prerequisites

VMware VM setup (quick)

Wazuh Server (Ubuntu) — installation & config

Agents — CentOS & Windows — installation & config

File Integrity Monitoring (FIM) — config & tests

Blocking Malicious Actor (Active Response) — detection & blocking workflows

Test cases / walkthrough

Useful config snippets (ossec.conf, fim rules, active-response)

Troubleshooting & tips

Folder structure & next steps

License & safety

1. Lab topology & network
[ win-client ]      \
[ centos-web ] ----> [ wazuh-server (Ubuntu) ]  (Wazuh Manager + Dashboard)
[ kali-attacker ]  /


All VMs on a single isolated VMware network (NAT or host‑only) to avoid impacting production.

Suggested IPs (static or DHCP with reservations):

wazuh-server: 192.168.56.10

centos-web: 192.168.56.20

win-client: 192.168.56.30

kali-attacker: 192.168.56.40

2. Prerequisites

VMware Workstation / Fusion / ESXi with enough resources (each VM ≥ 1 CPU, 2–4 GB RAM).

ISOLATED virtual network (host-only or NAT) for safety.

Ubuntu Server 20.04/22.04 ISO (for wazuh-server).

CentOS 7/8 ISO (for centos-web).

Windows 10/11 ISO (for win-client).

Kali Linux ISO (for attacker).

Basic familiarity with Linux shell, Windows PowerShell, and VMware.

3. VMware VM setup (quick)

For each VM:

Create new VM, choose OS type, allocate resources (2 vCPU, 2–4 GB RAM, 20+ GB disk).

Attach VM to same host-only / isolated network.

Install OS, apply updates, install VMware Tools / open-vm-tools.

Set static IPs or DHCP with fixed lease (recommended static inside each OS).

Snapshot VMs after initial install — you’ll revert during iterative testing.

4. Wazuh Server (Ubuntu) — installation & config (concise steps)

These are minimal commands — expand/check versions for your environment.

Update & prerequisites

sudo apt update && sudo apt upgrade -y
sudo apt install curl apt-transport-https lsb-release gnupg -y


Follow Wazuh official repo install (example for manager + dashboard):

# Wazuh repository
curl -sO https://packages.wazuh.com/key/GPG-KEY-WAZUH
sudo apt-key add GPG-KEY-WAZUH
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update

# Install manager and API
sudo apt install wazuh-manager -y
# Optional: Wazuh API (if using older installs) or Wazuh indexer + Kibana/Opensearch dashboard
sudo apt install wazuh-indexer wazuh-dashboard -y


Start & enable services

sudo systemctl enable --now wazuh-manager
sudo systemctl status wazuh-manager


Open firewall ports (if using UFW/firewalld) for agent registration and dashboard:

Manager listens 1514/udp (logs) and 1515/tcp (registration) by default. Dashboard port depends on install (5601 for Kibana/Opensearch).

5. Agents — installation & registration
CentOS web server (agent)
# Add Wazuh repo for RHEL/CentOS and install agent
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo rpm --import -
cat <<EOF | sudo tee /etc/yum.repos.d/wazuh.repo
[wazuh]
name=Wazuh repository
baseurl=https://packages.wazuh.com/4.x/yum/
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
EOF

sudo yum install wazuh-agent -y
sudo systemctl enable --now wazuh-agent


Register agent with manager:

# on wazuh-server
sudo /var/ossec/bin/manage_agents -a

# on centos-web, configure manager IP in /var/ossec/etc/ossec.conf <server> or using manage_agents output:
sudo /var/ossec/bin/agent-auth -m 192.168.56.10

Windows client (agent)

Download Wazuh agent MSI on win-client (place in Downloads).

Install via GUI or PowerShell:

msiexec /i wazuh-agent-4.x.x.msi /qn WAZUH_MANAGER=192.168.56.10
Start-Service WazuhSvc


If agent needs activation key or registration, use agent-auth.exe shipped with agent:

C:\Program Files\ossec-agent\agent-auth.exe -m 192.168.56.10


After registration, confirm agents appear in Wazuh Dashboard or manage_agents -l.

6. File Integrity Monitoring (FIM)
Goals

Monitor critical files/directories on CentOS web server and Windows client:

CentOS: /var/www, /etc/httpd/conf, /etc/passwd, etc.

Windows: C:\Program Files, C:\Users\Public\ImportantFile.txt, C:\Windows\System32\drivers\etc\hosts

ossec.conf (agent-side) — FIM snippet

Place under <ossec_config> on agent's /var/ossec/etc/ossec.conf (CentOS) or Windows agent equivalent:

<syscheck>
  <disabled>no</disabled>
  <frequency>3600</frequency>
  <directories check_all="yes">/var/www,/etc/httpd,/etc</directories>
  <ignore>/var/www/cache</ignore>
  <realtime>yes</realtime>       <!-- uses inotify on Linux -->
</syscheck>


Windows ossec.conf example:

<syscheck>
  <disabled>no</disabled>
  <directories check_all="yes">C:\Windows\System32;C:\Program Files;C:\Users</directories>
  <realtime>yes</realtime>
</syscheck>


Restart agent after modification:

sudo systemctl restart wazuh-agent

Test FIM

Modify a monitored file on CentOS:

echo "malicious change" | sudo tee /var/www/index.html


On Windows, edit monitored file or create a new file in monitored path.

Verify alert in Wazuh Dashboard → Security Events / Alerts. Alerts will show syscheck changes (file modified/added/removed).

7. Blocking Malicious Actor (Active Response)
Overview

Use Wazuh rules + active response to trigger firewall blocks (iptables/firewalld on Linux, Windows Firewall on Windows) when attacks detected (e.g., SSH brute force, numerous failed logins, web exploitation).

Active response components

Rule — detect event (e.g., multiple failed SSH logins)

Active response script — executed by manager to block IP (e.g., firewalld/iptables command)

Agent or manager executed? — usually executed on manager controlling firewall there, or executed on agent host using active-response script (recommended: block on affected host).

Example: Block SSH brute-force on CentOS

Enable detection rule in Wazuh (default rules often include SSH brute force).

Active response: use host-deny or custom script to add an iptables rule on CentOS.

Example active-response script (simple):

# /var/ossec/active-response/bin/block-ip.sh (on centos-web)
#!/bin/bash
IP=$1
ACTION=$2  # add or delete
if [ "$ACTION" = "add" ]; then
  /sbin/iptables -I INPUT -s $IP -j DROP
elif [ "$ACTION" = "delete" ]; then
  /sbin/iptables -D INPUT -s $IP -j DROP
fi


Make executable:

sudo chmod +x /var/ossec/active-response/bin/block-ip.sh


Register the active response in ossec.conf on manager:

<active-response>
  <command>block-ip.sh</command>
  <location>local</location>
  <level>10</level>
  <timeout>600</timeout>
</active-response>


Restart Wazuh manager.

Windows blocking via PowerShell

Active response script to add Windows Firewall rule (run on win-client agent):

param([string]$IP, [string]$ACTION)

if ($ACTION -eq "add") {
  New-NetFirewallRule -DisplayName "Wazuh Block $IP" -Direction Inbound -Action Block -RemoteAddress $IP -Profile Any
} elseif ($ACTION -eq "delete") {
  Get-NetFirewallRule -DisplayName "Wazuh Block $IP" | Remove-NetFirewallRule
}


Place under agent active-response folder and register similarly.

8. Test cases / walkthrough
Test A — FIM alert

On centos-web: modify /var/www/index.html.

Wait for agent re-scan or trigger manual syscheck run.

Confirm alert in Wazuh Dashboard showing file change with rule id 550 series (syscheck).

Test B — SSH brute force & blocking

From kali-attacker:

# simple SSH fail loop (demo only — be ethical & isolated)
for i in {1..50}; do ssh -o ConnectTimeout=5 -o StrictHostKeyChecking=no wronguser@192.168.56.20 exit; done


Or use hydra to brute-force.
2. Wazuh manager detects multiple failed SSH attempts; rule fires.
3. Active Response script runs and blocks attacker's IP on centos-web.
4. Verify iptables rule exists:

sudo iptables -L -n | grep 192.168.56.40


On kali-attacker, confirm SSH is blocked.

Test C — Web file tamper & attacker block

From kali-attacker, upload a webshell to centos-web /var/www.

FIM detects file added/changed → alert.

Create a rule to trigger active response blocking source IP of upload (parse webserver logs to find source IP).

Verify block.

9. Useful config snippets
Example rule (detect repeated SSH failures) — add to rules/local_rules.xml
<group name="local,">
  <rule id="100100" level="10">
    <if_sid>5716</if_sid> <!-- example Wazuh ssh failed login SID — adjust to your ruleset -->
    <description>Custom: SSH brute force detected — trigger block</description>
    <options>no_full_log</options>
    <action>active-response</action>
    <command>block-ip.sh</command>
    <location>local</location>
  </rule>
</group>

Example syscheck (fim) advanced
<syscheck>
  <frequency>7200</frequency>
  <directories check_all="yes">/var/www;/etc/httpd;/etc/</directories>
  <auto_ignore>3600</auto_ignore>
  <ignore>/var/www/cache</ignore>
  <realtime>yes</realtime>
</syscheck>
