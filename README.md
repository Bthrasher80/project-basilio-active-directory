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
- Join windows-victim-01 to the basil.local domain and validate domain user authentication
- Configure Group Policy and document the audit policy baseline

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

All VMs sit on vmbr0 in the 10.0.0.0/24 subnet.

| Host | IP | Assignment |
|---|---|---|
| DC01 | 10.0.0.200 | Static |
| windows-victim-01 | 10.0.0.101 | DHCP |
| Default gateway | 10.0.0.1 | — |

---

## Domain Controller Build — DC01

### 1. VM Provisioning on Proxmox

Provisioned VM 100 (DC01) on Proxmox using the Windows Server 2022 Evaluation ISO. The Create VM wizard was used to configure OS type, system firmware, SCSI controller, disk, and network settings before first boot.

| Setting | Value |
|---|---|
| VM ID | 100 |
| Name | DC01 |
| OS | Windows Server 2022 Standard Evaluation |
| BIOS | OVMF (UEFI) |
| Machine | pc-q35-10.1 |
| SCSI Controller | LSI 53C895A (see troubleshooting below) |
| RAM | 7.91 GiB |
| Disk | 80GB (sata0, local-lvm) |
| Network | VirtIO NIC on vmbr0, firewall enabled |
| EFI Disk | 4MB, ms-cert-2023, pre-enrolled keys |
| TPM State | v2.0 |

📸 `screenshots/ad-proxmox-create-vm-dc01-os-tab-server-eval-iso.png`  
📸 `screenshots/ad-proxmox-create-vm-dc01-system-tab-uefi-virtio-scsi.png`  
📸 `screenshots/ad-proxmox-vm100-dc01-hardware-final-config-lsi-sata.png`

### 2. VirtIO Driver Troubleshooting

During initial provisioning with VirtIO SCSI controller, Windows Server Setup showed a blank disk partition screen — the standard "no drives found" problem because Windows Server does not include VirtIO SCSI drivers natively.

**Resolution:** Switched the SCSI controller from VirtIO SCSI single to **LSI 53C895A**, which Windows Server supports natively without requiring driver injection. The VirtIO network adapter was retained and loaded successfully.

Additional issues encountered:

| Issue | Cause | Resolution |
|---|---|---|
| Blank disk partition screen | VirtIO SCSI not supported natively by Windows Server installer | Changed SCSI controller to LSI 53C895A |
| UEFI boot failure (BdsDxe errors) | EFI Storage not configured during VM creation | Rebuilt VM with EFI storage configured before first boot |
| CD/DVD drive shown as undefined/orange | VirtIO ISO mounted without proper media path | Re-attached ISO via Proxmox Edit CD/DVD Drive dialog |
| Load driver prompt — no drivers found | Browsing to Server ISO instead of VirtIO ISO | Correctly navigated to VirtIO ISO on second CD/DVD drive |

📸 `screenshots/ad-dc01-uefi-boot-failure-no-bootable-device.png`  
📸 `screenshots/ad-proxmox-vm100-dc01-hardware-virtio-undefined-server-iso.png`  
📸 `screenshots/ad-proxmox-vm100-dc01-cdvd-edit-virtio-iso-selected.png`  
📸 `screenshots/ad-dc01-server-setup-installation-type-selection.png`  
📸 `screenshots/ad-dc01-server-setup-browse-folder-server-iso-mounted.png`

### 3. Windows Server 2022 Post-Install Baseline

After successful installation, documented the Server Manager baseline state before AD DS role installation.

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

📸 `screenshots/ad-dc01-server-manager-dashboard-first-boot.png`  
📸 `screenshots/ad-dc01-server-manager-local-server-properties-pre-promotion.png`

### 4. Network Connectivity — VirtIO NIC

Confirmed network connectivity via the Red Hat VirtIO Ethernet Adapter. After driver initialization, DHCP assigned an address and connectivity was confirmed. Ethernet Properties verified the full adapter stack: Client for Microsoft Networks, File and Printer Sharing, QoS Packet Scheduler, IPv4, IPv6.

📸 `screenshots/ad-dc01-ethernet-properties-virtio-nic-confirmed.png`  
📸 `screenshots/ad-dc01-server-manager-network-dhcp-active-pre-promotion.png`

### 5. Static IP Assignment

Assigned static IP 10.0.0.200 to DC01 before promotion. Dynamic IP assignment would break DNS resolution across the domain after the domain controller role was installed.

| Setting | Value |
|---|---|
| IPv4 Address | 10.0.0.200 |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 10.0.0.1 |
| DNS Server | 127.0.0.1 (self) |

### 6. Active Directory Domain Services Installation

Installed the AD DS role via Server Manager → Add Roles and Features, then ran the Active Directory Domain Services Configuration Wizard to promote DC01 as the forest root domain controller.

**DNS delegation warning:** During the DNS Options step, the wizard displayed: *"A delegation for this DNS server cannot be created because the authoritative parent zone cannot be found."* This is expected behavior for a new forest root with no parent DNS zone and was dismissed without action.

| Setting | Value |
|---|---|
| Forest | basil.local (new) |
| Domain | basil.local |
| Domain Controller | DC01 |
| Functional Level | Windows Server 2016 |
| DNS Server | Yes |
| Global Catalog | Yes |

