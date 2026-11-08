# Evidence

Screenshots and exports that prove the pipeline works. Add the files below using exactly these names, then uncomment
the **Results** section in the main [README](../README.md).

| File | What it should show |
| :--- | :--- |
| `01-wazuh-agent-active.png` | Wazuh dashboard with the Windows agent **Active** |
| `02-sysmon-events-in-wazuh.png` | Sysmon events in Discover (`wazuh-archives-*`) |
| `03-mimikatz-alert-level15.png` | The level 15 "Mimikatz Usage Detected" alert |
| `04-shuffle-workflow.png` | The full workflow canvas (webhook, regex, VirusTotal, TheHive, email) |
| `05-shuffle-run-success.png` | A successful run with 200/201 status codes visible |
| `06-thehive-alert.png` | The alert created in TheHive |
| `07-analyst-email.png` | The notification email (blur personal addresses) |
| `shuffle-workflow.json` | Exported workflow (remove API keys and webhook URIs first) |

**Before adding any image:** blur or crop public IPs, API keys, webhook URIs, email addresses and hostnames.
