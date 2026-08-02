# ⚡ Modern SOC Automation Project - Part 4: Orchestration & Response with Shuffle SOAR, VirusTotal & The Hive

Welcome to the final chapter (Part 4) of my SOC Automation Project! In the previous parts of this series, I built the foundational security environment:

* **Parts 1 & 2:** Provisioned a cloud-hosted Wazuh SIEM server, set up a Windows 11 target VM with Sysmon telemetry, and deployed The Hive incident management system.
* **Part 3:** Configured custom log ingestion, turned on raw log archiving, and created custom XML detection rules to catch execution of credential-dumping tools like Mimikats.

In **Part 4**, I bring everything together into an automated **Security Orchestration, Automation, and Response (SOAR)** workflow using **Shuffle SOAR**. When an attack occurs, my SIEM automatically triggers a workflow that extracts threat indicators, checks them against threat intelligence (VirusTotal), opens an incident ticket (The Hive), and alerts the security team via email in seconds.

---

## 📋 Table of Contents
* [Part 1: Connecting Wazuh to Shuffle SOAR via Webhook](#part-1-connecting-wazuh-to-shuffle-soar-via-webhook)
* [Part 2: Extracting File Hashes with Regex in Shuffle](#part-2-extracting-file-hashes-with-regex-in-shuffle)
* [Part 3: Enriching Threat Intelligence with VirusTotal API](#part-3-enriching-threat-intelligence-with-virustotal-api)
* [Part 4: Automating Alert Ticket Creation in The Hive](#part-4-automating-alert-ticket-creation-in-the-hive)
* [Part 5: Sending Automated Email Alerts to SOC Analysts](#part-5-sending-automated-email-alerts-to-soc-analysts)
* [Part 6: End-to-End Workflow Verification & Testing](#part-6-end-to-end-workflow-verification--testing)

## Part 1: Connecting Wazuh to Shuffle SOAR via Webhook

To make my SIEM automatically talk to my SOAR platform, I needed a communication pipeline called a Webhook. Whenever Wazuh detects a Mimikats event, it sends the alert data directly to Shuffle using an HTTP request.

> 💡 **Simple Explanation:**
> Think of a Webhook like a door buzzer. Whenever a specific bad guy (Rule `100002`) walks into my SIEM house, Wazuh pushes the buzzer button (the Webhook URL), which wakes up Shuffle SOAR to start working immediately.

---

### Steps I Took

#### 1. Creating the Workflow & Webhook in Shuffle
1. I signed into my account at [shuffler.io](https://shuffler.io/).
2. I created a new workflow titled `"Naz-SOC-Auto-Project"`(whatever you choose).
3. I dragged a **Webhook** node onto the canvas and copied the generated Webhook URI.
 <img src="https://i.imgur.com/rQpvdNX.png"/>
 <img src="https://i.imgur.com/aIx1uMu.png"/>

#### 2. Configuring the Wazuh Integration (ossec.conf)
1. I SSH'd into my Wazuh Manager terminal.
2. I edited `/var/ossec/etc/ossec.conf` using nano:
   ```bash
   sudo nano /var/ossec/etc/ossec.conf
   ```
3. I added the `<integration>` XML block right below the `<global>` section:
   ```xml
   <integration>
     <name>shuffle</name>
     <hook_url>https://shuffler.io/api/v1/hooks/webhook_YOUR_UNIQUE_ID</hook_url>
     <rule_id>100002</rule_id>
     <alert_format>json</alert_format>
   </integration>
   ```
<img src="https://i.imgur.com/02jD87U.png"/>

##### Integration Tag Breakdown:
* **`<name>shuffle</name>`**: Tells Wazuh to use its built-in Shuffle integration script.
* **`<hook_url>`**: The exact web address Shuffle gave me.
* **`<rule_id>100002</rule_id>`**: Restricts this integration so it only fires when my custom Mimikats detection rule triggers.
* **`<alert_format>json</alert_format>`**: Formats the log data into JSON so computers can read it easily.

#### 3. Restarting Wazuh & Testing the Trigger
1. I saved the file and restarted the Wazuh service:
   ```bash
   sudo systemctl restart wazuh-manager
   ```
2. In Shuffle, I turned on the **Start / Listening** mode on the Webhook node.
3. On my Windows 11 target VM, I ran `.\mimikats.exe`.
4. I checked Shuffle's **Explore Runs** tab and confirmed that the raw JSON alert payload arrived successfully!
<img src="https://i.imgur.com/l3Bhuky.png"/>
<img src="https://i.imgur.com/ZKqMlXW.png"/>

## Part 2: Extracting File Hashes with Regex in Shuffle

Raw security logs are filled with hundreds of lines of text. To check if a file is malicious, I need its exact digital fingerprint—a 64-character string called a SHA256 hash. I used Regular Expressions (Regex) to filter out the noise and pull only the hash string.

> 💡 **Simple Explanation:**
> A Regex (Regular Expression) pattern is like a target search filter. Instead of reading an entire 500-word log by hand, I gave Shuffle a rule that says: *"Find and grab any sequence of 64 letters and numbers that looks like a file hash."*

---

### Steps I Took

#### 1. Adding Shuffle Tools to the Canvas
1. I dragged the **Shuffle Tools** node onto my workflow canvas and connected it to the Webhook node.
2. I renamed the node to `SHA256_Regex_Extract`.

#### 2. Configuring the Regex Pattern
1. I set the action to **Regex Capture Group**.
2. I entered the SHA256 extraction pattern:
   ```text
   SHA256=([0-9A-Fa-f]{64})
   ```
<img src="https://i.imgur.com/yeUFO98.png"/>
3. Attach Webhook to Shuffle, rename it as `SHA256-Hash`. 
4. For the **Input Data**, I clicked the `+` button and selected the hashes field coming from the raw Webhook event payload (`$exec.hashes`).
<img src="https://i.imgur.com/iFOB2SI.png"/>

#### 3. Testing the Extraction
1. Running a test execution confirmed that the node successfully extracted the clean hash string (e.g., `61c0...d8a1`) from the raw Sysmon text block.

## Part 3: Enriching Threat Intelligence with VirusTotal API

Once I extracted the hash, I needed to know: *"Is this file actually dangerous?"* I integrated VirusTotal, an online database that scans files with dozens of antivirus engines to automate this analysis.

---

### Steps I Took

#### 1. Obtaining my VirusTotal API Key
1. I created a free account on [virustotal.com](https://virustotal.com).
2. I clicked my user profile icon and copied my private **API Key**.
<img src="https://i.imgur.com/NXFQsbO.png"/>

#### 2. Adding and Authenticating VirusTotal in Shuffle
1. I dragged the **VirusTotal** app onto the Shuffle canvas and connected it to my Regex node.
2. I entered my **API Key** to establish a secure connection. (For the Authentication, type `VT-Auth`)
<img src="https://i.imgur.com/wsNwMjS.png"/>

#### 3. Configuring the Hash Query & Fixing Input Mapping
1. I set the action to **Get a Hash Report**.
2. **Troubleshooting:** Initially, mapping the raw Regex output passed success flags and group names alongside the hash, causing VirusTotal to return an `HTTP 404 Resource Not Found` error.
3. **The Fix:** I updated the parameter mapping to explicitly pull only the raw list items using the following expression:
   ```text
   \$SHA256_Regex_Extract.list.group
   ```
<img src="https://i.imgur.com/AHf8W7s.png"/>

#### 4. Verifying the Threat Report
1. I re-ran the execution and received an `HTTP 200 OK` response. 
2. The response returned full threat attributes, confirming that 50+ security vendors categorized the file hash as **Mimikats / Credential Stealer**.

## Part 4: Automating Alert Ticket Creation in The Hive

With confirmed threat intelligence, the system needs to create an official incident ticket so SOC analysts can track, investigate, and remediate the threat.

---

### Steps I Took

#### 1. Setting Up a Service Account & API Key in The Hive
1. I logged into The Hive web portal (`http://<THE_HIVE_IP>:9000`).
2. I created a dedicated organization and added a new service account user named `soar-service` with **Analyst** permissions.
3. I clicked the eye icon next to `soar-service` and generated a unique **API Key**.

#### 2. Connecting The Hive to Shuffle
1. I dragged **The Hive** app onto my Shuffle canvas and connected it to the VirusTotal node.
2. I configured authentication using my newly created API Key and specified my server URL: `http://<THE_HIVE_IP>:9000`.

#### 3. Configuring Advanced JSON Alert Payload
1. I changed the configuration mode from **Simple** to **Advanced**.
2. I mapped dynamic parameters so the ticket contains real data from the attack:

| Ticket Field | Value / Dynamic Mapping |
| :--- | :--- |
| **Title** | `Mimikats Activity Detected on Host` |
| **Description** | `$exec.title` *(Raw alert title from SIEM)* |
| **Severity** | `2` *(Medium/High Severity)* |
| **TLP / PAP** | `2` *(Traffic Light Protocol: Amber)* |
| **Tags** | `["T1003", "Mimikats", "Wazuh"]` *(MITRE ATT&CK Mapping)* |
| **Source** | `Wazuh-SIEM` |
<img src="https://i.imgur.com/05UQZCU.png"/>

#### 4. Testing Ticket Generation
1. I executed the workflow node. Shuffle returned an `HTTP 201 Created` status code.
2. I logged into The Hive as an analyst and verified that a brand-new alert ticket appeared in the dashboard!

## Part 5: Sending Automated Email Alerts to SOC Analysts

While tickets are great for tracking, analysts need immediate real-time notifications for critical incidents so they don't miss an active breach.

---

### Steps I Took

#### 1. Adding the Email Node in Shuffle
1. I dragged the **Shuffle Email** app onto the canvas and connected it alongside The Hive node.
2. I selected the action: **Send Email Shuffle**.

#### 2. Configuring Notification Details
1. **Recipient (To):** `hicksnaz@gmail.com` (any email of your chose)*
2. **Subject:** `Mimikats Detected!`
3. **Body:** I built a formatted message containing key investigation context:

```text

A high-severity security alert has been triggered on your network.

- Threat: Mimikats Execution
- Host Computer: realchill-pc
- SIEM Rule ID: 100002
- VirusTotal Report Status: Confirmed Malicious
- Incident Ticket: Created in The Hive

Please log into The Hive to review the full incident details and begin containment.
```
<img src="https://i.imgur.com/kQk3wUy.png"/>

#### 3. Testing Email Delivery
1. I ran the node test and checked my inbox.
2. The email arrived instantly with all dynamic fields correctly populated!

## Part 6: End-to-End Workflow Verification & Testing

To prove that the complete automated pipeline works smoothly without any human intervention, I ran a full end-to-end test.

---

### 🔄 The Full Automated Lifecycle

```text
[1. Attacker Runs Mimikats on VM]
               │
               ▼
[2. Sysmon Captures Event ID 1]
               │
               ▼
[3. Wazuh SIEM Triggers Rule 100002]
               │
               ▼
[4. Webhook Posts Data to Shuffle SOAR]
               │
               ▼
[5. Shuffle Extracts SHA256 Hash via Regex]
               │
               ▼
[6. VirusTotal API Confirms Malicious Hash]
               │
               ▼
┌──────────────┴──────────────┐
│                             │
▼                             ▼
[7. The Hive Ticket Created]  [8. Analyst Email Sent]
```

---

### Final Validation Test
1. I logged into my **Windows 11 target VM**.
2. I launched a **PowerShell** window and executed `.\mimikats.exe`.
3. Within less than **5 seconds**, the following actions occurred simultaneously:
   * **Wazuh** caught the process creation event.
   * **Shuffle** received the Webhook trigger payload.
   * The file hash was extracted and verified against **VirusTotal**.
   * A brand-new alert ticket appeared inside **The Hive**.
   * An urgent alert email landed in my inbox.
