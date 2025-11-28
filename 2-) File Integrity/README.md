## Wazuh Lab [File Integrity]

## File Integrity Monitoring (FIM / syscheck)

![File Intergrity](Images/File%20Intergrity.png)<br/>

FIM is handled by Wazuh's `syscheck` module. Edit agent `ossec.conf` to configure directories and exclusions.

### Linux agent example (`/var/ossec/etc/ossec.conf`)

```xml
<syscheck>
  <!-- Frequency in seconds (60 = 1 min) -->
  <frequency>60</frequency>

  <!-- Directories to monitor (comma-separated) -->
  <directories check_all="yes">/etc,/usr/bin,/var/www</directories>

  <!-- Exclude patterns -->
  <ignore>/var/www/tmp/*</ignore>
</syscheck>
```

### Windows agent example (`C:\Program Files\ossec-agent\ossec.conf`)

```xml
<syscheck>
  <frequency>60</frequency>
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

```

---
