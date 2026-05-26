# Home SOC Lab: Endpoint Telemetry and Network Forensics

### Project Overview

This home lab was built to simulate network reconnaissance, analyse endpoint logs, and inspect network traffic in an isolated environment. The goal was to configure security monitoring tools and understand how host-based defences impact network visibility.

### Security & Configuration Note
All IP addresses used in this lab (specifically the `10.0.2.0/24` subnet) belong to the private network range defined in RFC 1918. These are non-routable, isolated addresses used exclusively within the VirtualBox host-only/NAT network environment. No actual infrastructure or public IPs were exposed during this simulation.

## Technical Stack

* Hypervisor: Oracle VirtualBox

* Target Machine: Windows 11

* Attacker/Analyst Machine: Ubuntu Linux

* Tools: Microsoft Sysmon, Wireshark, Windows Event Viewer, Nmap, vsFTPd

# Environment Setup & Installation

To replicate this environment, the following configuration and installation steps were performed:

### Network & Virtual Machine Configuration

Installed Oracle VirtualBox and created two virtual machines: Windows 11 (Target) and Ubuntu Linux (Attacker).

![Virtual Machines Setup Complete](./images/virtualsetupcomplete.png)

Configured a NAT Network (subnet 10.0.2.0/24) in VirtualBox preferences to allow secure communication between both virtual machines while keeping them isolated from the physical home network.

![Nat Network Setup Complete](./images/Natnetworksetupcomplete.png)

Assigned static IP addresses to ensure reliable communication (Windows host: 10.0.2.3).

### Windows Endpoint Setup (Sysmon Installation)

Downloaded Microsoft Sysmon from the official Sysinternals suite.

Downloaded a popular, security-hardened Sysmon configuration file (e.g., SwiftOnSecurity) to filter out noise and capture high-fidelity security events.

Installed Sysmon via administrative PowerShell/Command Prompt using the command:

sysmon.exe -i sysmonconfig-export.xml

Verified that the Microsoft-Windows-Sysmon/Operational event log channel was successfully created and active in the Windows Event Viewer.

Command Breakdown:

sysmon.exe: The main System Monitor executable that runs as a background service.

-i: The installation flag that registers Sysmon as a Windows service.

sysmonconfig-export.xml: Your custom configuration file. It dictates which activities (like process creation, file changes, or network connections) are logged.

How the Filters Work:
The XML configuration acts as a traffic controller for your logs, ensuring you only collect relevant security data while ignoring "noise."

onmatch="include": Only logs events that match the specific criteria defined in the rule.

onmatch="exclude": Logs everything except for the events that match these criteria. This is typically used to filter out trusted background processes to keep your log files clean and performant.


### Ubuntu Attacker Setup (Wireshark & vsFTPd Installation)

Installed Nmap and Wireshark on the Ubuntu VM:

sudo apt update

sudo apt install nmap wireshark -y

Configured Wireshark to run without root privileges by reconfiguring the package and adding the user to the Wireshark group:

sudo dpkg-reconfigure wireshark-common

sudo usermod -aG wireshark $USER

Installed and configured vsFTPd (Very Secure FTP Daemon) to act as our plaintext FTP target:

sudo apt install vsftpd -y

sudo systemctl start vsftpd

sudo systemctl enable vsftpd

Network Analysis Tools
sudo apt update: Refreshes the local package index against the online repositories to ensure you are downloading the latest available versions of the tools.

sudo apt install nmap wireshark -y: Downloads and installs Nmap (network discovery) and Wireshark (packet analysis). The -y flag automatically confirms the installation prompts, allowing for a non-interactive setup.

Wireshark Privilege Configuration
sudo dpkg-reconfigure wireshark-common: This modifies the security permissions of the dumpcap binary. By default, only root can capture packets. This command allows the system to grant "packet capture" capabilities to specific users.

sudo usermod -aG wireshark $USER: Adds your current user to the wireshark group. This is a security best practice, as it allows you to run Wireshark without elevated root (sudo) privileges, reducing the risk of a compromised GUI application having full system control.

vsFTPd Service Management
sudo apt install vsftpd -y: Installs the "Very Secure FTP Daemon," a lightweight and common FTP server.

sudo systemctl start vsftpd: Immediately launches the FTP background service so it can begin accepting connections.

sudo systemctl enable vsftpd: Configures the service to trigger automatically upon system boot. This ensures your lab target is always ready for testing whenever the VM is powered on, without requiring manual intervention.


## Endpoint Monitoring and Troubleshooting

I deployed Microsoft Sysmon on the Windows endpoint to gather system events.

### Event Viewer Snap-in Crash

During log analysis, the Windows Event Viewer crashed with a System.InvalidOperationException error because of high log volume.

![Event Viewer Crash](./images/event_viewer_error.png)

### Process Analysis (Event ID 1)

I analysed Event ID 1 (Process Creation) to check parent-child process relationships. Using the filtered logs, I verified the execution details of system tools:

Launching Notepad from Windows Search showed the parent process as taskhostw.exe.

![Sysmon Notepad Taskhostw](./images/event_viewer_taskhostw.png)

Launching commands like whoami successfully generated an Event ID 1 log, showing the exact image path and process ID.

![Sysmon Whoami Event 1](./images/event_viewer_whoami_event1.png)

### Network Reconnaissance and Firewall Configuration

From the Ubuntu machine, I used Nmap to scan port 445 (SMB) on the Windows host.

### Port Filtered State

The initial scan returned a filtered state. This happened because the default Windows Defender Firewall profiles were active and silently dropping incoming TCP probes.

![Nmap Filtered Scan](./images/nmap_filtered.png)

Analysis of the filtered state for 10.0.2.3

The Command: nmap -Pn -p 445 10.0.2.3

