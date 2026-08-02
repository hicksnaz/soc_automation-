# 🛡️ Modern SOC Automation Project - Part 3: Sysmon Ingestion, Log Archiving & Custom Threat Detection Rules

Welcome to Part 3 of my SOC Automation Project! In Part 1 and Part 2, I established the core infrastructure: setting up a Windows 11 target virtual machine (VM) with Sysmon, deploying a Wazuh SIEM server in the cloud, and configuring The Hive incident response platform.

In this phase, I focused on turning my raw data into actionable security alerts. I configured my Windows 11 Wazuh Agent to collect Sysmon logs, enabled full raw log archiving on my Wazuh Manager, created an index pattern to inspect raw events, authored a custom Wazuh XML detection rule to catch credential-dumping tools (Mimikats), and verified that the alert triggers successfully in my SIEM.

---

## 📋 Table of Contents
- [Part 1: Ingesting Sysmon Telemetry into the Wazuh Agent](#part-1-ingesting-sysmon-telemetry-into-the-wazuh-agent)
- [Part 2: Preparing the Target Environment \& Downloading Mimikats](#part-2-preparing-the-target-environment--downloading-mimikats)
- [Part 3: Enabling Full Log Archiving on the Wazuh Manager](#part-3-enabling-full-log-archiving-on-the-wazuh-manager)
- [Part 4: Creating the Archives Index Pattern in Wazuh Dashboard](#part-4-creating-the-archives-index-pattern-in-wazuh-dashboard)
- [Part 5: Writing a Custom Wazuh XML Detection Rule](#part-5-writing-a-custom-wazuh-xml-detection-rule)
- [Part 6: Executing Mimikats \& Verifying Alert Generation](#part-6-executing-mimikats--verifying-alert-generation)

# Part 1: Ingesting Sysmon Telemetry into the Wazuh Agent

By default, the Wazuh Windows agent collects basic Windows Event logs (like Application and System logs). To catch stealthy attack techniques, I needed my agent to send deep system logs from Sysmon (System Monitor) to my central Wazuh SIEM.

---

## Steps I Took

### 1. Backing Up the Configuration File
Before making changes, I made a safety backup copy of the agent configuration file on my Windows 11 VM:
* **File location:** `C:\Program Files (x86)\ossec-agent\ossec.conf`
* **Action:** I copied `ossec.conf` and pasted a duplicate (`ossec - Copy.conf`) in the same folder. If anything goes wrong, I can easily revert back.
<img src="https://i.imgur.com/rpOvasL.png"/>

### 2. Finding the Exact Sysmon Log Channel Name
1. I opened **Event Viewer** on my Windows 11 VM.
2. I navigated to: `Applications and Services Logs` > `Microsoft` > `Windows` > `Sysmon` > `Operational`.
3. I right-clicked **Operational**, clicked **Properties**, and copied the exact full channel path:
   ```text
   Microsoft-Windows-Sysmon/Operational
   ```

### 3. Updating the Agent Configuration (ossec.conf)
1. I opened **Notepad** as Administrator.
2. I opened `C:\Program Files (x86)\ossec-agent\ossec.conf` (selecting *All Files* in the file picker).
3. Under the `<localfile>` section where Windows logs are specified, I replaced the default Application event log block with my Sysmon channel path:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```
<img src="https://i.imgur.com/hEskC14.png"/>
<img src="https://i.imgur.com/J6NU61B.png"/>

### 4. Restarting the Wazuh Agent Service
To apply the new log ingestion settings:
1. I opened Windows Services (`services.msc`).
2. Found **Wazuh**, right-clicked, and selected **Restart**.
<img src="https://i.imgur.com/OEfnRRI.png"/>

### 5. Verifying Telemetry Ingestion in Wazuh
1. I opened my **Wazuh SIEM web dashboard**.
2. Went to **Discover** and searched for *Sysmon*.
3. I expanded incoming events to verify that `Microsoft-Windows-Sysmon` logs were actively flowing into the SIEM.
<img src="https://i.imgur.com/fRlfXSW.png"/>

# Part 2: Preparing the Target Environment & Downloading Mimikats

To test if my SIEM detects real attacks, I needed to run a known threat tool: **Mimikats**. Mimikats is an open-source tool used by ethical hackers and cybercriminals to extract plain-text passwords, hashes, and PINs from computer memory.

---

## Steps I Took

### 1. Disabling Windows Defender Real-Time Protection
1. I opened **Windows Security** on my Windows 11 VM.
2. I went to **Virus & threat protection** > **Manage settings**.
3. I toggled **Real-time protection**, **Cloud-delivered protection**, **Automatic sample submission**, and **Tamper Protection** to **Off**.

### 2. Adding Folder Exclusions
To stop Defender from automatically deleting my test tool after downloading:
1. I scrolled down to **Exclusions** > **Add or remove exclusions**.
2. I added an exclusion for the entire `C:\` drive (or your dedicated test folder).

### 3. Downloading Mimikats
1. I opened **Microsoft Edge** and navigated to the official GentleKiwi Mimikats GitHub repository. (Link if you can't find it: `https://github.com/gentilkiwi/mimikatz/security`)
2. I downloaded the compiled binary package (`mimikats_trunk.zip`).
<img src="https://i.imgur.com/eSgPPyj.png"/>
   
3. When Edge warned that the download was unsafe, I selected **Keep** > **Keep anyway**.
4. I extracted the ZIP folder contents to my downloads directory.
<img src="https://i.imgur.com/RkL41kp.png"/>
<img src="https://i.imgur.com/DWkKwdA.png"/>

# Part 3: Enabling Full Log Archiving on the Wazuh Manager

By default, Wazuh processes incoming logs, checks them against active detection rules, and discards raw logs that do not trigger an alert. To build custom rules for new threats, I needed Wazuh to save all raw logs into an archive file.

---

## Steps I Took

### 1. Backing Up and Modifying the Manager Configuration (ossec.conf)
1. I SSH'd into my Wazuh Manager server from my terminal.
2. I created a backup copy of the configuration file:
   ```bash
   cp /var/ossec/etc/ossec.conf ~ossec_backup.conf
   ```
3. I opened the file with nano:
   ```bash
   sudo nano /var/ossec/etc/ossec.conf
   ```
4. I searched for the `<global>` XML section and set both `logall` and `logall_json` to `yes`:
   ```xml
   <global>
     <jsonout_output>yes</jsonout_output>
     <alerts_log>yes</alerts_log>
     <logall>yes</logall>
     <logall_json>yes</logall_json>
   </global>
   ```
<img src="https://i.imgur.com/Tdz8ko3.png"/>
   
5. I saved the file (`Ctrl + X`, `Y`, `Enter`) and restarted the Wazuh Manager:
   ```bash
   systemctl restart wazuh-manager.service
   ```
<img src="https://i.imgur.com/WfEYQKt.png"/>

### 2. Updating Filebeat Settings (filebeat.yml)
To allow Filebeat (the data shipper) to send archived logs to the dashboard index, I edited Filebeat's config file:
1. I opened the file with nano:
   ```bash
   sudo nano /etc/filebeat/filebeat.yml
   ```
2. Under the archives section, I changed `enabled: false` to `enabled: true`:
   ```yaml
   filebeat.modules:
     - module: wazuh
       archives:
         enabled: true
   ```
3. I saved the file and restarted Filebeat:
   ```bash
   sudo systemctl restart filebeat
   ```
