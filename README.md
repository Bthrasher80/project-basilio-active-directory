# Project Basilio — Active Directory Domain Deployment

## Overview

This project documents the deployment and configuration of an Active Directory environment on a Windows Server 2022 VM hosted on Proxmox as part of Project Basilio, a self-built SOC home lab. The lab covers domain controller deployment, OU structure design, user and role creation, DNS configuration, network infrastructure setup via pfSense, and endpoint domain join. This environment serves as the identity and access management foundation for the broader Project Basilio SOC lab.

**Domain:** basil.local
**Domain Controller:** DC01 — Windows Server 2022 Standard Evaluation
**DC IP:** 10.0.0.200 (static)
**Proxmox Node:** projectbasilio-proxmox
**Hypervisor:** Proxmox VE 9.1.2
**Firewall/Router:** pfSense (VM 100)

## Objectives

- Deploy Windows Server 2022 as a domain controller on Proxmox
- Configure Active Directory Domain Services for the basil.local domain
- Design and implement an organizational unit structure
- Create user accounts and role-based groups
- Configure DNS forwarding for external resolution
- Join a Windows endpoint to the domain and verify authentication
- Configure pfSense as the network gateway with dual-NIC setup

## Infrastructure Overview

### Proxmox VM Inventory

| VM ID | Name | OS | Role | RAM | Disk | Network |
|-------|------|----|------|-----|------|---------|
| 100 | pfSense | FreeBSD | Firewall/Router | 2GB | 20GB | vmbr0 (dual NIC) |
| 101 | ubuntu-test | Ubuntu | Test endpoint | - | - | - |
| 102 | AD-DC01 | Windows Server 2022 | Domain Controller | 8GB | 80GB | vmbr1 |
| 103 | qualys-vm | TBD | Vulnerability Scanner | TBD | TBD | TBD |
| 104 | Wazuh-Server-Ubuntu | Ubuntu | SIEM | - | - | - |
| 105 | linux-victim-01 | Ubuntu | Linux target endpoint | 4GB | 40GB | vmbr0 |

*Note: Qualys vulnerability scanner deployment planned for future lab stage.*

📸 `screenshots/Proxmox_node_view___all_5_VMs_visible__pfSense__ubuntu-test__AD-DC01__openvas__Wazuh_.png`

### Network Configuration

Two Linux bridges configured on physical NIC eno2:

| Bridge | CIDR | Gateway | Purpose |
|--------|------|---------|---------|
| vmbr0 | 10.0.0.50/24 | 10.0.0.1 | Primary LAN — lab traffic |
| vmbr1 | - | - | Isolated internal segment |

📸 `screenshots/Proxmox_Network_tab___vmbr0__vmbr1_Linux_bridges_configured__eno2_physical_NIC.png`

## Domain Controller Deployment

### 1. VM Provisioning — AD-DC01

Provisioned Windows Server 2022 VM on Proxmox with the following configuration:

| Setting | Value |
|---------|-------|
| VM ID | 102 |
| Name | AD-DC01 |
| OS | Windows Server 2022 (OVMF UEFI) |
| Machine | pc-q35-8.1 |
| RAM | 8.00 GiB |
| CPUs | 4 cores (1 socket) |
| Disk | 80GB SCSI (local-lvm) |
| Network | vmbr1 |
| KVM Virtualization | Enabled |

📸 `screenshots/Proxmox_AD-DC01_summary___running__8GB_RAM__4_CPUs.png`
📸 `screenshots/Proxmox_AD-DC01_hardware___Windows_Server_20222025__OVMF_UEFI__80GB__vmbr1.png`

### 2. Static Network Configuration

Configured a static IP on DC01 to ensure stable DNS and domain resolution across the lab:

| Setting | Value |
|---------|-------|
| IPv4 Address | 10.0.0.200 |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 10.0.0.1 |

📸 `screenshots/Project_Basilio___Domain_Controller_Static_Network_Configuration.png`

### 3. Active Directory Domain Services Installation

Installed AD DS role and promoted DC01 as the domain controller for the new forest `basil.local`. Confirmed domain membership and server role in Server Manager.

| Setting | Value |
|---------|-------|
| Computer Name | DC01 |
| Domain | basil.local |
| OS | Windows Server 2022 Standard Evaluation |
| RAM | 7.89 GB |
| Disk | 79.37 GB |

📸 `screenshots/Project_Basilio___Active_Directory_Domain_Controller_Deployment.png`

### 4. DNS Forwarder Configuration

Configured DNS forwarders on DC01 to enable external name resolution while maintaining internal AD DNS authority:

| Forwarder | FQDN |
|-----------|------|
| 8.8.8.8 | dns.google |
| 1.1.1.1 | one.one.one.one |

📸 `screenshots/Project_Basilio___Active_Directory_DNS_Forwarder_Configuration.png`