What it does: This command probes TCP port 445 (SMB) on the target host while bypassing the standard "ping" (host discovery) phase. By using -Pn, you force the scan to proceed even if the target is configured to ignore ICMP requests.

Why is this happening: This is a direct result of an active Windows Defender Firewall profile on the target machine. The firewall is configured to silently drop incoming TCP packets destined for port 445. Because the firewall discards the packets instead of sending a rejection message, Nmap is left waiting for a response that never arrives, leading it to mark the port as filtered.

What this tells you:

Active Defence: The host is not simply unprotected; it has an active firewall policy that successfully obscures the service status from your network scans.

Reduced Visibility: The firewall is effectively hiding the SMB service. You cannot confirm if the service is running or what version it is, because the firewall acts as a barrier that prevents your scan probes from reaching the target application.

### Disabling Firewall for Telemetry Capture

To allow the network traffic to reach the OS layer so Sysmon could log the activity, I disabled the firewall profiles via Windows CMD:

netsh advfirewall set allprofiles state off

![Firewall](./images/firewall.png)

Firewall Configuration
netsh advfirewall set allprofiles state off: This command uses the Network Shell (netsh) utility to disable the Windows Defender Firewall for all three profiles (Domain, Private, and Public).

Why this is necessary for the lab:
By default, the Windows Firewall is designed to block unsolicited incoming traffic. Disabling it allows your Nmap scans and network probes to reach the host OS, which is required for Sysmon to generate the Event ID 3 (Network Connection) logs.

Security Warning: Disabling the firewall is only appropriate for isolated, non-production lab environments. In a real-world scenario, you would instead create specific "Allow" rules for the service you are testing (e.g., allowing port 445 for SMB) rather than disabling the entire firewall, as the latter leaves the system completely exposed to network-based attacks.

### Capturing Event ID 3

After disabling the firewall and scanning the correct Windows IP address (10.0.2.3), the packets successfully reached the host. This immediately generated an Event ID 3 (Network Connection Detected) log inside Sysmon, capturing the source IP, destination IP, and target port.

![Sysmon Event 3](./images/sysmon_event3_part1.png)

Understanding Event ID 3 (Network Connection)
Event ID 3: Network Connection Detected: This event logs TCP/UDP connections on the machine. Unlike a simple firewall log that only records "allowed" or "blocked" traffic, Sysmon provides granular detail by linking the network activity directly to a specific process.

Why this is significant for your lab:

Process-to-Network Mapping: In your screenshot, Sysmon doesn't just tell you that a connection happened on port 445; it identifies which specific process initiated or accepted that connection. This allows you to verify that your Nmap scan is actually interacting with the SMB service (or whatever service is listening on that port).

Closing the Visibility Gap: By disabling the firewall, you transitioned from a "stealth" state (where probes are silently dropped) to an "active" state. Event ID 3 serves as the "smoking gun" in your logs, confirming that your attacker machine (Ubuntu) successfully reached the target (Windows) and that the target acknowledged the connection.

Telemetry Correlation: This event allows you to correlate the timestamp of your Nmap scan on the Ubuntu VM with the exact moment the Windows host registered the incoming connection. This correlation is the foundational skill of Network Forensics.

### Packet Inspection with Wireshark

I configured Wireshark on Ubuntu to capture traffic on the virtual network interface. To do this without root permissions, I reconfigured wireshark-common and managed dumpcap privileges.

Understanding Packet Inspection with Wireshark
The Challenge of Packet Capture: Modern operating systems restrict access to the Network Interface Card (NIC) to prevent unauthorized applications from "sniffing" raw data. By default, only the root (superuser) has the authority to put a network interface into "promiscuous mode"—a state where it captures all passing traffic, not just traffic intended for that specific machine.

Reconfiguring wireshark-common: This utility is essential because it bridges the gap between security and utility. It modifies the dumpcap binary (the backend engine that Wireshark uses to capture packets). By assigning it specific capabilities (CAP_NET_RAW and CAP_NET_ADMIN), you allow a non-root user to perform live captures safely.

Why this matters for your lab:

Visibility: Once these permissions are set, Wireshark gains full access to the virtual network adapter. It can now reconstruct the stream of packets sent between your Windows target and the Ubuntu attacker machine.

Forensic Capability: As seen in your FTP testing, Wireshark allows you to reassemble these raw packets into readable data (e.g., login credentials). Without these permissions, the capture tool would simply return an "Access Denied" error or fail to list any available interfaces.

Plaintext FTP Credential Capture

To test unencrypted traffic, I configured a vsFTPd server on Ubuntu and connected to it from the Windows machine. Using Wireshark's "Follow TCP Stream" feature, I recovered the login credentials directly from the traffic stream because FTP transmits data in plain text.

![Wireshark FTP](./images/wireshark_ftp.png)

This demonstration confirms that FTP is an inherently insecure protocol. Because the entire authentication exchange—including the username (hacekirs123) and password (jojo550)—is transmitted in cleartext, any attacker positioned on the same network segment can easily perform a Man-in-the-Middle (MitM) attack and intercept credentials using tools like Wireshark or tcpdump. To secure such communications, protocols like FTPS (FTP over SSL/TLS) or SFTP (SSH File Transfer Protocol) should be implemented, as they encrypt the command and data channels, rendering intercepted traffic unreadable.

## Conclusion and Skills Verified

Log Analysis: Filtering and tracking Sysmon Event ID 1 and Event ID 3.

Network Tools: Using Nmap flags and troubleshooting port states (open, closed, filtered).

Packet Analysis: Using Wireshark display filters and rebuilding TCP streams.

Troubleshooting: Resolving Windows MMC snap-in issues and Linux packet capture permissions.
