# Home SOC / SIEM Lab

## Overview
This project documents the design and implementation of a home Security Operations Center (SOC) lab environment built using open source tools. The goal is to simulate real-world threat detection and incident response workflows in a controlled environment.

The lab consists of four virtual machines: a SIEM server, two target endpoints, and an attacker machine. Attack simulations are performed from the attacker machine against the endpoints, with Wazuh aggregating telemetry and generating alerts that are triaged and investigated through the dashboard.

Skills demonstrated in this project include:
- Virtual lab design and provisioning
- SIEM deployment and configuration
- Endpoint telemetry collection and enrichment
- Vulnerability management (detection, triage, remediation, and risk acceptance)
- Attack simulation and threat detection
- Custom detection rule development mapped to the MITRE ATT&CK framework

---

## Lab Environment

| VM           | Operating System     | Role                                                                            |
| ------------ | -------------------- | ------------------------------------------------------------------------------- |
| SIEM Server  | Ubuntu Server 24.04  | Hosts Wazuh Manager (aggregates and analyzes security telemetry from endpoints) |
| Attacker     | Kali Linux 2026.3    | Simulates adversary activity against target endpoints                           |
| Endpoint (1) | Ubuntu Desktop 26.04 | Target endpoint (generates Linux security telemetry)                            |
| Endpoint (2) | Windows 11 25H2      | Target endpoint (generates Windows security telemetry via Sysmon)               |

**Host Machine Specs**

| Processor                | RAM       | Disk  |
| ------------------------ | --------- | ----- |
| AMD Ryzen 5 5600X 6-Core | 32GB DDR4 | 954GB |

---

## Technologies Used

| Technology                      | Purpose                                                                            |
| ------------------------------- | ---------------------------------------------------------------------------------- |
| VMware Workstation Pro          | Hypervisor for provisioning and managing lab VMs                                   |
| Wazuh v4.14.7                   | Open-source SIEM/XDR (log aggregation, threat detection, vulnerability management) |
| Sysmon (Microsoft Sysinternals) | Windows system service providing detailed endpoint telemetry                       |
| SwiftOnSecurity Sysmon Config   | Community-vetted baseline Sysmon configuration                                     |
| Hydra                           | Network login brute force tool used for attack simulation                          |

---

## Section 1: Provisioning the Lab

### Objective
Provision four virtual machines in VMware Workstation Pro with appropriate resources for each role in the lab.

### VM Specifications

| VM           | OS                   | vCPU | RAM | Disk |
| ------------ | -------------------- | ---- | --- | ---- |
| SIEM Server  | Ubuntu Server 24.04  | 4    | 8GB | 80GB |
| Endpoint (1) | Ubuntu Desktop 26.04 | 2    | 4GB | 40GB |
| Endpoint (2) | Windows 11 25H2      | 2    | 4GB | 60GB |
| Attacker     | Kali Linux 2026.3    | 2    | 4GB | 40GB |

*Specifications reflect available resources on the host machine. RAM and vCPU allocations can be adjusted based on host capacity.*

### Provisioning Steps
Each VM was provisioned using the following process in VMware Workstation Pro:

1. Navigate to **File -> New Virtual Machine**
2. Select **"Installer disc image file (iso)"** and attach the appropriate ISO
3. At **Processor Configuration**, set 1 processor with the appropriate number of cores per the table above
4. Allocate RAM per the table above
5. At **Specify Disk Capacity**, set the disk size per the table above and select **"Split virtual disk into multiple files"**
6. Click **Finish** (do not power on the VM yet.)

*The above process was repeated for each VM. OS installation is covered in subsequent sections.*

### Notes on Windows 11 Provisioning
Windows 11 enforces a Microsoft account requirement during the Out of Box Experience (OOBE) by default. In a lab environment, a local account is preferred to avoid unnecessary Microsoft account integration. To bypass this, the VM's network adapter was temporarily disconnected in VMware Workstation settings prior to OS installation. With no network connectivity available, Windows proceeds through the OOBE without enforcing the Microsoft account requirement, allowing local account creation.

---

## Section 2: Deploying Wazuh

### Objective
Deploy Wazuh on the SIEM Server as the lab's central log aggregation and threat detection platform.

### What is Wazuh?
Wazuh is an open-source SIEM and XDR (Extended Detection and Response) platform. It consists of three core components:

- **Manager** (the analysis engine responsible for processing incoming logs, applying detection rules, and generating alerts)
- **Indexer** (an OpenSearch-based data store for log retention, storage, and search)
- **Dashboard** (a Kibana-based web interface for visualizing alerts, managing agents, and investigating findings)

