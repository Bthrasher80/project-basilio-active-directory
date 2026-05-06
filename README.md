# Project Basilio — Active Directory Domain Deployment

## Overview

This project documents the full build-out of an Active Directory environment on Proxmox as part of Project Basilio, a self-built SOC home lab. The lab covers Windows Server 2022 VM provisioning, domain controller promotion, OU structure design, user and group creation, DNS configuration, Windows endpoint domain join, and Group Policy baseline configuration. This environment serves as the identity and access management foundation for the broader Project Basilio SOC lab and enables realistic attack simulation, detection engineering, and GPO-based audit policy enforcement.

**Domain:** basil.local  
**Domain Controller:** DC01 — Windows Server 2022 Standard Evaluation  
**DC IP:** 10.0.0.200 (static)  
**Workstation:** windows-victim-01 — Windows 11 Pro 25H2 — 10.0.0.101  
**Proxmox Node:** projectbasilio-proxmox  
**Hypervisor:** Proxmox VE 9.1.2  

---

## Objectives

- Provision a Windows Server 2022 VM on Proxmox and resolve VirtIO driver compatibility issues during installation
- Install and configure Active Directory Domain Services on DC01
- Promote DC01 as the forest root domain controller for basil.local
- Configure DNS and validate internal name resolution from domain-joined endpoints
- Design and implement a role-based OU structure reflecting a realistic enterprise environment
- Create user accounts and security groups scoped to SOC lab personas
- Join windows-victim-01 to the basil.local domain and validate authentication
- Open Group Policy Management and document the pre-GPO audit policy baseline

---

## Infrastructure Overview

### Proxmox VM Inventory

| VM ID | Name | OS | Role | RAM | Disk | Network |
|---|---|---|---|---|---|---|
| 100 | DC01 | Windows Server 2022 | Domain Controller | 7.91 GiB | 80GB | vmbr0 |
| 101 | windows-victim-01 | Windows 11 Pro 25H2 | Domain workstation | 6.00 GiB | 80GB | vmbr0 |
| 104 | Wazuh-Server-Ubuntu | Ubuntu 22.04 LTS | SIEM | — | — | vmbr0 |
| 105 | linux-victim-01 | Ubuntu 22.04 LTS | Linux target endpoint | 4.00 GiB | 40GB | vmbr0 |

### Network Configuration

All VMs sit on vmbr0 in the 10.0.0.0/24 subnet. The Proxmox node runs two Linux bridges:

| Bridge | CIDR | Purpose |
|---|---|---|
| vmbr0 | 192.168.1.50/24 | Primary LAN — lab traffic (physical NIC: enx8cae4c) |
| vmbr1 | — | Secondary bridge (physical NIC: eno2) |

DC01 static IP: **10.0.0.200**  
windows-victim-01 DHCP IP: **10.0.0.101**  
Default gateway: **10.0.0.1**

📸 `screenshots/proxmox-network-vmbr0-vmbr1-bridges.png`  
📸 `screenshots/proxmox-node-summary-all-vms-running.png`

---

## Domain Controller Build — DC01

### 1. VM Provisioning on Proxmox

Provisioned VM 100 (DC01) using the Windows Server 2022 Evaluation ISO with the following configuration:

| Setting | Value |
|---|---|
| VM ID | 100 |
| Name | DC01 |
| OS | Windows Server 2022 Standard Evaluation |
| BIOS | OVMF (UEFI) |
| Machine | pc-q35-10.1 |
| SCSI Controller | LSI 53C895A |
| RAM | 7.91 GiB |
| Disk | 80GB (sata0, local-lvm) |
| Network | VirtIO NIC on vmbr0, firewall enabled |
| EFI Disk | 4MB, ms-cert-2023, pre-enrolled keys |
| TPM State | v2.0 |

📸 `screenshots/proxmox-dc01-hardware-config-final.png`

### 2. VirtIO Driver Troubleshooting

During initial provisioning with VirtIO SCSI controller, Windows Server Setup showed a blank disk partition screen — the standard "no drives found" problem because Windows Server does not include VirtIO SCSI drivers natively. The installation could not detect the virtual disk.

**Resolution:** Switched the SCSI controller from VirtIO SCSI single to **LSI 53C895A**, which Windows Server supports natively without requiring a driver injection step. The VirtIO network adapter was retained as a separate driver and loaded successfully without issue.

Additional issues encountered during the build:

