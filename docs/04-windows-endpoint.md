[Back to README](../README.md)

# 04. Windows 11 Endpoint (VirtualBox, Sysmon, Wazuh Agent)

This VM is the "victim": it generates the telemetry the rest of the pipeline reacts to.

> [!CAUTION]
> **Lab safety.** This guide deliberately weakens Windows Defender and runs credential-theft tooling.
> - Do it **only** on a disposable VM that holds no personal data and no real credentials.
> - Take a VirtualBox snapshot before you start (`Machine > Take Snapshot`) so you can roll back.
> - Do not reuse any password from your real accounts on this VM.
> - Use these techniques only on systems you own.

## Part 1: Create the VM

**Prerequisites**
- VirtualBox 7.0 or newer (built-in TPM 2.0 and Secure Boot support, required by Windows 11).
- The official Windows 11 ISO from Microsoft.

**Steps**
1. In VirtualBox click **New**.
2. **Name and OS:** name it (for example `Windows-11-Endpoint`), pick the ISO, and tick **Skip Unattended Installation**
   so you can complete setup manually.
3. **Hardware:** at least 4096 MB RAM (8192 MB recommended), at least 2 CPUs, and **Enable EFI** ticked.
4. **Disk:** at least 64 GB (80 GB recommended).
5. Before first boot: **Settings > Display** and raise Video Memory to the maximum (128 MB).
6. Start the VM. When it says *Press any key to boot from CD or DVD*, press a key immediately.

## Part 2: Create a local account (skip the Microsoft account)

1. Go through Language, Region and Keyboard screens.
2. At the network or sign-in screen press **Shift + F10** (or **Fn + Shift + F10** on some laptops) to open a console.
3. Run:
   ```cmd
   start ms-cxh:localonly
   ```
4. The "Who is going to use this PC?" window appears. Create a local account and continue.

> [!NOTE]
> Microsoft changes the out-of-box experience regularly. If this command stops working on a newer build, search for
> the current local-account workaround for your Windows 11 version.

## Part 3: Install Sysmon

Sysmon (Microsoft Sysinternals) writes detailed process, network and file events to the Windows event log.

1. In the VM, download Sysmon from the
   [Sysinternals page](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) and extract it to
   `C:\Tools\Sysmon`.
2. Get a configuration file. Running Sysmon without one is very noisy, so use the community
   [sysmon-modular](https://github.com/olafhartong/sysmon-modular) config: download its `sysmonconfig.xml`
   (open the file on GitHub, click **Raw**, save as) into the same folder.
3. Open **Command Prompt as Administrator** and run:
   ```cmd
   cd C:\Tools\Sysmon
   sysmon64.exe -accepteula -i sysmonconfig.xml
   ```
   You should see `Sysmon installed.` and `Sysmon started.`
4. Verify: `Win + R` > `eventvwr.msc` > **Applications and Services Logs > Microsoft > Windows > Sysmon > Operational**.
   Events (for example Event ID 1, process creation) should be appearing.

## Part 4: Install and enroll the Wazuh agent

The agent is what ships Sysmon events to the Wazuh manager from [02](02-wazuh-server.md).

1. In the Wazuh dashboard open **Endpoints > Deploy new agent** (menu names vary slightly between 4.x releases).
2. Choose **Windows**, enter the manager address `<WAZUH_PUBLIC_IP>`, and give the agent a name.
3. Copy the PowerShell command the dashboard generates and run it in an **Administrator** PowerShell on the VM.
   It has this shape (the installer URL and version come from the dashboard, so always copy it from there):
   ```powershell
   Invoke-WebRequest -Uri <MSI_URL_FROM_DASHBOARD> -OutFile $env:TEMP\wazuh-agent.msi
   msiexec.exe /i $env:TEMP\wazuh-agent.msi /q WAZUH_MANAGER='<WAZUH_PUBLIC_IP>' WAZUH_AGENT_NAME='win11-endpoint'
   NET START WazuhSvc
   ```
4. Confirm in the dashboard that the agent shows as **Active**. If it does not, check that your endpoint's public IP is
   allowed on ports 1514 and 1515 in the Wazuh firewall ([01](01-cloud-infrastructure.md)).

## Part 5: Send Sysmon logs to Wazuh

By default the agent does not read the Sysmon channel. Tell it to.

1. Open Notepad **as Administrator** and edit:
   `C:\Program Files (x86)\ossec-agent\ossec.conf`
2. Next to the other `<localfile>` blocks add:
   ```xml
   <localfile>
     <location>Microsoft-Windows-Sysmon/Operational</location>
     <log_format>eventchannel</log_format>
   </localfile>
   ```
3. Restart the agent:
   ```powershell
   Restart-Service -Name WazuhSvc
   ```

## Part 6: Prepare for the Mimikatz test

Mimikatz is a well-known credential-dumping tool and Defender will delete it on sight. To run it as a **detection
test** you need a scoped exception.

1. Create a dedicated folder, for example `C:\LabTools`.
2. Open **Windows Security > Virus & threat protection > Manage settings > Exclusions > Add or remove exclusions**,
   and exclude **only** `C:\LabTools`.
3. Prefer this folder exclusion over turning real-time protection off. If you must disable real-time protection,
   remember it re-enables itself after a reboot, and a Group Policy change makes it permanent for the lab VM only.
4. Download the release archive from the
   [Mimikatz releases page](https://github.com/gentilkiwi/mimikatz/releases) into `C:\LabTools` and extract it there.
5. Run it from an Administrator Command Prompt to confirm it launches, then exit:
   ```cmd
   cd C:\LabTools\mimikatz_trunk\x64
   mimikatz.exe
   ```
   At the `mimikatz #` prompt, `privilege::debug` confirms it is working. Type `exit` when done.

Next: [05. Telemetry review and custom detection rule](05-custom-detection-rules.md).