All three components were deployed on a single node (the SIEM Server) using Wazuh's all-in-one installer.

### Steps

**1. Complete Ubuntu Server OS installation**

During installation, OpenSSH was enabled to allow remote terminal access to the server without needing to interact directly with the VM console. This allows copy + paste without the installation of open-vm-tools.

**2. Update the system**

```bash
sudo apt update && sudo apt upgrade -y
```

**3. Reboot**

```bash
sudo reboot
```

**4. Install Wazuh using the all-in-one installer**

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

> **Note:** 4.14.7 was the most current version of Wazuh available at time of installation. The installer outputs admin credentials at the end of the process (these are displayed only once and must be saved before the terminal is closed.)

**5. Verify all three services are running**

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

All three should show **active (running)**.

**6. Access the Wazuh dashboard**

Run `ip a` on the SIEM Server to obtain its local IP address. From the host machine's browser, navigate to: `https://<server-ip-address>`

Log in using the admin credentials saved during installation.

### Images
![Image shows Wazuh Manager process is 'active' on the SIEM server](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Wazuh%20Manager%20Active.png)

![Image of Wazuh Dashboard](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Wazuh%20Dashboard%20-%20First%20Access.png)

---

## Section 3: Enrolling Endpoints

### Objective
Install and configure Wazuh agents on both target endpoints so the SIEM server begins receiving security telemetry from each machine.

### Deploying the Wazuh Agent - Ubuntu Endpoint

**1. Generate the install command from the Wazuh dashboard**

In the Wazuh dashboard, navigate to **Deploy a New Agent**. Select **Linux** as the operating system and **amd64** as the architecture. Enter the SIEM Server's IP address as the server address and assign a name to the agent (e.g., `ubuntu-victim`). The dashboard generates a set of install commands pre-configured for the environment. Paste and run the generated commands in the terminal on the Ubuntu endpoint.

**2. Start the Wazuh agent**

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

**3. Verify in the dashboard**

Refresh the Wazuh dashboard. The Ubuntu endpoint should appear as "**Active**" in the Agents section within a few minutes.

### Deploying the Wazuh Agent - Windows Endpoint

**1. Generate the install command from the Wazuh dashboard**

Repeat the same process in the dashboard, selecting **Windows** as the operating system. The dashboard generates a PowerShell command pre-configured for the environment. Paste and run the generated PowerShell command.

**2. Start the Wazuh agent**

```powershell
NET START WazuhSvc
```

**3. Verify in the dashboard**

Both endpoints should now appear as **Active** in the Wazuh Agents section.

![Image of Wazuh Dashboard with both endpoints active](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Wazuh%20Dashboard%20-%20After%20Endpoints%20Deployed.png)

---

## Section 4: Enhancing Windows Telemetry with Sysmon

### Objective
Install Sysmon on the Windows endpoint to provide enriched telemetry to Wazuh beyond what default Windows Event Logging offers.

### Why Sysmon?
Wazuh is an XDR/SIEM platform, but it is only as effective as the telemetry it receives. Default Windows Event Logging lacks the granularity needed for effective threat detection (process creation events, for example, omit command-line arguments and parent-child process relationships that are critical for identifying malicious behavior.)

Sysmon (System Monitor) is a Windows system service and device driver from the Microsoft Sysinternals suite that addresses this gap by generating significantly richer endpoint telemetry, including:

- Full command-line arguments for every process created
- Parent-child process relationships
- Network connections per process
- File creation events with cryptographic hashes
- Registry modifications

Wazuh ingests this data via the Sysmon event log channel (`Microsoft-Windows-Sysmon/Operational`). Sysmon acts as a more detailed data source that enables Wazuh to detect sophisticated attack techniques that would otherwise go unnoticed.

### Installation

**1. Download Sysmon and the SwiftOnSecurity configuration**

