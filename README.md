# SOC Automation Lab

## Overview
Welcome to the central repository for my SOC Automation Project!

When I set out to build this lab, my goal wasn't just to install security tools and look at dashboard charts. I wanted to experience what it takes to design, build, and run a modern, enterprise-style Security Operations Center (SOC) pipeline from scratch.

In a traditional security team, analysts often spend hours copying file hashes, pasting them into threat check websites, writing up incident tickets, and sending manual emails. I wanted to eliminate that friction by building an automated pipeline where an attack on an endpoint is detected, analyzed, enriched with threat intelligence, ticketed, and alerted on in under 5 seconds.

## Objectives 
   - Watch Everything: Set up detailed activity logging on a Windows computer using Sysmon so no sneaky behavior goes unseen.

   - Collect It All in One Place: Send all computer logs to a central security brain (Wazuh SIEM).

   - Write Custom Alarm Rules: Create custom detection rules to catch specific attack tools like Mimikats (a tool bad guys use to steal passwords).

   - Automate the Boredom: Use Shuffle SOAR to connect all my tools together like Lego bricks.

   - Check Threats Automatically: Automatically ask VirusTotal if a suspicious file is actually dangerous.

   - Alert the Team Instantly: Automatically create an incident ticket in The Hive and drop an email straight into my inbox.

## Lab Breakdown
### Lab 01: Environment Setup & Core Infrastructure
- What I Built: I spun up a cloud-hosted Linux server for my Wazuh SIEM and The Hive incident portal. Locally, I built a isolated Windows 11 Virtual Machine to act as my target endpoint.

- Key Concept: Setting up the foundation. Think of this phase as building the physical house before bringing in security guards.
### Lab 02: Configuring The Hive, Elasticsearch, Cassandra & Wazuh Agent
- What I Built: I installed Sysmon on my Windows 11 target machine using a security configuration file and attached the Wazuh Agent.

- Key Concept: Getting data moving. Telemetry simply means "activity reports." By installing Sysmon, my endpoint started recording granular events like process creations, network connections, and file changes.

### Lab 03: Sysmon Ingestion, Log Archiving & Custom Threat Detection Rules
- What I Built: I turned on full raw log archiving (logall_json) on my Wazuh Manager, built an archives index in the web dashboard, and downloaded Mimikats to test my setup. I then authored a custom XML detection rule (Rule 100002) that flags Mimikats execution even if the executable file is renamed.

- Key Concept: Teaching the SIEM what "bad" looks like. Default rules miss custom attacks, so I wrote my own signature that specifically inspects an executable's inner metadata (originalFileName).

### Lab 04: Orchestration & Response with Shuffle SOAR, VirusTotal & The Hive
- What I Did: I hooked Wazuh up to Shuffle SOAR. When Mimikats runs, Shuffle grabs the alert, pulls out the file's fingerprint (SHA256 hash), checks it with VirusTotal, opens a ticket in The Hive, and emails me.

- Key Concept: Full automation. Instead of spending 15 minutes manually investigating a suspicious file, my system completes the entire pipeline in less than 5 seconds.
