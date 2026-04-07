# Purple Team Project: Both Sides Of The Same Incident

Table of Contents
1. [Introduction](#1-introduction)  
2. [Infrastructure](#2-infrastructure)
    * [**2.1 Target Configuration (Vulnerability Simulation)**](#21-target-configuration-vulnerability-simulation)
3. [Techniques](#3-techniques)
   * [**3.1 Reconnaissance**](#31-reconnaissance)  
        * [Active Scanning](#active-scanning-t1595) [(T1595)](https://attack.mitre.org/techniques/T1595/)
        * [Gather Victim Host Information](#gather-victim-host-information-t1592) [(T1592)](https://attack.mitre.org/techniques/T1592/)
    * [**3.2 Initial Access**](#32-initial-access)
        * [Brute Force Password Guessing](#brute-force-password-guessing-t1110001) [(T1110.001)](https://attack.mitre.org/techniques/T1110/001/)
        * [Windows Management Instrumentation](#windows-management-instrumentation-t1047) [(T1047)](https://attack.mitre.org/techniques/T1047/)
        * [Remote Services: SSH](#remote-services-ssh-t1021004) [(T1021.004)](https://attack.mitre.org/techniques/T1021/004/)
    * **3.3 Post-Exploitation** (**Work In Progress**)
        * Privilege Escalation: Highjack Execution Flow: Service File Permission Weakness [(T1574.010)](https://attack.mitre.org/techniques/T1574/010/)
        * Privilege Escalation: Bypass User Access Control [(T1548.002)](https://attack.mitre.org/techniques/T1548/002/)
        * Exfiltration Over C2 Channel [(T1041)](https://attack.mitre.org/techniques/T1041/)
    *  **3.4 Persistence**
       * Registry Run Keys / Startup Folder [(T1547.001)](https://attack.mitre.org/techniques/T1547/001/)
       * Scheduled Tasks [(T1053.005)](https://attack.mitre.org/techniques/T1053/005/)
4. Conclusion
5. Lessons Learned
6. References


## 1. Introduction
Currently in cybersecurity, a professional needs to go beyond executing exploits, they must have a deep understanding of the telemetry and forensic data generated during an incident. This **Purple Team** project was designed to simulate a complete incident lifecycle, applying my knowledge of offensive execution and defensive analysis.  

This project focuses on the **Initial Access**, **Privilege Escalation** and **Persistence** phases of an incident. 

**Resource Limitations GUI Implementation:** Due to limited resources in the host computer I decided to implement a **minimalist interface (XFCE)** to access **Wazuh Dashboard** from it. While not optimal for high-end production servers, this choice reflects a balance between the need for local accessibility and the lab's hardware constraints, ensuring that the **Wazuh Indexer** and **Manager still** have sufficient resources.

**Advanced Telemetry: Sysmon and SwiftOnSecurity:** To ensure a visibility beyond standard Microsoft Event Logs, Microsoft Sysmon was integrated into the Windows target. Utilizing SwiftOnSecurity configuration, this lab ensures a more optimal setup of logging of process creation, network connections and file system changes. This lab is following the MITRE ATT&CK framework.  

This project has purposely [disabled or modified some defense capabilities](#21-target-configuration-vulnerability-simulation), including, but not limited to, disabling Windows Defender for easier detection in Wazuh (SIEM) and to avoid interference with the techniques used to exploit the endpoint.


## 2. Infrastructure
For a better view of the network environment and telemtry flow, see the diagram below:
![Purple Team Lab Infrastructure](\Purple_Team_Lab_Infrastructure.png)
_Figure 1: Lab's infrastructure_

This lab's infrastructure is composed of 4 Virtual Machines (VMs) each serving a role in thje security lifecycle:

- **pfSense** is used to simulate the firewall and to isolate the laboratory from the host machine.  

- **Kali Linux** is the adversary role in this lab, using brute force and exploits to elevate its privilege and get persistence.  

- **Windows Endpoint** is the target running an SMB Server that is misconfigured to be publicly visible. A Wazuh agent is installed in it. It also has Windows Defender disabled, since the the goal of this project is to gather and analyse the telemetry generated because of the attack.

- **Ubuntu Server** is where Wazuh Manager and Indexer are installed, it is the brain of the project. Due to limited resources from the host machine, a minimalist interface (XFCE) was installed to analyze the generated logs. Wazuh Dashboard is accessed from a browser from here.

### 2.1 Target Configuration (Vulnerability Simulation)
To simulate a realistic attack surface and ensure telemetry collection, the Windows Endpoint was configured with the following modifications:

* **Defender:** Disabled to prevent automatic blocking of the exploitation phase, focusing the project on **Detection (Wazuh)** rather than **Prevention**.  
* **WMI Remote Access:** The interface was made accessible over the network to simulate a misconfigured administrative service. 
Modified ```LocalAccountTokenFilterPolicy``` in the Windows Registry:
Path:  ```"HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" ``` 
Value: ```LocalAccountTokenFilterPolicy = 1 (DWORD)```
* **Firewall Rules:** To allow ports to be easily opened and standardized I organized them in an array and created a script that runs each rule that was added to the list:
``` 
$FirewallRules = @(
    @{ Name = "OpenSSH-In"; Port = 22; Display = "SSH Port" },
    @{ Name = "RPC-WMI-In"; Port = 135; Display = "RPC/WMI Port" }
)
```
The following script runs and sets up each rule:
```
foreach ($Rule in $FirewallRules) {
    if (Get-NetFirewallRule -Name $Rule.Name -ErrorAction SilentlyContinue) {
        Write-Host "LOG: Rule $($Rule.Name) already exists." -Foreground Yellow
    }   else    {
        New-NetFirewallRule -Name $Rule.Name -DisplayName $Rule.Display -LocalPort $Rule.Port -Protocol TCP -Action Allow -Direction Inbound
        Write-Host "LOG: Rule $($Rule.Name) successfully created!" -Foreground Green
    }
}
```
* **Network Profile:** The default network profile (Public) has a more restrictive firewall against external connections, meaning it blocks unsolicited inbound cinnection by default, resulting in _timeouts_. It was necessary to modify the profile to Private to make the endpoint remotely manageable, which is a prerequisite for the techniques being tested in this project.
For this modification the following command was used:
```
Set-NetConnectionProfile -InterfaceAlias "Ethernet0" -NetworkCategory Private
```
* **Advanced Auditing: Microsoft Sysmon** was installed with the **SwiftOnSecurity** configuration to provide high detailed process creation (Event ID 1) and network connections (Event ID 3) triggered by WMI calls.

## 3. Techniques
The techniques used in this lab aim to simulate a complete incident lifecycle based on MITRE ATT&CK framework. This includes **Reconnaissance**, **Initial Access**, **Privilege Escalation**, **Data Exfiltration** and **Persistence**, demonstrating both offensive capabilities and expected response/telemetry from the Blue Team.

#### 3.1 Reconnaissance
One of the most important phases in penetration testing is Reconnaissance. It allows the adversary to understand the target, and also enables him to visualize the attack surface of the network, which leads to use the most suitable techniques for this system.
#### Active Scanning (T1595)
 A scan using Nmap flags ```-sV```and ```-sC``` enumerated publicly visible services. It was identified the operating system as Windows and the following services: **RPC-WMI (Port 135)**, **NetBIOS (Port 139)** and **SMB (Port 445)**.
    
**Blue Team analysis:** Nmap doesn't complete the TCP's "Three way Handshake" or opens and closes a connection very quickly. Often, Windows doesn't consider it an error and doesn't log it in Event Viewer. Wazuh is a HIDS (Host Intrusion Detection System) and for it to be logged, it is necessary to have a NIDS (Network Intrusion Detection System) like Suricata or Snort.


 ![Initial Scan using Nmap](\Prints\Reconnaissance\Initial_Scan.png)
 _Figure 2: Initial Nmap scan with services running_

#### Gather Victim Host Information (T1592)
In the initial scan, nmap's enumeration script (nbstat) discovered the NetBIOS name: **WINDOWS-VICTIM**. In misconfigured systems, hostnames can be used as a potential username where often administrative accounts share the machine's name. It can be used to limit the scope of valid users in the network, making it possible to use brute force to gather valid credentials.

**Blue Team analysis:** NetBIOS requests (nbstat) are a common and legitimate behaviour of a Windows network protocol. Being an automatic response, it rarely generates security or alert logs in Event Viewer, as a result, it becomes a quiet way the adversary can gather information about the target.

### 3.2 Initial Access
After gathering a potential username in the reconnaissance phase, the next step is to exploit the services running on the target. In this instance I will be using brute force to compromise the WMI service on port 135/tcp.
#### Brute Force: Password Guessing (T1110.001)
For this task it will be used ```NetExec```, the successor of ```CrackMapExec```. Instead of using a robust password list like ```rockyou.txt```, for demonstration and telemetry purposes, I'll be using only the first 100 passwords in ```rockyou.txt```.

**Findings:** During the initial Brute Force (T1110.001) attempts, the Windows host succesfully triggered Account Lockout Policy, preventing further access.

**Telemetry:** This event was captured by Wazuh as high-severity alerts:

_Wazuh rule level ranges from 0 to 16, it determines the severity and alerting action for detection events. Levels indicate importance (0 = ignored and 16 = critical), with higher level triggering alerts and active responses._

**Rule 60122 Level 5 - EventID 4625:** An account failed to log on.
**Rule 60204 Level 10 - EventID 4625:** Multiple Windows Logon Failures.
**Rule 60115 Level 9 - EventID 4740:** User account was locked out.
_While rules **60122** and **60204** have the same EventID, Wazuh analyzes the behavior that is happening. After multiple attempts in a short time span the SIEM identifies this as a potential **Brute Force** attack and triggers a higher-severity alert._

![Brute Force detection](\Prints\Initial-Access\Brute-Force-Fail-Wazuh.png)
_Figure 3: Wazuh Brute Force detection showing Multiple Logon Failures and Account Lockout._

For brute force to be successful, login attempts have to be set to a high amount for testing and telemetry gathering:
```net accounts /lockoutthreshold:999```

After the threshold is set, another attempt was made and NetExec successfully found a match for the username we identified in the Reconnaissance phase.

![Brute Force success](\Prints\Initial-Access\Brute-Force-Success.png)
_Figure 4: NetExec found a match for WINDOWS-VICTIM_.

This time is possible to observe a successful logon in the telemetry. Right after a match was found, the tool logs off, because its purpose was only to discover the password, not establish a connection.

**Rule 92652 Level 6 - EventID 4624:** An account was successfully logged on.
**Rule 60137 Level 3 - EventID 4623:** Impacket has disconnected.

![Wazuh detecting successful logon](\Prints\Initial-Access\Brute-Force-Logon-Wazuh.png)
_Figure 5: Wazuh logs detecting a new logon in the network._

#### Windows Management Instrumentation (T1047)
After obtaining the creditials, the next step is to connect to the WMI. I'll be using impacket-wmiexec.

Although the credentials were valid, the connection via WMI failed. This is due to WMI being a remote administrative tool, which implies that you have to be part of the administrator group. From an attacker perspective what can be deducted is that this user has only low-privileges meaning that another route has to be identified to compromise this endpoint.

![WMI Connection Failed](\Prints\Initial-Access\WMI-Failed.png)
_Figure 6: WMI access denied due to a low-privilege user even with valid credentials_
#### Remote Services: SSH (T1021.004)
SSH is one of the most used protocols for remote access. Although secure, it is still subject to human mishandling when lost or leaked credentials fall into the hands of bad actors.

After the attempt to compromise the WMI failed, another network scan was performed to observe if a different vector could be exploited.

![New Nmap scan performed](\Prints\Initial-Access\New-Nmap-Scan.png)
_Figure 7: New Nmap scan was performed in search of another vector to be exploited._

As seen on _Figure 8_ we can observe that now there is an SSH service open on Port 22.

For this technique, it will be used the same credentials that have been identified during previous phases.

![Connecting to SSH](\Prints\Initial-Access\SSH-Connection.png)
_Figure 8: Successful login into SSH using valid credentials discovered during previous phases._

The ```whoami```command demonstrates that we successfully compromised the target.

_The **WARNING!** sign is a disclaimer from recent versions of OpenSSH informing that it is not protected against post-quantum decryption. This **does not** have any influence on the integrity of this project._

**Blue Team Analysis:** The telemetry gathered in Wazuh demonstrates the highly effective monitoring. While Windows generates multiple logs with EventID 4624, the SIEM is able to distinguish between the service initialization (Rule 67024), the operational privileges attribution (Rule 67028). The Windows Logon Success (Rule 60106) is a immediate confirmation that Windows communicate that the credentials are valid and the connection is established and the success of the final authentication via network (Rule 60200), providing a complete view of the chain of events of the incident.

The telemetry also shows the closure of operational sessions after authentication, tagged by the Rule 60137 (EventID 4634). This behavior is common in connections via network (_Logon type 3_), where the operating system closes the authentication process as soon as the interactive shell is established for the user.

![Wazuh detecting a new login into SSH](\Prints\Initial-Access\SSH-Log-Wazuh.png)
_Figure 9: Wazuh detecting a new login in SSH._
 
