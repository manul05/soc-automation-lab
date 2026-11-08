[Back to README](../README.md)

# 05. Telemetry Review and Custom Detection Rule

This guide covers analyzing ingested Windows logs in the Wazuh Dashboard and translating raw telemetry into actionable, high-fidelity custom rules. This follows the "Reviewing Wazuh" and "Creating Custom Rules" chapters of the MyDFIR SOC Lab series (see credits in the README).

---

## Phase 1: Enabling Log Archiving for Sysmon Telemetry (Wazuh Server)

By default, Wazuh only stores and indexes logs that trigger an alert rule. To visualize *all* Sysmon telemetry (useful for threat hunting without pre-defined rules), you must instruct Wazuh to archive everything and configure Filebeat to parse these archives into the Indexer.

**1. Enable Archiving in Wazuh Manager**
Log into your Wazuh Manager (Ubuntu server) via SSH and edit the global configuration file:
```bash
sudo nano /var/ossec/etc/ossec.conf
```
Find the `<global>` section near the top and change the `<logall>` and `<logall_json>` tags from `no` to `yes`:
```xml
  <global>
    <logall>yes</logall>
    <logall_json>yes</logall_json>
    ...
```
Save and exit.

**2. Enable Archive Forwarding in Filebeat**
By default, Filebeat only reads the `alerts` log. You must configure it to also read the `archives` log.
Open the Filebeat configuration file:
```bash
sudo nano /etc/filebeat/filebeat.yml
```
Scroll down to the `filebeat.modules:` block for Wazuh, and under `archives:`, change `enabled: false` to `enabled: true`:
```yaml
filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
    archives:
      enabled: true
```
Save and exit.

**3. Restart Services**
Restart both the Wazuh Manager and Filebeat services so the new configs take effect:
```bash
sudo systemctl restart wazuh-manager
sudo systemctl restart filebeat
```

**4. Create the Archives Index Pattern in Dashboard**
- Open your Wazuh Dashboard in the web browser.
- Click the top-left menu icon and navigate to **Stack Management** > **Data Views** (or **Index Patterns**).
- Click **Create Data View**.
- Set the Index pattern name to: `wazuh-archives-*`
- Select `@timestamp` as the Time field and save.
- Finally, navigate to the **Discover** tab and switch your data view from `wazuh-alerts-*` to `wazuh-archives-*`. You will now see a constant stream of ALL raw Sysmon events flowing in!

---

## Phase 2: Reviewing Wazuh Telemetry

To write an effective custom rule, you first need to identify what the raw event looks like when it hits Wazuh.

### 1. Generate Test Telemetry
- On your Windows VM, perform a test action that you want to detect. A common SOC lab example is executing Mimikatz.
- Download a sample of Mimikatz (disable Windows Defender Real-Time Protection locally if needed).
- Open an administrative command prompt and run `mimikatz.exe`, then exit.

### 2. Locate the Event in Wazuh Dashboard
- Open the Wazuh Dashboard.
- If you enabled archiving (so all logs are stored), navigate to **Discover** and select the `wazuh-archives-*` data view.
- If you rely purely on default alerts, check the **Security events** module.
- Type `*mimikatz*` in the search bar.
- Find the newly generated log entry corresponding to your test (typically a Sysmon Event ID 1 - Process Creation event).

### 3. Analyze the JSON Payload
- Expand the document in the Discover tab and switch to the **JSON** view.
- Identify the exact field names and values you can use for a precise detection rule. Pay close attention to:
  - `data.win.eventdata.image` *(e.g., `C:\Users\Admin\Desktop\mimikatz.exe`)*
  - `data.win.eventdata.originalFileName` *(e.g., `mimikatz.exe`)* — This is highly reliable even if the attacker renames the executable.
  - `data.rule.id` — Note down the parent rule ID that originally captured this event (often `61603` or `92211` for Sysmon, or `80203` if coming via Windows Event logs).

---

## Phase 3: Creating a Custom Rule

Wazuh allows you to create custom rules in `local_rules.xml` to single out dangerous activities and raise their alert level (e.g., to level 15).

### 1. Access the Rules Editor
1. In the Wazuh Dashboard, expand the main menu (Wazuh logo toggle).
2. Navigate to **Management** > **Rules** > **Custom rules**.
3. Select `local_rules.xml` from the list.
4. Click the **Edit** (pencil) icon.

*(Alternatively, you can SSH into your Wazuh Manager and edit `/var/ossec/etc/rules/local_rules.xml` using a terminal text editor like `nano`)*.

### 2. Write the Custom XML Rule
Inside the initial `<group name="local,syslog,sshd,">` block, press enter to create space and add your custom rule. 

A standard custom rule for catching Mimikatz execution looks like this:

```xml
  <rule id="100002" level="15">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.originalFileName" type="pcre2">(?i)mimikatz\.exe</field>
    <description>Mimikatz Usage Detected</description>
    <mitre>
      <id>T1003</id>
    </mitre>
  </rule>
```
**Rule Breakdown**:
- `id="100002"`: All custom rules must have IDs starting at `100001` or higher to avoid conflicts with Wazuh's default rules.
- `level="15"`: The severity level (0-15). Level 15 is the highest priority (Critical).
- `<if_group>`: Tells Wazuh to only evaluate this rule IF the prerequisite parent rule group (in this case, `sysmon_event1` for general Sysmon process creation) triggers first.
- `<field>`: The specific condition logic. We use `win.eventdata.originalFileName` because it maps to `data.win.eventdata.originalFileName` in the JSON payload (leave off the `data.` prefix in rules).

### 3. Save and Restart
1. Click the **Save** button in the top right corner of the dashboard editor.
2. The Dashboard will prompt you with a notification that the Wazuh Manager requires a restart to apply the new rules. 
3. Click the **Restart** button in the prompt and wait for the service to finish restarting.

---

## Phase 4: Triggering and Verifying the Custom Rule

1. Return to your Windows Endpoint and re-execute your test action (`mimikatz.exe`).
2. Go back to the Wazuh Dashboard.
3. Navigate to the **Security events** module (or use the `wazuh-alerts-*` data view in Discover).
4. You should now see a **Level 15** alert titled "Mimikatz Usage Detected".
5. The finished rule is also stored in this repo at [`detections/local_rules.xml`](../detections/local_rules.xml).
6. Because this is now a structured, high-severity alert, you can comfortably configure Wazuh Integration blocks to forward this specific Rule ID directly to your SOAR platform (like Shuffle) for automated incident response!

---

## Phase 5: Validation Test Plan

A rule is only useful if you know when it does *not* fire. Run these after the rule is live and record the outcome
(tick the box and note the alert ID or a screenshot name from [`evidence/`](../evidence/README.md)).

| # | Test | Expected | Result |
| :-: | :--- | :--- | :-: |
| 1 | Run `mimikatz.exe` unmodified | Rule `100002` fires at level 15 | ☐ |
| 2 | Rename the binary (for example `notmimi.exe`) and run it | Still fires, because `originalFileName` is read from the PE header, not the file name | ☐ |
| 3 | Run an unrelated signed tool (for example `calc.exe`) | Rule does **not** fire (false-positive check) | ☐ |
| 4 | Stop the Sysmon service, run Mimikatz | No alert, which shows how blind the rule is if telemetry is tampered with | ☐ |

> [!NOTE]
> Test 4 is the honest limitation of every signature-style rule: it depends on the telemetry arriving.
> See the roadmap in [07](07-lessons-and-roadmap.md) for what would harden this.