📸 `screenshots/ad-dc01-addsconfig-wizard-dns-options-delegation-warning.png`  
📸 `screenshots/ad-dc01-server-manager-post-promotion-basil-local-domain.png`

---

## DNS Configuration

### 7. DNS Manager — Forward Lookup Zone

After domain promotion, DNS Manager confirmed the Forward Lookup Zone for basil.local was automatically created and populated.

| Record | Type | Data |
|---|---|---|
| (same as parent folder) | SOA | dc01.basil.local |
| (same as parent folder) | NS | dc01.basil.local |
| (same as parent folder) | Host (A) | 10.0.0.200 |
| dc01 | Host (A) | 10.0.0.200 |

📸 `screenshots/ad-dc01-dns-manager-basil-local-forward-lookup-zone.png`

---

## OU Structure and User/Group Configuration

### 8. Organizational Unit Design

Designed a role-based OU structure in Active Directory Users and Computers to reflect a realistic enterprise environment.

| OU | Purpose |
|---|---|
| Admins | Administrative user accounts |
| Domain Controllers | DC01 computer object |
| Servers | Member server objects |
| Users | Standard domain users |
| Users-OU | Extended user container |
| Workstations | Domain-joined endpoint objects |

📸 `screenshots/ad-dc01-aduc-basil-local-ou-structure.png`

### 9. User and Group Creation

Created SOC lab user accounts and a security group scoped to administrative functions.

- **SOC_Admins** — Security group in the Admins OU
- **Sarah Admin** — Member of SOC_Admins, sourced from basil.local/Users-OU
- **John Analyst** — Standard domain user (janalyst@basil.local), used to validate domain authentication from windows-victim-01

📸 `screenshots/ad-dc01-aduc-soc-admins-group-sarah-admin-member.png`

---

## Windows Endpoint Domain Join — windows-victim-01

### 10. Pre-Join Network Configuration

Before joining the domain, configured windows-victim-01 to point DNS at DC01 (10.0.0.200) so the endpoint could resolve basil.local. Verified the existing IP configuration via PowerShell.

📸 `screenshots/ad-windows-victim-01-ipconfig-pre-domain-join.png`  
📸 `screenshots/ad-windows-victim-01-tcpip-dns-set-to-dc01.png`

### 11. DNS Resolution Validation

Confirmed DNS resolution from windows-victim-01 using `nslookup` before proceeding with domain join:

```
nslookup basil.local
Server: UnKnown
Address: 10.0.0.200

Name: basil.local
Address: 10.0.0.200
```

📸 `screenshots/ad-windows-victim-01-nslookup-basil-local-resolves-dc01.png`

### 12. Domain Join and Authentication Validation

Joined windows-victim-01 to basil.local via System > About > Domain or workgroup. The "Welcome to the basil.local domain" confirmation dialog was captured at join time.

Post-join, logged in as domain user John Analyst (janalyst@basil.local) and confirmed full device name showing WINDOWS-VICTIM-01.basil.local in System > About — proving domain authentication is working end-to-end.

📸 `screenshots/ad-windows-victim-01-welcome-to-basil-local-domain-join.png`  
📸 `screenshots/ad-windows-victim-01-about-domain-joined-john-analyst.png`

---

## Group Policy Configuration

### 13. Group Policy Management

Opened Group Policy Management console on DC01. Confirmed DC01.basil.local as the baseline domain controller for the basil.local domain. The full OU structure was visible in the GPM tree: Admins, Domain Controllers, Servers, Users-OU, Workstations.

📸 `screenshots/ad-dc01-group-policy-management-basil-local-dc01-baseline.png`

### 14. SOC-Audit-Policy GPO Creation

Created a new GPO named **SOC-Audit-Policy** linked to the basil.local domain to enforce audit logging across domain-joined endpoints.

📸 `screenshots/ad-dc01-gpm-new-gpo-soc-audit-policy-creation.png`

### 15. Audit Policy Validation

Verified audit policy application on windows-victim-01 using `auditpol /get /category:*` from an elevated PowerShell session. Logon auditing confirmed as **Success and Failure**, proving the GPO was applied to the endpoint.

```
Category/Subcategory          Setting
Logon/Logoff
  Logon                       Success and Failure
```

📸 `screenshots/ad-windows-victim-01-auditpol-logon-success-failure-confirmed.png`

---

## Summary

| Milestone | Status |
|---|---|
| DC01 VM provisioned on Proxmox | ✅ |
| VirtIO driver issues resolved | ✅ |
| Windows Server 2022 installed | ✅ |
| AD DS role installed and DC01 promoted | ✅ |
| basil.local domain created | ✅ |
| DNS forward lookup zone configured | ✅ |
| OU structure designed and built | ✅ |
| User accounts and security groups created | ✅ |
| windows-victim-01 joined to basil.local | ✅ |
| Domain user authentication validated | ✅ |
| Group Policy Management configured | ✅ |
| SOC-Audit-Policy GPO created | ✅ |
| Audit policy applied and verified on endpoint | ✅ |

---

*Part of [Project Basilio](https://github.com/Bthrasher80) — a self-built SOC home lab covering network segmentation, SIEM deployment, Active Directory, and detection engineering.*