- [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [SwiftOnSecurity config](https://github.com/SwiftOnSecurity/sysmon-config)

The SwiftOnSecurity configuration is a community-maintained baseline ruleset widely used in the security industry. It balances detection coverage with log noise reduction, making it a practical starting point for a lab environment.

**2. Install Sysmon with the configuration**

```powershell
sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

**3. Configure Wazuh to ingest Sysmon logs**

Open `C:\Program Files (x86)\ossec-agent\ossec.conf` and add the following block inside the `<ossec_config>` tags:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

**4. Restart the Wazuh agent**

```powershell
Restart-Service WazuhSvc
```

### Verification
Sysmon telemetry was verified by navigating to **Threat Hunting** in the Wazuh dashboard and selecting the Windows agent. On this page, there is a graph titled "Top 10 Alert groups evolution." On the right side of this graph, the log sources are listed. As seen in the image below, "sysmon" is on this list.

![Image of the Wazuh Threat Hunting dashboard, showing "sysmon" in the list of alert groups](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Sysmon%20Wazuh.png)

---

## Section 5: Vulnerability Management

### Objective
Perform vulnerability scans on both endpoints using Wazuh's built-in vulnerability detection module, triage findings by severity, remediate when possible, and document risk acceptance decisions for findings that cannot be immediately resolved.

### Ubuntu Endpoint

#### Initial Scan Results
![Ubuntu Vulnerability Scan 2](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Ubuntu%20Vulnerability%20Scan%202.png)

The initial scan against the Ubuntu endpoint returned a significant number of findings, the majority attributed to the Linux kernel packages (`linux-image-7.0.0-29-generic` and `linux-image-7.0.0-30-generic`).

#### Remediation Steps

**1. Update all packages**

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

**2. Remove unused kernel**

Following the update, two kernel versions were present on the system. As only `linux-image-7.0.0-30-generic` was active, the older `linux-image-7.0.0-29-generic` was removed to reduce attack surface:

```bash
sudo apt remove --purge linux-image-7.0.0-29-generic -y
sudo apt autoremove --purge -y
sudo reboot
```

This removal eliminated approximately half of the total findings, as each kernel package was contributing equally to the vulnerability count.

#### Post-Remediation Results
![Ubuntu Vulnerability Scan 3](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Ubuntu%20Vulnerability%20Scan%203.png)

#### Remaining Findings
The remaining findings are attributed entirely to `linux-image-7.0.0-30-generic` — the active running kernel. As the active kernel cannot be removed, these findings represent the current unpatched attack surface pending upstream kernel patches. This was documented as accepted risk with the following rationale:

- No patches are currently available in Ubuntu's repositories for the affected kernel version
- The system is a lab VM with no exposure to untrusted networks
- Findings will be re-evaluated as patches become available

---

### Windows Endpoint

#### Initial Scan Results
![Windows Vulnerability Dashboard](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Windows%20Vulnerability%20Scan.png)

#### CVE-2026-45585 (YellowKey)

| Field              | Detail                   |
| ------------------ | ------------------------ |
| CVE ID             | CVE-2026-45585           |
| Severity           | Medium (CVSS: 6.8)       |
| Affected Component | Windows BitLocker        |
| Attack Vector      | Physical access required |
| Patch Available    | Yes (KB5094126)          |

CVE-2026-45585, known as YellowKey, is a security feature bypass vulnerability affecting Windows BitLocker. It was disclosed by a researcher known as Nightmare Eclipse, with a proof-of-concept published on May 13, 2026, prompting Microsoft to publish an advisory on May 19, 2026. The patch `KB5094126` addresses the vulnerability.

**Remediation:** The system was confirmed to be running build 26200.9168, which is above the patched build threshold of 26200.8655 introduced by KB5094126. The patch was therefore already applied.

**False Positive**: Despite the patch being applied, CVE-2026-45585 continued to appear in the Wazuh vulnerability inventory. With no other explanation for the vulnerability scan triggering this detection, it's likely a false positive.

![Windows Version](https://github.com/aarondiggs/home-soc-lab/blob/main/images/winver.png)

---

## Section 6: Attack Simulation (SSH Brute Force)

### Objective
Simulate a brute force attack against the Ubuntu endpoint using Hydra from the Kali attacker machine, and observe Wazuh's detection capability against MITRE ATT&CK technique T1110 (Brute Force.)

### Background
Brute force attacks are one of the most common techniques used by threat actors to gain unauthorized access to systems. By repeatedly attempting credentials against an authentication service, an attacker can eventually discover valid credentials if no lockout or rate-limiting controls are in place. This simulation demonstrates how Wazuh detects this behavior through SSH authentication log analysis.

### Environment

| Machine                     | IP Address     | Role                                             |
| --------------------------- | -------------- | ------------------------------------------------ |
| Kali Linux (Attacker)       | 192.168.31.137 | Simulated adversary (runs Hydra)                 |
| Ubuntu Desktop (Endpoint 1) | 192.168.31.135 | Target (exposed SSH service)                     |
| SIEM Server (Wazuh)         | 192.168.31.134 | Detects authentication failures and fires alerts |

### Steps

**1. Confirm SSH is running on the Ubuntu endpoint**

SSH was installed and enabled on the Ubuntu Desktop endpoint prior to the simulation:

```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
```

Connectivity was verified from the Kali machine:

```bash
nc -v 192.168.31.135 22
```

**2. Create a targeted wordlist on the Kali machine**

A small wordlist was created on the Kali attacker machine containing common passwords followed by the correct credential, ensuring Hydra would produce both failed and successful authentication attempts (generating the full attack pattern in Wazuh):

```bash
echo -e "password\n123456\nadmin\nletmein\nqwerty\n<password>" > /tmp/passwords.txt
```

In a real-world attack, a threat actor would typically use a much larger wordlist such as `rockyou.txt` (14 million entries). A smaller list was crafted here to keep the simulation concise while still producing meaningful telemetry.

**3. Run the brute force attack**

Hydra was run against the Ubuntu endpoint's SSH service with a single thread (`-t 1`) to avoid triggering SSH's connection rate limiting, which causes connection errors:

```bash
hydra -l ubuntuv -P /tmp/passwords.txt ssh://192.168.31.135 -t 1
```

Hydra attempted 6 login tries and successfully identified the valid credential on the final attempt, completing the attack between 16:07:35 and 16:07:50.

![Image displays Hydra scan results](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Hydra%20scan.png)

**4. Observe detection in Wazuh**

The Threat Hunting module in the Wazuh dashboard was used to monitor events on the ubuntu-victim agent during the attack. The event timeline showed a clear pattern of authentication failures followed by a successful login, correlating directly with Hydra's output.

![Wazuh Threat Hunting dashboard, showcasing the alerts of the attack. Filtered to show only Ubuntu logs.](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Threat%20Hunting%20-%20Ubuntu%20Kali%20Hydra.png)

### Findings

Two distinct rules fired during the simulation:

**Rule 5760 (sshd): authentication failed**

| Field           | Detail                              |
| --------------- | ----------------------------------- |
| Rule ID         | 5760                                |
| Rule Level      | 5                                   |
| Description     | sshd: authentication failed         |
| Log Source      | journald (sshd-session)             |
| Rule Groups     | syslog, sshd, authentication_failed |
| MITRE Technique | T1110.001 (Password Guessing)       |
| MITRE Tactic    | Credential Access, Lateral Movement |
| Source IP       | 192.168.31.137 (Kali attacker)      |
| Target User     | ubuntuv                             |
| Fired Times     | 5                                   |

This rule fired once for each failed authentication attempt. The full log entry confirmed the source IP, target username, and timestamp for each attempt, providing a clear trail of the attack.

![Shows the expanded alert for sshd: authentication failed.](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Threat%20Hunting%20-%20sshd%20alert%20failed%20password.png)

**Rule 5715 (sshd): authentication success**

| Field           | Detail                                                                               |
| --------------- | ------------------------------------------------------------------------------------ |
| Rule ID         | 5715                                                                                 |
| Rule Level      | 3                                                                                    |
| Description     | sshd: authentication success                                                         |
| Log Source      | journald (sshd-session)                                                              |
| Rule Groups     | syslog, sshd, authentication_success                                                 |
| MITRE Technique | T1078 (Valid Accounts), T1021 (Remote Services)                                      |
| MITRE Tactic    | Defense Evasion, Persistence, Privilege Escalation, Initial Access, Lateral Movement |
| Source IP       | 192.168.31.137 (Kali attacker)                                                       |
| Target User     | ubuntuv                                                                              |

Rule 5715 fired one second after the final authentication failure, confirming successful access was obtained immediately following the brute force sequence.

![Shows the expanded alert for sshd: authentication success.](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Threat%20Hunting%20-%20sshd%20alert.png)

### Observations

**Rule 5763 (Multiple Authentication Failures) did not fire**

Wazuh's aggregation rule for multiple authentication failures (5763) was not triggered during this simulation. This is likely because the single-threaded attack (`-t 1`) spaced attempts far enough apart that they fell outside the rule's time window threshold. This is a notable finding (it demonstrates that low-and-slow brute force attacks can evade threshold-based detection rules while still being visible in individual event logs. This gap is addressed in Section 7, where a custom detection rule is developed to improve coverage of this scenario.)

**SSH rate limiting behavior**

During initial testing, running Hydra with multiple concurrent threads (`-t 4`) caused SSH's connection rate limiting to reject connections entirely, producing connection errors rather than authentication failure events. Reducing to a single thread (`-t 1`) resolved this. From a defensive perspective, SSH rate limiting provides meaningful protection against high-speed brute force attempts, but does not prevent slower, more deliberate attacks (reinforcing the need for properly tuned detection rules.)

---

## Section 7: Detection Engineering

### Objective
Develop custom Wazuh detection rules to address a gap identified during the brute force simulation in Section 6 (specifically, that Wazuh's built-in aggregation rule (5763) did not fire against the low-and-slow attack pattern produced by Hydra running at single-thread concurrency.) The goal is to write rules that correctly detect this pattern and demonstrate detection engineering as a core SOC analyst skill.

### Background
Wazuh ships with thousands of built-in detection rules covering a wide range of attack patterns. However, built-in rules are designed around general assumptions about attack behavior (they cannot account for every environment or attack pattern.) Detection engineering is the practice of writing and tuning custom rules to close gaps in coverage based on observed threats.

In this case, the built-in rule 5763 (Multiple Authentication Failures) did not fire during the brute force simulation because the single-threaded attack spaced authentication attempts far enough apart to fall outside the rule's detection threshold. The custom rules developed in this section address this gap directly, using thresholds tuned to the observed attack pattern.

Custom rules in Wazuh are written in XML and are stored in `/var/ossec/etc/rules/local_rules.xml`. This file is separate from Wazuh's built-in ruleset, ensuring custom rules persist across Wazuh updates without being overwritten.

### Rule Design

Two rules were created to implement a detection chain:

**Rule 100001** acts as an intermediary (it fires on every SSH authentication failure (inheriting from built-in rule 5760) and tags it with the appropriate MITRE ATT&CK technique.) This gives rule 100002 a clean, labeled event to count against.

**Rule 100002** is the aggregation rule (it fires when 5 or more instances of rule 100001 are observed from the same source IP within a 30-second window.) The threshold of 5 failures in 30 seconds was derived directly from the observed behavior of the brute force simulation in Section 6.

### Implementation

**1. Open the custom rules file on the SIEM Server**

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

**2. Remove the "example" rule and add the following rules**

![Custom Detection Rules](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Wazuh%20Custom%20Detection%20Rule%20Remote%20CLI.png)

**3. Restart the Wazuh manager to apply the changes**

```bash
sudo systemctl restart wazuh-manager
```

**4. Verify the rules appear in the dashboard**

Rules were verified by navigating to **Management -> Rules** in the Wazuh dashboard and filtering by custom rules. 

![Wazuh Dashboard filtered for Custom Rules](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Wazuh%20Dashboard%20Custom%20Rules.png)

### Testing

The brute force simulation from Section 6 was repeated from the Kali attacker machine to validate the new rules:

```bash
hydra -l ubuntuv -P /tmp/passwords.txt ssh://192.168.31.135 -t 1
```

### Results

The custom rules fired as expected. The Threat Hunting dashboard showed the following event sequence:

![Custom rules firing in Threat Hunting log](https://github.com/aarondiggs/home-soc-lab/blob/main/images/Wazuh%20Threat%20Hunting%20Custom%20Rule%20Firing.png)

Rule 100002 fired at level 12 after 5 authentication failures were observed from the same source IP within the 30-second window, followed immediately by a successful authentication event (rule 5715) (confirming the full brute force and access sequence was detected.)

---

## Summary

This project documents the design and implementation of a home SOC lab environment using open source tooling. A four-VM lab was provisioned in VMware Workstation Pro with Wazuh deployed as the central SIEM/XDR platform, aggregating telemetry from Linux and Windows endpoints enriched via Sysmon.

Key outcomes:
- Deployed and configured Wazuh
- Enrolled Linux and Windows endpoints as Wazuh agents
- Conducted vulnerability scans across both endpoints, triaging findings and documenting remediation and risk acceptance decisions
- Simulated an SSH brute force attack from Kali Linux and observed detection through Wazuh's built-in ruleset
- Identified a gap in built-in detection coverage and developed custom rules to address it, validated through a second attack simulation

---

## Lessons Learned

**Vulnerability scanner output requires context.** Raw findings are not meaningful without triage (severity, exploitability, and patch availability all inform the decision to remediate or accept risk.) Scanner feed lag can also produce false positives for already-patched vulnerabilities, a limitation worth understanding in any vulnerability management workflow.

**Detection gaps are expected.** The failure of Wazuh's built-in rule 5763 to fire against the low-and-slow brute force pattern was a realistic finding that led directly to a detection engineering exercise — the kind of continuous gap identification and remediation that defines real SOC work.

**Troubleshooting is part of the process.** Several components required reconfiguration during the project. Documenting these challenges demonstrates realistic problem-solving ability.
