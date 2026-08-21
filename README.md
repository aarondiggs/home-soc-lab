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
