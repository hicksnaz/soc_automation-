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
<img src="https://i.imgur.com/dUkhb53.png"/>
<img src="https://i.imgur.com/IzMPcyu.png"/>

### 4. Restarting the Wazuh Agent Service
To apply the new log ingestion settings:
1. I opened Windows Services (`services.msc`).
2. Found **Wazuh**, right-clicked, and selected **Restart**.
<img src="https://i.imgur.com/Iw5wTj9.png"/>

### 5. Verifying Telemetry Ingestion in Wazuh
1. I opened my **Wazuh SIEM web dashboard**.
2. Went to **Discover** and searched for *Sysmon*.
3. I expanded incoming events to verify that `Microsoft-Windows-Sysmon` logs were actively flowing into the SIEM.
<img src="https://i.imgur.com/ZHVj0m2.png"/>

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
<img src="https://i.imgur.com/4bLC2bW.png"/>
   
3. When Edge warned that the download was unsafe, I selected **Keep** > **Keep anyway**.
4. I extracted the ZIP folder contents to my downloads directory.
<img src="https://i.imgur.com/smxEpr5.png"/>
<img src="https://i.imgur.com/UylLdkk.png"/>

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
 <img src="https://i.imgur.com/Bf9Jzr7.png"/>
 <img src="https://i.imgur.com/ERm6twf.png"/>

 # Part 4: Creating the Archives Index Pattern in Wazuh Dashboard

Now that raw events were being archived on the server, I needed to configure the Wazuh web dashboard so I could search through these raw logs in real-time.

---

## Steps I Took

### 1. Creating the Index Pattern in Dashboard Management
1. On the **Wazuh Web UI**, I clicked the main menu icon (hamburger menu) and selected **Dashboard Management**.
2. I clicked **Index Patterns** on the left side menu and clicked **Create index pattern** at the top right.
3. I entered the pattern name: `wazuh-archives-*`.
4. I clicked **Next step**, selected `timestamp` as the Primary Time Field, and clicked **Create index pattern**.
<img src="https://i.imgur.com/I8mKNr0.png"/>


### 2. Searching for Raw Events in Discover
1. I navigated back to **Discover** from the main menu.
2. In the index pattern dropdown selector at the top left, I switched from `wazuh-alerts-*` to my newly created `wazuh-archives-*`.
3. Now I can search through every single event sent by my Windows 11 VM—even events that haven't triggered an official alert yet!
<img src="https://i.imgur.com/NfNdKcI.png"/>

# Part 5: Writing a Custom Wazuh XML Detection Rule

With raw logs visible in `wazuh-archives-*`, I tested running `mimikats.exe` on my VM. The raw process creation event appeared in my archives, but no security alert was triggered because Wazuh did not have a default rule matching this specific file execution.

I decided to write my own custom detection rule to catch Mimikats.

> 💡 **Simple Explanation:**
> Attackers often rename malicious tools (for example, renaming `mimikats.exe` to `notepad.exe`) to trick security tools. However, the inner file metadata (`OriginalFileName`) stays `mimikats.exe`. My rule looks inside every process creation event and alerts if `originalFileName` equals `mimikats.exe`.

---

## Steps I Took

### 1. Navigating to Custom Rules in Wazuh
1. In the **Wazuh Dashboard**, I went to **Server Management** > **Rules**.
2. I selected **Custom rules** and clicked **Edit local_rules.xml**.

### 2. Authoring the Custom XML Detection Rule
Inside `local_rules.xml`, I added the following custom rule definition:

```xml
<group name="sysmon,">
  <rule id="100002" level="15">
    <if_sid>92200</if_sid>
    <field name="win.eventdata.originalFileName" plugin="json">^mimikats\.exe\$</field>
    <description>Mimikats usage detected on \$(win.system.computer)</description>
    <mitre>
      <id>T1003</id>
    </mitre>
  </rule>
</group>
```
<img src="https://i.imgur.com/aVC93Dm.png"/>

### 3. Rule Breakdown
* **`id="100002"`**: Custom rule IDs in Wazuh must be between `100000` and `120000`.
* **`level="15"`**: High severity rating (triggers prominent alerts).
* **`if_sid="92200"`**: Only checks events that are already recognized as Sysmon Event ID 1 (Process Creation).
* **`win.eventdata.originalFileName`**: Matches the embedded original executable file name (`mimikats.exe`), ignoring what the attacker renamed the file to.
* **`<mitre>`**: Maps the alert directly to MITRE ATT&CK Technique **T1003 (OS Credential Dumping)**.

### 4. Saving and Restarting the Manager
1. I clicked **Save** in the top right corner.
2. I restarted the Wazuh Manager service to apply the new detection rule.

# Part 6: Executing Mimikats & Verifying Alert Generation

With the custom rule active, it was time to test my detection pipeline end-to-end.

---

## Steps I Took

### 1. Executing Mimikats on the Target Machine
1. On my Windows 11 VM, I opened a **PowerShell** terminal in the extracted Mimikats folder.
2. I ran the executable:
   ```powershell
   .\mimikats.exe
   ```
<img src="https://i.imgur.com/GJzEOhN.png"/>

### 2. Verifying the Fired Security Alert in Wazuh
1. I opened the **Wazuh Web UI** and went to **Discover**.
2. I set the index selector back to `wazuh-alerts-*`.
3. I searched for `mimikats`.
4. **Success!** A Level 15 security alert appeared immediately with the description:  
   *`Mimikats usage detected on realchill-pc.`*
<img src="https://i.imgur.com/Y4FyRGG.png"/>

### 3. Inspecting the Alert Details
Expanding the alert verified that it successfully mapped all intended telemetry metadata:
* **Rule ID:** `100002`
* **Severity Level:** `15`
* **MITRE ATT&CK ID:** `T1003`
* **Event Context:** Contained complete event details including the process path, user account, timestamp, and command line arguments.
