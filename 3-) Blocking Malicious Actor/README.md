## Wazuh Lab [Blocking Malicious Actor]

## Blocking a known malicious actor (active response)
![Events And Alerts](Images/Events%20And%20Alerts.png)<br/>
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
      <timeout>60</timeout>
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