## Active Directory Structure

### 5. Organizational Unit Design

Built out the basil.local OU structure to reflect a realistic enterprise environment with role-based separation:

| OU | Purpose |
|----|---------|
| Admins | Administrative accounts |
| Servers | Server computer objects |
| Users-OU | Standard user accounts |
| Workstations | Endpoint computer objects |

📸 `screenshots/Project_Basilio___Active_Directory_Domain_Structure.png`
📸 `screenshots/Project_Basilio___Active_Directory_Domain_Structure__2_.png`

### 6. User and Role Account Creation

Created user accounts in Users-OU representing realistic SOC lab personas:

| Name | Type | OU |
|------|------|----|
| John Analyst | User | Users-OU |
| Sarah Admin | User | Users-OU |

📸 `screenshots/Project_Basilio___Active_Directory_Users_and_Role_Groups.png`

## Endpoint Domain Join

### 7. Windows Endpoint Joined to Domain

Joined WINDOWS-VICTIM-01 to the basil.local domain and verified authentication using the John Analyst account:

| Setting | Value |
|---------|-------|
| Device Name | WINDOWS-VICTIM-01 |
| Full Domain Name | WINDOWS-VICTIM-01.basil.local |
| Logged In As | janalyst@basil.local |
| RAM | 6.00 GB |
| OS | Windows 11 64-bit |

📸 `screenshots/Project_Basilio___Windows_Endpoint_Joined_to_Active_Directory_Domain.png`

## Network Infrastructure

### 8. pfSense Firewall — Dual NIC Configuration

pfSense deployed as VM 100 with two VirtIO network adapters both on vmbr0, providing gateway and firewall services for the lab network:

| Interface | Assignment | IP |
|-----------|------------|-----|
| WAN (vtnet0) | vmbr0 | External |
| LAN (vtnet1) | vmbr0 | 192.168.50.1/24 |

📸 `screenshots/Proxmox_pfSense_VM_hardware___2_NICs_on_vmbr0_confirmed.png`
📸 `screenshots/pfSense_console___LAN_192_168_50_124__interface_assignment_menu.png`

### 9. Linux Victim Endpoint Provisioned

Provisioned linux-victim-01 (VM 105) as a target endpoint for future threat hunting and detection exercises:

| Setting | Value |
|---------|-------|
| VM ID | 105 |
| Name | linux-victim-01 |
| OS | Ubuntu 22.04 |
| CPUs | 2 cores |
| RAM | 4.00 GiB |
| Disk | 40GB SCSI |
| Network | vmbr0 |

📸 `screenshots/Proxmox___linux-victim-01_summary__running__40GB_disk.png`
📸 `screenshots/Proxmox_Create_VM_confirm_screen___linux-victim-01__VM_ID_105.png`

## VM Creation Process — Wazuh Server Reference

The following screenshots document the Proxmox VM creation wizard used to provision the Wazuh-Server VM (VM ID 104), included here as a reference for the lab build process:

📸 `screenshots/Proxmox_Create_VM___General_tab__naming_Wazuh-Server__VM_ID_104.png`
📸 `screenshots/Proxmox_Create_VM___OS_tab__Ubuntu_22_04_ISO_selected.png`
📸 `screenshots/Proxmox_Create_VM___System_tab__q35SeaBIOS.png`
📸 `screenshots/Proxmox_Create_VM___Disks_tab__80GB_SCSI.png`
📸 `screenshots/Proxmox_Create_VM___CPU_tab__1_core.png`
📸 `screenshots/Proxmox_Create_VM___Memory_tab__8096_MiB.png`

## Key Takeaways

- Windows Server 2022 deploys cleanly on Proxmox with OVMF UEFI and pc-q35 machine type
- Static IP assignment on the domain controller is essential before promoting to AD DS — dynamic IP will break DNS resolution across the domain
- DNS forwarders (8.8.8.8, 1.1.1.1) allow domain-joined machines to resolve external names without leaving the internal DNS hierarchy
- OU structure should reflect realistic enterprise role separation from the start — retrofitting it later breaks GPO targeting
- pfSense on Proxmox with dual VirtIO NICs provides reliable lab gateway and firewall services
- Vulnerability scanning will be handled by Qualys — planned for a future lab stage

## Related Projects

- [Project Basilio — Main Hub](https://github.com/Bthrasher80/project-basilio)
- [Network Segmentation](https://github.com/Bthrasher80/project-basilio-network-segmentation)
- [Centralized Logging](https://github.com/Bthrasher80/project-basilio-centralized-logging)
- [TOR Browser Threat Hunt](https://github.com/Bthrasher80/project-basilio-tor-threat-hunt)
- [Wazuh SIEM Deployment](https://github.com/Bthrasher80/project-basilio-wazuh-siem)
