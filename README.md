# Purple Team Project: Both Sides Of The Same Incident

Table of Contents
1. [Introduction](#1-introduction)  
2. [Infrastructure](#2-infrastructure)
3. Techniques  
   * **3.1 Reconnaissance**  
        * Active Scanning [(T1595)](https://attack.mitre.org/techniques/T1595/)
        * Gather Victim Host Information [(T1592)](https://attack.mitre.org/techniques/T1592/)
    * **3.2 Initial Access**
        * Brute Force Password Guessing [(T1110.001)](https://attack.mitre.org/techniques/T1110/001/)
    * **3.3 Post-Exploitation**
        * Privilege Escalation: Highjack Execution Flow: Service File Permission Weakness [(T1574.010)](https://attack.mitre.org/techniques/T1574/010/)
        * Privilege Escalation: Bypass User Access Control [(T1548.002)](https://attack.mitre.org/techniques/T1548/002/)
    *  **3.4 Persistence**
       * Registry Run Keys / Startup Folder [(T1547.001)](https://attack.mitre.org/techniques/T1547/001/)
       * Scheduled Tasks [(T1053.005)](https://attack.mitre.org/techniques/T1053/005/)
4. Conclusion
5. Lessons Learned
6. References


## 1. Introduction
Currently in cybersecurity, a professional needs to go beyond executing exploits, they must have a deep understanding of the telemetry and forensic data generated during an incident. This **Purple Team** project was designed to simulate a complete incident lifecycle, applying my knowledge of offensive execution  and defensive analysis.  

This project focuses on the **Initial Access**, **Privilege Escalation** and **Persistence** phases of an incident. 

**Resource Limitations GUI Implementation:** Due to limited resources in the host computer I decided to implement a **minimalist interface (XFCE)** to access **Wazuh Dashboard** from it. While not optimal for high-end production servers, this choice reflects a balance between the need for local accessibility and the lab's hardware constraints, ensuring that the **Wazuh Indexer** and **Manager still** have sufficient resources.

**Advanced Telemetry: Sysmon and SwiftOnSecurity:** To ensure a visibility beyond standard Microsoft Event Logs, Microsoft Sysmon was integrated into the Windows target. Utilizing SwiftOnSecurity configuration, this lab ensures a more optimal setup of logging of process creation, network connections and file system changes. This lab is following the MITRE ATT&CK framework.  


## 2. Infrastructure
This lab's infrastructure is composed of 4 Virtual Machines (VMs) each serving a role in thje security lifecycle:

- **pfSense** is used to simulate the firewall and to isolate the laboratory from the host machine.  

- **Kali Linux** is the adversary role in this lab, using brute force and exploits to elevate its privilege and get persistence.  

- **Windows Endpoint** is the target running an SMB Server that is misconfigured to be publicly visible. A Wazuh agent is installed in it. It also has Windows Defender disabled, since the the goal of this project is to gather and analyse the telemetry generated because of the attack.

- **Ubuntu Server** is where Wazuh Manager and Indexer are installed, it is the brain of the project. Due to limited resources from the host machine, a minimalist interface (XFCE) was installed to analyze the generated logs. Wazuh Dashboard is accessed from a browser from here.
