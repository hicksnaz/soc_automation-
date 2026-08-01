# 🛡️ Modern SOC Automation Project - Part 2: Configuring The Hive, Elasticsearch, Cassandra & Wazuh Agent

Welcome back to the documentation for my updated Security Operations Center (SOC) Automation Project! In Part 1, I built my base infrastructure by setting up a Windows 11 endpoint with Sysmon, deploying a Wazuh SIEM server in the cloud, and installing the foundational packages for The Hive.

In this second part, I will configure The Hive, its database backend (Apache Cassandra), its search engine (Elasticsearch), and connect my Windows 11 endpoint to Wazuh as an active agent so I can start gathering security logs.

---

## 📋 Table of Contents
- [Part 1: Configuring Apache Cassandra for The Hive](#part-1-configuring-apache-cassandra-for-the-hive)
- [Part 2: Installing and Configuring Elasticsearch](#part-2-installing-and-configuring-elasticsearch)
- [Part 3: Updating Directory Permissions for The Hive (`/opt/thp`)](#part-3-updating-directory-permissions-for-the-hive-optthp)
- [Part 4: Configuring The Hive Application Settings](#part-4-configuring-the-hive-application-settings)
- [Part 5: Opening Firewalls and Accessing The Hive Web Interface](#part-5-opening-firewalls-and-accessing-the-hive-web-interface)
- [Part 6: Setting up VirtualBox Guest Additions on Windows 11](#part-6-setting-up-virtualbox-guest-additions-on-windows-11)
- [Part 7: Deploying and Connecting the Wazuh Windows Agent](#part-7-deploying-and-connecting-the-wazuh-windows-agent)

## Part 1: Configuring Apache Cassandra for The Hive

Apache Cassandra is the distributed database that stores all case management data, alerts, and investigations for The Hive. The configuration file must be updated with the server's specific network details to function correctly.

### Configuration Steps

1. **Open the Configuration File**  
   Log into the Hive cloud server via SSH and open the Cassandra configuration file using `nano`:
   ```bash
   sudo nano /etc/cassandra/cassandra.yaml
   ```

2. **Modify Network and Cluster Parameters**  
   Inside the file, locate and update the following variables:
   * **Cluster Name:** Change from `Test Cluster` to `realchill`. (whatever name you choose)
  <img src="https://i.imgur.com/WSyVsji.png"/>

   * **Listen Address:** Replace `localhost` with the Hive server's public IP address (`155.138.x.x`).
  <img src="https://i.imgur.com/HDp9ung.png"/>

   * **RPC Address:** Update this parameter to match the public IP address (`155.138.x.x`).
  <img src="https://i.imgur.com/ZD9E9qK.png"/>

4. **Save Changes**  
   Save and close the file by pressing `Ctrl + X`, then `Y`, then `Enter`.

5. **Reset and Restart the Service**  
   Stop the service, clear old working data to ensure a clean deployment, and restart Cassandra:
   ```bash
   sudo systemctl stop cassandra
   sudo rm -rf /var/lib/cassandra/*
   sudo systemctl start cassandra
   sudo systemctl status cassandra
<img src="https://i.imgur.com/FmpaWVu.png"/>

## Part 2: Installing and Configuring Elasticsearch

Elasticsearch is the search engine that powers fast text searches and data indexing for The Hive. It must be properly bound to your network interface to communicate with the rest of the stack.

### Configuration Steps

1. **Open the Configuration File**  
   Open the Elasticsearch configuration file using `nano`:
   ```bash
   sudo nano /etc/elasticsearch/elasticsearch.yml
   ```

2. **Modify Node and Network Parameters**  
   Locate the following sections, uncomment them by removing the `#` symbol, and update their values:
   * **cluster.name:** Change to `realchill`.
   * **node.name:** Set to `node-1`.
 <img src="https://i.imgur.com/tmgh6ht.png"/>
     
   * **network.host:** Replace `localhost` with your Hive server's public IP address.
   * **http.port:** Ensure this points to port `9200`.
   * **cluster.initial_master_nodes:** Change to `["node-1"]`
<img src="https://i.imgur.com/Tg20ULf.png"/>

3. **Save Changes**  
   Save and close the file by pressing `Ctrl + X`, then `Y`, then `Enter`.

4. **Enable and Start the Service**  
   Reload the systemd daemon, enable Elasticsearch to start on boot, and start the service:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable elasticsearch
   sudo systemctl start elasticsearch
   sudo systemctl status elasticsearch
   ```
<img src="https://i.imgur.com/SrWKkoS.png"/>

## Part 3: Updating Directory Permissions for The Hive (/opt/thp)

Before launching The Hive application, directory ownership must be modified. This ensures that the specialized service account has the proper permissions to read and write application files.

### Configuration Steps

1. **Verify Current Directory Ownership**  
   Navigate to the `/opt` directory and list the contents to inspect current user and group ownership:
   ```bash
   cd /opt/thp
   ll
   ```
   *Note: Observe that the `thp` (The Hive Project) folder is initially owned by the `root` user.*

2. **Modify Ownership Permissions**  
   Change both the owner and group ownership of the directory recursively (`-R`) to the dedicated `thehive` user account:
   ```bash
   sudo chown -R thehive:thehive /opt/thp
   ```

3. **Verify Changes**  
   Execute the listing command again to confirm that user and group permissions are successfully updated:
   ```bash
   ll
   ```
<img src="https://i.imgur.com/QmP8zwC.png"/>

## Part 4: Configuring The Hive Application Settings

With the database and search engine fully operational, the final step is to update The Hive's main application settings file. This allows the core service to communicate with Cassandra and Elasticsearch.

### Configuration Steps

1. **Open the Application Configuration File**  
   Open the main configuration file using `nano`:
   ```bash
   sudo nano /etc/thehive/application.conf
   ```

2. **Modify Application and Database Links**  
   Locate and update the following settings to match your network deployment:
   * **Host Name / Base URL:** Replace `127.0.0.1` (localhost) with your Hive server's public IP address.
   * **Cluster Name:** Set this parameter to `realchill`. (whatever name you chose for your cluster)
   * **Elasticsearch Hosts:** Replace the default localhost IP with your public IP address.
<img src="https://i.imgur.com/Pho54km.png"/>

3. **Save Changes**  
   Save and close the file by pressing `Ctrl + X`, then `Y`, then `Enter`.

4. **Enable and Start the Service**  
   Configure the service to start automatically on system boot and initialize the application:
   ```bash
   sudo systemctl enable thehive
   sudo systemctl start thehive
   sudo systemctl status thehive
   ```
<img src="https://i.imgur.com/psF3YiQ.png"/>

## Part 5: Opening Firewalls and Accessing The Hive Web Interface

By default, cloud firewalls block external web traffic. To access the web interface from a remote host machine, explicit communication ports must be allowed through the firewall.

### Configuration Steps

1. **Configure the Local Firewall**  
   Open the application network interface by allowing incoming traffic on port `9000` (The Hive's default web port) using the Uncomplicated Firewall (UFW):
   ```bash
   sudo ufw allow 9000
   ```

2. **Access the Web Interface**  
   On your local host machine browser, navigate to the deployment IP address:
   ```text
   http://<YOUR_THEHIVE_PUBLIC_IP>:9000
   ```

3. **Perform Initial Authentication**  
   Log into the dashboard using the default initial administrative credentials:
   * **Username:** `admin@thehive.local`
   * **Password:** `secret`
<img src="https://i.imgur.com/cRpcTWh.png"/>

> [!NOTE]  
> Unlike older iterations where The Hive stayed completely unrestricted, modern community tiers apply an integrated 14-to-15-day evaluation license window. Ensure all active lab scenarios and exercises are finalized within this timeframe.

## Part 6: Setting up VirtualBox Guest Additions on Windows 11

To optimize the management of the Windows 11 virtual machine, VirtualBox Guest Additions must be installed. This software enables dynamic screen resizing, shared clipboards for copy/paste functionality, and drag-and-drop support between the host machine and the guest VM.

### Configuration Steps

1. **Mount the Guest Additions Image**  
   At the top of the VirtualBox Windows 11 VM window, navigate to the **Devices** menu and select **Insert Guest Additions CD Image...**

2. **Launch the Installer**  
   Open **File Explorer** inside the Windows 11 VM, navigate to the virtual optical CD drive, and double-click the setup file:
   ```text
   VBoxWindowsAdditions.exe
   ```

3. **Complete the Installation Wizard**  
   Proceed through the installation wizard prompts, approve the necessary kernel driver installations, and restart the virtual machine when prompted to finalize changes.
   <img src="https://i.imgur.com/s482HVh.png"/>

   ## Part 7: Deploying and Connecting the Wazuh Windows Agent

With the Wazuh SIEM server deployed and fully operational, the Wazuh agent must be installed on the Windows 11 target machine. This configuration enables the endpoint to securely stream Windows Event logs and Sysmon telemetry back to the central SIEM.

### Configuration Steps

1. **Access the Agent Deployment Wizard**  
   Log into the Wazuh Web UI via the web browser inside your Windows 11 virtual machine. From the main dashboard home page, select **Deploy new agent**.

2. **Generate the Deployment Script**  
   Configure the agent settings with the following parameters:
   * **Operating System:** Windows
   * **Wazuh Server IP:** Input your Wazuh server's public IP address.
   * **Agent Name:** `realchill-windows` (choose whatever)
<img src="https://i.imgur.com/BRRx69x.png"/>
<img src="https://i.imgur.com/1FWm6ly.png"/>

3. **Install and Initialize the Agent**  
   Open **PowerShell as Administrator** inside the Windows 11 VM, paste the auto-generated deployment command block to download the installer, and then start the endpoint service:
   ```powershell
   net start wazuhsvc
   ```
<img src="https://i.imgur.com/JTgvqRQ.png"/>

4. **Configure Network Firewalls for Agent Traffic**  
   By default, cloud firewalls restrict agent telemetry streams. Connect to your Wazuh server via SSH and execute the following commands to allow inbound communication on the required ports using the Uncomplicated Firewall (UFW):
   ```bash
   sudo ufw allow 1514
   sudo ufw allow 1515
   ```
<img src="https://i.imgur.com/Mfjhl3c.png"/>
<img src="https://i.imgur.com/0W5D88V.png"/>