| Issue | Cause | Resolution |
|---|---|---|
| Blank disk partition screen | VirtIO SCSI not supported natively by Windows Server installer | Changed SCSI controller to LSI 53C895A |
| UEFI boot failure (BdsDxe errors) | EFI Storage not selected during VM creation wizard | Rebuilt VM with EFI storage configured before first boot |
| CD/DVD drive shown as undefined/orange | VirtIO ISO mounted on ide0 without proper media path | Re-attached ISO via Proxmox Edit CD/DVD Drive dialog |
| Load driver prompt — no drivers found | Browsing to Server ISO instead of VirtIO ISO | Correctly navigated to VirtIO ISO on second CD/DVD drive |

📸 `screenshots/windows-server-setup-blank-disk-no-drivers.png`  
📸 `screenshots/proxmox-dc01-uefi-boot-failure-bdsdxe.png`  
📸 `screenshots/proxmox-dc01-edit-cdvd-virtio-iso.png`  
📸 `screenshots/proxmox-create-vm-system-tab-scsi-controller-selection.png`

### 3. Windows Server 2022 Post-Install Baseline

After successful installation, documented the Server Manager baseline state before AD DS role installation:

| Property | Value |
|---|---|
| Computer Name | DC01 |
| Workgroup | WORKGROUP (pre-domain) |
| OS | Microsoft Windows Server 2022 Standard Evaluation |
| RAM | 7.89 GB |
| Disk | 79.37 GB |
| Defender Firewall | Public: On |
| Remote Desktop | Disabled |
| Remote Management | Enabled |
| Total Services | 206 |

📸 `screenshots/dc01-server-manager-local-server-properties-baseline.png`  
📸 `screenshots/dc01-server-manager-services-bpa-clean.png`  
📸 `screenshots/dc01-server-manager-roles-features-baseline.png`

### 4. Network Connectivity — VirtIO NIC

Confirmed network connectivity via the Red Hat VirtIO Ethernet Adapter. Initial state showed "Not connected" in Windows Settings due to the NIC driver loading after first boot. After driver initialization, DHCP assigned an address and connectivity was confirmed.

Ethernet Properties confirmed the adapter stack: Client for Microsoft Networks, File and Printer Sharing, QoS Packet Scheduler, IPv4, IPv6.

📸 `screenshots/dc01-network-status-not-connected.png`  
📸 `screenshots/dc01-ethernet-properties-virtio-nic.png`  
📸 `screenshots/dc01-server-manager-ethernet-dhcp-confirmed.png`

### 5. Static IP Assignment

Assigned a static IP to DC01 before promoting to domain controller. Dynamic IP assignment would break DNS resolution across the domain after promotion.

| Setting | Value |
|---|---|
| IPv4 Address | 10.0.0.200 |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 10.0.0.1 |
| DNS Server | 127.0.0.1 (self) |

📸 `screenshots/dc01-server-manager-static-ip-10-0-0-200-confirmed.png`

### 6. Active Directory Domain Services Installation

Installed the AD DS role via Server Manager → Add Roles and Features, then ran the Active Directory Domain Services Configuration Wizard to promote DC01 as the forest root domain controller.

**DNS delegation warning:** During the DNS Options step, the wizard displayed: *"A delegation for this DNS server cannot be created because the authoritative parent zone cannot be found."* This is expected behavior for a new forest root with no parent DNS zone — it was dismissed without action.

| Setting | Value |
|---|---|
| Forest | basil.local (new) |
| Domain | basil.local |
| Domain Controller | DC01 |
| Functional Level | Windows Server 2016 |
| DNS Server | Yes |
| Global Catalog | Yes |

📸 `screenshots/dc01-ad-ds-configuration-wizard-dns-options.png`  
📸 `screenshots/dc01-server-manager-domain-basil-local-confirmed.png`

---

## DNS Configuration

### 7. DNS Manager — Forward Lookup Zone

After domain promotion, DNS Manager was opened to verify the zone configuration. The Forward Lookup Zone for basil.local was automatically created and populated during AD DS promotion.

| Record | Type | Data |
|---|---|---|
| (same as parent folder) | SOA | dc01.basil.local |
| (same as parent folder) | NS | dc01.basil.local |
| (same as parent folder) | Host (A) | 10.0.0.200 |
| dc01 | Host (A) | 10.0.0.200 |

📸 `screenshots/dc01-dns-manager-basil-local-forward-lookup-zone.png`

### 8. DNS Resolution Validation

Verified DNS resolution from windows-victim-01 after domain join using `nslookup`:
