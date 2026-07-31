# 🛡️ Modern SOC Automation Project - Part 1: Environment Setup & Core Infrastructure

Welcome to the documentation for my updated **Security Operations Center (SOC) Automation Project**! I built this end-to-end lab to simulate real-world enterprise threat detection, log collection, and incident response workflows.

Because application interfaces and installation steps change over time, I created this updated guide to make building the lab smoother, more reliable, and beginner-friendly. 

---

## 📋 Table of Contents
- [Part 1: Installing and Verifying VirtualBox](#part-1-installing-and-verifying-virtualbox)
- [Part 2: Downloading the Windows 11 ISO Image](#part-2-downloading-the-windows-11-iso-image)
- [Part 3: Creating and Configuring the Windows 11 Virtual Machine](#part-3-creating-and-configuring-the-windows-11-virtual-machine)
- [Part 4: Installing and Configuring Windows 11](#part-4-installing-and-configuring-windows-11)
- [Part 5: Installing Sysmon and Olaf's Modular Configuration](#part-5-installing-sysmon-and-olafs-modular-configuration)
- [Part 6: Taking a VirtualBox Snapshot](#part-6-taking-a-virtualbox-snapshot)
- [Part 7: Deploying and Configuring the Wazuh Server in the Cloud](#part-7-deploying-and-configuring-the-wazuh-server-in-the-cloud)
- [Part 8: Deploying The Hive Incident Response Platform](#part-8-deploying-the-hive-incident-response-platform)

---

## Part 1: Installing and Verifying VirtualBox

To run my target endpoint (Windows 11) locally, I need a virtualization tool. While the absolute newest version of VirtualBox (7.2) is available, I found that version 7.1 (specifically version 7.1.12) is much more stable for my setup.

### Steps I Took:
1. I went to the official VirtualBox website and downloaded VirtualBox 7.2.6 for Windows hosts.
2. To ensure the downloaded installer file was safe and not corrupted during download, I verified its SHA-256 checksum.
3. I opened PowerShell in my `Downloads` folder and ran the following hash check command:
   ```powershell
   Get-FileHash .\VirtualBox-7.2...exe

<img src="https://i.imgur.com/hawMaih.png"/>

## Part 2: Downloading the Windows 11 ISO Image

Next, I needed an operating system installation file (ISO) to build my local Windows 11 target virtual machine.


### Steps I Took:

1. **Navigated to Official Microsoft Page:** I visited Microsoft's official Windows 11 download portal to ensure I was using a secure, uncorrupted image.
2. **Downloaded Creation Media Tool:** Instead of choosing the direct ISO link (which sometimes fails or drops mid-download), I downloaded and ran the **Create Windows 11 Installation Media** tool.
3. **Selected ISO Output:** I accepted the license terms, verified the language settings, and selected **ISO file** when prompted to choose which media to use.
4. **Saved to Local Storage:** I saved the `.iso` file directly into my `Downloads` folder so VirtualBox could easily locate it during VM creation.
<img src="https://i.imgur.com/D43HIIG.png"/>
<img src="https://i.imgur.com/oC9qUw5.png"/>

## Part 3: Creating and Configuring the Windows 11 Virtual Machine

With my ISO file downloaded, I set up a new virtual machine inside VirtualBox to serve as my local target workstation.

### Steps I Took:

1. **Created New Machine in VirtualBox:**
   - I opened VirtualBox, clicked **New**, and named the virtual machine `Windows 11`.
   - I linked the ISO Image selector directly to the Windows 11 `.iso` file I saved in my `Downloads` folder.
   - VirtualBox automatically assigned the OS Type as **Microsoft Windows** and Version as **Windows 11 (64-bit)**.

2. **Allocated System Resources:**
   - **RAM (Memory):** Increased from the 4 GB default up to **8 GB (8192 MB)** to keep the operating system responsive.
   - **Processors:** Allocated **2 vCPUs** so background security logging doesn't freeze the system.
   - **Virtual Storage:** Set the virtual hard disk capacity to **80 GB**.

3. **Initiated Windows OS Setup:**
   - I powered on the VM and pressed a key when prompted to boot from the virtual CD/DVD drive.
   - Selected **Windows 11 Pro** as the target edition.
   - Clicked **"I don't have a product key"** to proceed with the installation.

<img src="https://i.imgur.com/hulrsDv.png"/>

## Part 4: Installing and Configuring Windows 11

Once the initial Windows setup wizard finished copying operating system files, I walked through the initial Out-of-Box Experience (OOBE) setup to configure my target endpoint.


### Steps I Took:

1. **Region & Keyboard Settings:** Selected my region and keyboard layout preferences, then assigned the computer name `realchill-pc`. (whatever name you choose)
2. **Setup Preference:** When asked how I wanted to set up the device, I selected **Set up for personal use**.
3. **Account Sign-In:** Signed in with Microsoft account credentials to satisfy the initial online setup requirement (noting that local account workarounds can also be used if preferred).
4. **Privacy & Telemetry Settings:** Disabled extra Microsoft diagnostic data and telemetry options (like location tracking and tailored ads) to keep background network traffic as clean as possible.
5. **Desktop Initialization:** Skipped optional setup features and landed on the clean, fresh Windows 11 desktop environment.

<img src="https://i.imgur.com/x44m4iD.png"/>

## Part 5: Installing Sysmon and Olaf's Modular Configuration

System Monitor (Sysmon) is a critical Windows system service that logs deep activity (like process creations, network connections, and file changes) directly to the Windows Event Log. This is essential for feeding high-quality data into my SOC detection pipeline.


### Steps I Took:

1. **Downloaded Sysmon:** Inside my Windows 11 VM, I opened Microsoft Edge, searched for **Sysmon** on Microsoft's Sysinternals page, downloaded it, and extracted the ZIP folder.
2. **Downloaded Olaf's Modular Configuration:** I navigated to Olaf Hartong's Sysmon-modular GitHub repository. I viewed the raw `sysmonconfig.xml` file, right-clicked to save it, and placed it directly inside my extracted Sysmon folder.
3. **Installed Sysmon via PowerShell:** 
   - I opened PowerShell as an **Administrator**.
   - I used the `cd` command to navigate to my extracted Sysmon folder.
   - I installed Sysmon and applied the custom configuration file with the following command:
     ```powershell
     .\sysmon64.exe -i sysmonconfig.xml
     ```
4. **Verified the Installation:** 
   - I opened `Services.msc` and confirmed that the **Sysmon64** service was running.
   - I opened **Event Viewer** and navigated to `Applications and Services Logs` > `Microsoft` > `Windows` > `Sysmon` > `Operational` to confirm that rich telemetry events were actively generating.

<img src="https://i.imgur.com/EC5sGlW.png"/>
<img src="https://i.imgur.com/1OkxieM.png"/>
<img src="https://i.imgur.com/L8sXmsH.png"/>

## Part 6: Taking a VirtualBox Snapshot

Before moving on to cloud server deployments and running attack simulations, I saved a safe restore point for my virtual machine.

### Steps I Took:

1. **Accessed Snapshot Menu:** Inside VirtualBox, I selected my Windows 11 VM and clicked **Machine** > **Take Snapshot** (or used the `Ctrl + T` shortcut).
2. **Named the Restore Point:** I named the snapshot `sysmon-installed` and added a quick note indicating that Windows 11 is fully set up with Sysmon active.
3. **Saved the Snapshot:** Clicked **OK** to preserve the system state.

<img src="https://i.imgur.com/Xel1pAM.png"/>

## Part 7: Deploying and Configuring the Wazuh Server in the Cloud

Wazuh serves as my central Security Information and Event Management (SIEM) and Extended Detection and Response (XDR) platform. It aggregates telemetry from my Windows target endpoint, correlates event logs, and alerts me to suspicious behavior.


### Steps I Took:

1. **Provisioned Cloud Server on Vultr:**
   - Logged into my cloud provider (Vultr) and deployed a new instance.
   - **OS:** Ubuntu 24.04 LTS
   - **Hardware Specs:** 4 vCPUs and 8 GB RAM *(Wazuh requires at least 8GB RAM to run smoothly)*.
   - **Server Hostname:** `realchill-wazuh` (can use whatever hostname you choose)
<img src="https://i.imgur.com/ATSc8w1.png"/>

2. **Connected via SSH:** Opened my local terminal and logged into the cloud server:
   ```bash
   ssh root@<YOUR_WAZUH_PUBLIC_IP>
<img src="https://i.imgur.com/amsIZd8.png"/>

3. I updated and upgraded package repositories `(apt update && apt upgrade -y)`.

4. I downloaded and ran the official Wazuh installation assistant script:

   ```bash
   curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh && sudo bash wazuh-install.sh -a

<img src="https://i.imgur.com/xFQUQ9L.png"/>

5. After installation completed, I copied and saved the automatically generated admin credentials.

6. To access the web UI, I allowed incoming HTTPS traffic on port 443 using the Uncomplicated Firewall (UFW):

   ```Bash
   sudo ufw allow 443

<img src="https://i.imgur.com/kgcQ2By.png"/>

7. Finally, I opened my web browser, navigated to `https://<YOUR_WAZUH_PUBLIC_IP>`, bypassed the self-signed SSL certificate warning, and logged into the Wazuh dashboard successfully.

<img src="https://i.imgur.com/khpdz6L.png"/>

## Part 8: Deploying The Hive Incident Response Platform

The Hive is my Security Incident Response Platform (SIRP). It acts as a centralized case management system where security analysts can organize alerts, log investigative findings, attach evidence, and collaborate on security incidents.

### Steps I Took:

1. **Provisioned Cloud Server on Vultr:**
   - Deployed a second cloud server running **Ubuntu 24.04 LTS**.
   - **Hardware Specs:** 6 vCPUs and 16 GB RAM *(The Hive requires heavier resources to support its underlying Cassandra database and Elasticsearch engine)*.
   - **Server Hostname:** `realchill-thehive` (can use whatever hostname you choose)

2. **Connected via SSH:** Opened my terminal and logged into the new cloud instance:
   ```bash
   ssh root@<YOUR_THEHIVE_PUBLIC_IP>

3. Updated Packages & Installed Prerequisites:
   Updated system package lists and installed required database and backend dependencies:
<img src="https://i.imgur.com/cl4okv2.png"/>
<img src="https://i.imgur.com/yzMXuSD.png"/>
   - Java Virtual Machine (JVM)
<img src="https://i.imgur.com/eelfnnD.png"/>
   - Apache Cassandra
<img src="https://i.imgur.com/VgY6VA0.png"/>
   - Elasticsearch
<img src="https://i.imgur.com/w30i8qa.png"/>

4. Finally, I downloaded and installed The Hive package (v5.5.7) using wget and prepared the foundational environment configuration files.

<img src="https://i.imgur.com/bDkQmCS.png"/>

<img src="https://i.imgur.com/YSosyzH.png"/>
