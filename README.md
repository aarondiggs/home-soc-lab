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
