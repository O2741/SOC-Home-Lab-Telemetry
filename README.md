# Home SOC Lab: Endpoint Telemetry and Network Forensics

## Project Overview
This home lab was built to simulate network reconnaissance, analyze endpoint logs, and inspect network traffic in an isolated environment. The goal was to configure security monitoring tools and understand how host-based defenses impact network visibility.

## Technical Stack
* Hypervisor: Oracle VirtualBox
* Target Machine: Windows 11
* Attacker/Analyst Machine: Ubuntu Linux
* Tools: Microsoft Sysmon, Wireshark, Windows Event Viewer, Nmap, vsFTPd

---

## 1. Endpoint Monitoring and Troubleshooting

I deployed Microsoft Sysmon on the Windows endpoint to gather system events. 

### Event Viewer Snap-in Crash
During log analysis, the Windows Event Viewer crashed with a System.InvalidOperationException error because of high log volume.

![Event Viewer Crash](./images/event_viewer_error.png)

### Process Analysis (Event ID 1)
I analyzed Event ID 1 (Process Creation) to check parent-child process relationships. Using the filtered logs, I verified the execution details of system tools:
* Launching Notepad from Windows Search showed the parent process as taskhostw.exe.

![Sysmon Notepad Taskhostw](./images/event_viewer_taskhostw.png)

* Launching commands like whoami successfully generated an Event ID 1 log, showing the exact image path and process ID.

![Sysmon Whoami Event 1](./images/event_viewer_whoami_event1.png)

---

## 2. Network Reconnaissance and Firewall Configuration

From the Ubuntu machine, I used Nmap to scan port 445 (SMB) on the Windows host.

### Port Filtered State
The initial scan returned a filtered state. This happened because the default Windows Defender Firewall profiles were active and silently dropping incoming TCP probes.

![Nmap Filtered Scan](./images/nmap_filtered.png)

### Disabling Firewall for Telemetry Capture
To allow the network traffic to reach the OS layer so Sysmon could log the activity, I disabled the firewall profiles via Windows CMD:
```cmd
netsh advfirewall set allprofiles state off
Capturing Event ID 3
After disabling the firewall and scanning the correct Windows IP address (10.0.2.3), the packets successfully reached the host. This immediately generated an Event ID 3 (Network Connection Detected) log inside Sysmon, capturing the source IP, destination IP, and target port.

3. Packet Inspection with Wireshark
I configured Wireshark on Ubuntu to capture traffic on the virtual network interface. To do this without root permissions, I reconfigured wireshark-common and managed dumpcap privileges.

Plaintext FTP Credential Capture
To test unencrypted traffic, I configured a vsFTPd server on Ubuntu and connected to it from the Windows machine. Using Wireshark's "Follow TCP Stream" feature, I recovered the login credentials directly from the traffic stream because FTP transmits data in plain text.

Conclusion and Skills Verified
Log Analysis: Filtering and tracking Sysmon Event ID 1 and Event ID 3.

Network Tools: Using Nmap flags and troubleshooting port states (open, closed, filtered).

Packet Analysis: Using Wireshark display filters and rebuilding TCP streams.

Troubleshooting: Resolving Windows MMC snap-in issues and Linux packet capture permissions.