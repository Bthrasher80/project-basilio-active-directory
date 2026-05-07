# Wazuh SIEM Deployment: Docker POC to Multi-OS Bare-Metal Production Lab

**Project:** Project Basilio — SOC-Lab-Architecture  
**Date:** January–February 2026  
**Stage:** Stage 1 — Infrastructure (Wazuh Component)  
**Tools:** Wazuh 4.7.2 / 4.7.5 · Docker · Proxmox 9.1.2 · Ubuntu 22.04 LTS · Windows 11 Pro · pfSense  
**MITRE ATT&CK:** T1110 (Brute Force) · T1110.001 · T1021.004 · T1078 (Valid Accounts) · T1548.003 (Sudo/Sudoers) · T1531 (Account Access Removal) · T1485 (Data Destruction) · T1565.001 (Stored Data Manipulation) · T1070.004 (File Deletion)

---

## Objective

Deploy Wazuh as the SIEM/XDR layer of a physical SOC home lab, progressing from an initial Docker-based proof of concept to a production-style bare-metal installation monitoring a heterogeneous multi-OS endpoint environment. Validate detection capability across Linux and Windows endpoints using simulated attack techniques mapped to MITRE ATT&CK.

---

## Problem Statement

A SIEM is only credible if it can ingest telemetry from diverse endpoints, correlate events against known attack frameworks, and surface actionable alerts. A self-monitoring single-node deployment with no victim machines provides no real signal. This project builds toward a realistic detection environment: a dedicated Wazuh server monitoring two separate victim hosts — one Linux, one Windows — running inside a segmented Proxmox hypervisor network with pfSense controlling traffic between zones.

---

## Environment

| Component | Details |
|---|---|
| Wazuh Server | Dell Latitude 5400 (Ubuntu 22.04 LTS) — Proxmox VM 104 |
| Linux Victim | Proxmox VM 105 — Ubuntu 22.04 LTS — 4GB RAM, 2 vCPU, 40GB disk |
| Windows Victim | Proxmox VM 101 — Windows 11 Pro 10.0.26200.7840 — 6GB RAM, 2 vCPU, 80GB disk |
| Hypervisor | Dell Latitude 5411 running Proxmox VE 9.1.2 (node: projectbasilio-proxmox) |
| Firewall | pfSense on VM 100 — controls inter-zone traffic |
| Network | 192.168.1.0/24 lab segment via vmbr0 |
| Wazuh Version | 4.7.2 (Docker POC) → 4.7.5 (bare-metal production) |

---

## Phase 1 — Docker Proof of Concept

### What I Did

Stood up Wazuh using the official `wazuh-docker` single-node deployment on the Ubuntu server. Installed Docker CE 29.2.1 and Docker Compose v5.0.2, enabled the Docker daemon via systemd, cloned the `wazuh/wazuh-docker` repository, and attempted deployment.

**Evidence:** `PB-WAZUH-01_Docker_Runtime_Installed_and_Verified.png`

### Deployment Errors Encountered and Resolved

**Error 1 — Image Tag Resolution Failure:**  
Running `docker compose up -d` against the default `docker-compose.yml` failed because `wazuh/wazuh-indexer:5.0.0` did not exist in Docker Hub at the time of deployment. The compose file referenced a version tag that had not yet been published.

Resolution: Pinned all three service images (`wazuh-manager`, `wazuh-indexer`, `wazuh-dashboard`) to the `4.7.2` tag in `docker-compose.yml`.

**Evidence:** `PB-WAZUH-02_Image_Tag_Resolution_Failure_5_0_0.png`

**Error 2 — Port 443 Conflict:**  
After pinning tags, the dashboard container failed to start because port 443 was already bound by a `node` process (PID 718). Identified the conflict with `ss -tulnp | grep ':443'`, stopped the conflicting service with `systemctl stop wazuh-dashboard`, killed the orphan process, verified the port was free, then brought all containers down and back up cleanly.

**Evidence:** `PB-WAZUH-02b_Port_443_Conflict_Resolution.png`

**Error 3 — Port 1514 Conflict:**  
A subsequent attempt failed when the manager container could not bind `0.0.0.0:1514/tcp` because a prior run had left a wazuh-manager process holding the port. Resolved by running `docker compose down` to clear all containers and volumes, then `docker compose up -d` for a clean start.

**Evidence:** `PB-WAZUH-04b_Port_1514_Manager_Bind_Conflict.png`

### Phase 1 Validation

Dashboard accessible at `https://192.168.1.170`. One agent enrolled (wazuh-server self-monitoring via loopback — expected behavior for single-node Docker deployment). Sudo to ROOT and shadow file read triggered T1548.003 and T1078 alerts, confirming the detection pipeline was functional.

**Evidence:** `PB-WAZUH-03_Dashboard_Login_Page.png` · `PB-WAZUH-04_Dashboard_Overview_Agent_Active.png` · `PB-WAZUH-06_Agents_Overview_Server_Active.png` · `PB-WAZUH-07_Agent_Service_Status_Running.png` · `PB-WAZUH-08_Security_Events_Dashboard_Privilege_Escalation.png` · `PB-WAZUH-09_Privilege_Escalation_Detected_MITRE.png` · `PB-WAZUH-10_Root_Access_Shadow_File_Read.png`

---

## Phase 2 — Bare-Metal Production Install with Linux Victim

### Why the Rebuild

The Docker deployment validated that the stack worked. For production-quality lab use — persistent storage, performance at scale, and a real separate endpoint — a bare-metal install on the dedicated Ubuntu VM was the correct approach.

### What I Did

Downloaded the Wazuh 4.7.5 unified install script, made it executable, and ran `sudo ./wazuh-install.sh -a` for an all-in-one deployment. The installer provisioned the Wazuh indexer, manager, Filebeat, and dashboard sequentially — all four services confirmed started within approximately four minutes.

**Evidence:** `Wazuh_4_7_5_full_installation_log___indexer__manager__Filebeat__dashboard_all_installed_successfully.png` · `wazuh-indexer_service_activerunning__Java_process_confirmed.png` · `Wazuh_server_IP_address___192_168_50_10624_via_ip_a.png`

### Linux Victim Agent Enrollment

Deployed Wazuh agent v4.7.2 on VM 105 (linux-victim-01 at 192.168.1.173). Configured `/var/ossec/etc/ossec.conf` with the server address `192.168.1.170` on port 1514 via TCP. Restarted the agent service and confirmed all five daemons running: `wazuh-execd`, `wazuh-agentd`, `wazuh-syscheckd`, `wazuh-logcollector`, `wazuh-modulesd`.

**Evidence:** `Wazuh_agent_ossec_conf___server_address_192_168_1_170__port_1514_configured.png` · `Wazuh_agent_active_and_running_on_linux-victim-01___all_daemons_started.png` · `Wazuh_dashboard___2_agents_active__wazuh-server___linux-victim-01_.png`

### Detection Validation — Linux

**SSH Brute Force (T1110 / T1110.001 / T1021.004):**  
Ran repeated failed SSH authentication attempts against linux-victim-01 from 192.168.1.154. Wazuh detected the pattern via `/var/log/auth.log` ingestion and fired rule 2502 (level 10 — "User missed the password more than one time") and rule 5710 (level 5 — "sshd: Attempt to login using a non-existent user"). Alert detail showed source IP 192.168.1.154, destination user `bthrasher80`, decoder `sshd`, and compliance mappings across GDPR IV_35.7.d/IV_32.2, HIPAA 164.312.b, GPG 13 7.1, and TSC CC6.1/CC6.8/CC7.2/CC7.3.

**Evidence:** `Wazuh_security_alerts___SSH_brute_force_attempts__T1110T1021_Credential_Access.png` · `Wazuh_alert_detail___brute_force_event__source_IP_192_168_1_154__full_log__compliance_mappings.png` · `Wazuh_rule_5710_detail___sshd_invalid_user_rule__compliance_mappings.png`

**File Integrity Monitoring (T1565.001 / T1070.004 / T1485):**  
Created, modified, and deleted `/tmp/basilio-test.sh` on linux-victim-01 to validate FIM. Wazuh syscheck detected all three actions (added → rule 554 level 5, modified → rule 550 level 7, deleted → rule 553 level 7) and surfaced them in the Integrity Monitoring dashboard within the same minute.

**Evidence:** `Wazuh_FIM_events___basilio-test_sh_added__modified__deleted_on_linux-victim-01.png` · `Wazuh_FIM_integrity_monitoring_dashboard___alerts_by_action_over_time.png`

**Login Activity (T1078 / T1548.003):**  
Interactive login sessions and sudo escalation to ROOT on linux-victim-01 fired T1078 (Valid Accounts — rule 5501/5502) and T1548.003 (Successful sudo to ROOT — rule 5402) consistently across multiple sessions.

**Evidence:** `Wazuh_security_events___login_sessions__sudo_to_root__T1078T1548_MITRE_tags.png` · `Wazuh_security_events_on_linux-victim-01___brute_force_SSH__FIM_file_deleted__T1110T1485.png`

---

## Phase 3 — Windows Endpoint Added, Full Three-Agent Lab

### Windows VM Provisioning (Proxmox VM 101)

Built windows-victim-01 in Proxmox with the following configuration:

- OS: Windows 11 Pro 10.0.26200.7840 (UEFI/OVMF, TPM v2.0)
- RAM: 6.00 GiB · CPU: 2 cores (x86-64-v2-AES) · Disk: 80GB (VirtIO SCSI)
- Network: VirtIO NIC on vmbr0 with firewall enabled (192.168.1.174)
- Installation required mounting both the Windows 11 ISO and the VirtIO driver ISO simultaneously to resolve driver detection during setup

Troubleshooting note: An initial hardware config attempt resulted in an undefined CD/DVD Drive (ide0) showing in orange in Proxmox — caused by a duplicate drive entry from an earlier ISO mount that was not properly cleaned up. Resolved by removing the undefined entry and correctly assigning the VirtIO ISO to ide0 and the Windows ISO to ide2.

**Evidence:** `wz-proxmox-node-summary-all-vms-running.png` · `wz-proxmox-vm101-windows-victim-01-hardware-config.png` · `wz-proxmox-vm105-linux-victim-01-hardware-config.png` · `wz-proxmox-vm101-windows-victim-cdvd-undefined-orange-virtio-conflict.png` · `wz-proxmox-vm101-windows-victim-virtio-and-win11-iso-both-mounted.png`

### Windows Agent Enrollment

Installed Wazuh agent on windows-victim-01 and confirmed enrollment as agent 003 at 192.168.1.174. The three-agent environment — wazuh-server (001), linux-victim-01 (002), windows-victim-01 (003) — was fully active at 100% agent coverage.

Troubleshooting note: Attempted `net start wazuh` and `net start WazuhSvc` from PowerShell — both returned "service name is invalid." Correct service management on Windows is handled through the Wazuh agent tray application or Services console, not PowerShell `net start` with those aliases. Agent was confirmed operational via the dashboard.

**Evidence:** `wz-dashboard-agents-3-active-all-online.png` · `wz-dashboard-agents-3-active-clean-view.png` · `wz-modules-overview-all-categories-3-agents.png` · `wz-windows-victim-01-powershell-wazuh-service-not-found.png`

### Detection Validation — Windows

**Logon Failure (EID 4625 — T1078 / T1531):**  
Simulated failed logon attempts against windows-victim-01 using an invalid username ("testuser"). Wazuh rule 60122 fired (level 5 — "Logon failure - Unknown user or bad password"), triggered by Windows Security Event IDs 529 and 4625. Alert detail captured in Table, JSON, and Rule views showing agent 003, `data.win.eventdata.logonProcessName: User32/seclogo`, `failureReason: %%2313`, `logonType: 2`, `processName: C:\Windows\System32\svchost.exe`, and MITRE mappings T1078/T1531. Compliance mappings include GDPR IV_32.2/IV_35.7.d, HIPAA 164.312.b, and GPG 13 7.1.

The security events dashboard for agent 003 showed 626 total alerts with 5 authentication failures, 133 authentication successes, and top alert category "Logon failure - Unknown user or bad password" confirming active Windows Security Log ingestion.

**Evidence:** `wz-windows-victim-01-security-events-dashboard-626-total.png` · `wz-windows-victim-01-security-events-top5-alerts-logon-failures.png` · `wz-windows-victim-01-alert-60122-logon-failure-table-raw-fields.png` · `wz-windows-victim-01-alert-60122-logon-failure-json-raw.png` · `wz-windows-victim-01-rule-60122-logon-failure-compliance-mapping.png` · `wz-security-alerts-table-windows-victim-01-mixed-events.png`

**File Integrity Monitoring — Windows (T1565.001 / T1070.004):**  
Ran the following PowerShell sequence on windows-victim-01 to exercise FIM on the monitored `C:\Windows\Temp` path:

```powershell
echo "Test Integrity Monitoring" > C:\Windows\Temp\fim_test.txt
echo "Modified content" >> C:\Windows\Temp\fim_test.txt
del C:\Windows\Temp\fim_test.txt
```

All three syscheck events (added, modified, deleted) were captured in the Integrity Monitoring dashboard. The 7-day FIM view across all agents showed windows-victim-01 generating the highest volume of file change events (registry modifications dominate Windows FIM), with linux-victim-01 and wazuh-server contributing file system changes. Top 5 users showed SYSTEM (63 events) and Administrators (33 events) on agent 003, and `_apt` (85) and `root` (77) on agent 002.

**Evidence:** `wz-windows-victim-01-powershell-fim-test-create-modify-delete.png` · `wz-integrity-monitoring-dashboard-7day-fim-alerts.png` · `wz-integrity-monitoring-rule-distribution-top5-users.png`

---

## Key Findings

- Wazuh 4.7.x Docker images for the `5.0.0` tag were not yet published at the time of lab build — pinning to `4.7.2` resolved deployment. Version tag stability is a real operational concern for any Docker-based SIEM deployment.
- Port conflicts on 443 and 1514 are common in environments where Wazuh was previously installed natively. Identifying conflicting processes with `ss -tulnp` and using `systemctl` to stop services cleanly prevents corrupt container state.
- Rule 5710 (invalid SSH user) fires at level 5 individually; rule 2502 escalates to level 10 when the brute force threshold is crossed — demonstrating Wazuh's correlation logic layering child rules on parent rule groups.
- Windows Security Event ID 4625 maps to Wazuh rule 60122, which carries MITRE T1078 and T1531 — both relevant to credential access triage in a SOC context.
- The `wazuh-server` agent self-monitors via loopback (127.0.0.1) as expected in a single-node deployment. External endpoints report via their LAN IPs. This is not a misconfiguration.
- The `/var/ossec/bin/` directory on the bare-metal install contains: `agent-auth`, `manage_agents`, `wazuh-agentd`, `wazuh-control`, `wazuh-execd`, `wazuh-logcollector`, `wazuh-modulesd`, `wazuh-syscheckd`. Useful reference for manual agent management and troubleshooting.

---

## Security and SOC Relevance

In a production SOC, this deployment pattern maps directly to the agent enrollment and tuning workflow a SOC analyst or detection engineer would maintain. The key skills demonstrated — standing up a SIEM, enrolling heterogeneous endpoints, validating detection rules fire on known-bad activity, reading raw alert JSON to trace field values back to log source — are exactly what is tested in SOC Analyst L1 and Vulnerability Management Analyst interviews.

Rule 60122 with EID 4625 is a real-world detection that fires in every Windows enterprise environment. Being able to explain the alert lifecycle — Windows generates Event 4625 → Wazuh agent collects via the Windows Event Log channel → Filebeat ships to the indexer → rule 60122 matches pattern `^529$|^4625$` → alert surfaces with T1078/T1531 MITRE tags and compliance mappings — is a concrete interview answer.

**Interview formula:**  
"I demonstrated SIEM deployment and alert triage when I stood up Wazuh 4.7.5 bare-metal, enrolled a Windows 11 and a Linux Ubuntu endpoint in a Proxmox lab environment, and validated detection of SSH brute force (T1110, rule 2502 escalating to level 10) and Windows logon failure (EID 4625, rule 60122, T1078/T1531) with full compliance mapping output — documented at github.com/Bthrasher80/SOC-Lab-Architecture."

---

## Evidence Captured

All screenshots are in `/screenshots` in the repository. Files follow the `wz-` naming convention for Wazuh artifacts and `PB-WAZUH-` for Docker POC phase artifacts.

**Phase 1 — Docker POC (11 files):**
- `PB-WAZUH-01_Docker_Runtime_Installed_and_Verified.png`
- `PB-WAZUH-02_Image_Tag_Resolution_Failure_5_0_0.png`
- `PB-WAZUH-02b_Port_443_Conflict_Resolution.png`
- `PB-WAZUH-03_Dashboard_Login_Page.png`
- `PB-WAZUH-04_Dashboard_Overview_Agent_Active.png`
- `PB-WAZUH-04b_Port_1514_Manager_Bind_Conflict.png`
- `PB-WAZUH-06_Agents_Overview_Server_Active.png`
- `PB-WAZUH-07_Agent_Service_Status_Running.png`
- `PB-WAZUH-08_Security_Events_Dashboard_Privilege_Escalation.png`
- `PB-WAZUH-09_Privilege_Escalation_Detected_MITRE.png`
- `PB-WAZUH-10_Root_Access_Shadow_File_Read.png`

**Phase 2 — Bare-Metal Linux (13 files):**
- `Wazuh_4_7_5_full_installation_log___indexer__manager__Filebeat__dashboard_all_installed_successfully.png`
- `wazuh-indexer_service_activerunning__Java_process_confirmed.png`
- `Wazuh_server_IP_address___192_168_50_10624_via_ip_a.png`
- `Wazuh_agent_ossec_conf___server_address_192_168_1_170__port_1514_configured.png`
- `Wazuh_agent_active_and_running_on_linux-victim-01___all_daemons_started.png`
- `Wazuh_dashboard___2_agents_active__wazuh-server___linux-victim-01_.png`
- `Wazuh_security_alerts___SSH_brute_force_attempts__T1110T1021_Credential_Access.png`
- `Wazuh_alert_detail___brute_force_event__source_IP_192_168_1_154__full_log__compliance_mappings.png`
- `Wazuh_rule_5710_detail___sshd_invalid_user_rule__compliance_mappings.png`
- `Wazuh_FIM_events___basilio-test_sh_added__modified__deleted_on_linux-victim-01.png`
- `Wazuh_FIM_integrity_monitoring_dashboard___alerts_by_action_over_time.png`
- `Wazuh_security_events___login_sessions__sudo_to_root__T1078T1548_MITRE_tags.png`
- `Wazuh_security_events_on_linux-victim-01___brute_force_SSH__FIM_file_deleted__T1110T1485.png`

**Phase 3 — Windows Endpoint / Full Lab (20 files):**
- `wz-proxmox-node-summary-all-vms-running.png`
- `wz-proxmox-vm101-windows-victim-01-hardware-config.png`
- `wz-proxmox-vm105-linux-victim-01-hardware-config.png`
- `wz-proxmox-vm101-windows-victim-cdvd-undefined-orange-virtio-conflict.png`
- `wz-proxmox-vm101-windows-victim-virtio-and-win11-iso-both-mounted.png`
- `wz-dashboard-agents-3-active-all-online.png`
- `wz-dashboard-agents-3-active-clean-view.png`
- `wz-modules-overview-all-categories-3-agents.png`
- `wz-wazuh-server-ossec-bin-directory-contents.png`
- `wz-windows-victim-01-powershell-wazuh-service-not-found.png`
- `wz-windows-victim-01-security-events-dashboard-626-total.png`
- `wz-windows-victim-01-security-events-top5-alerts-logon-failures.png`
- `wz-windows-victim-01-alert-60122-logon-failure-table-raw-fields.png`
- `wz-windows-victim-01-alert-60122-logon-failure-localhost-user32.png`
- `wz-windows-victim-01-alert-60122-logon-failure-json-raw.png`
- `wz-windows-victim-01-rule-60122-logon-failure-compliance-mapping.png`
- `wz-security-alerts-table-windows-victim-01-mixed-events.png`
- `wz-windows-victim-01-powershell-fim-test-create-modify-delete.png`
- `wz-integrity-monitoring-dashboard-7day-fim-alerts.png`
- `wz-integrity-monitoring-rule-distribution-top5-users.png`

---

## Challenges and How I Solved Them

1. **Docker image tag 5.0.0 not found** — pinned all services to `4.7.2` in `docker-compose.yml`
2. **Port 443 conflict with orphan node process** — identified with `ss -tulnp`, killed PID, verified port free before re-deploying
3. **Port 1514 bind failure** — `docker compose down` to clear all containers, clean `up -d`
4. **Windows VM no storage during install** — VirtIO drivers required a second ISO mounted during Windows setup; both ISOs had to be present simultaneously
5. **Duplicate undefined CD/DVD drive in Proxmox** — removed the orange undefined entry, correctly reassigned ISOs to ide0 and ide2
6. **`net start wazuh` failing on Windows** — service name alias doesn't match; Windows agent managed via tray app or Services console

---

## Lessons Learned

- Docker deployments are fast for POC but fragile at version boundaries — bare-metal is the right choice for a persistent lab
- `ss -tulnp` is the correct tool for identifying what's holding a port on Linux; faster than netstat in modern Ubuntu
- Wazuh's rule hierarchy — parent rules group by decoder/program, child rules escalate on frequency or pattern match — is worth reading in `0095-sshd_rules.xml` to understand how brute force detection actually works
- Windows Security Event ID 4625 is not a single-field match; Wazuh decodes the entire Windows eventdata block into structured fields (`failureReason`, `logonType`, `processName`) before rule matching — this is why the JSON view is more informative than the table view for Windows events
- VirtIO drivers are required for Proxmox Windows VMs to detect storage during installation; without the driver ISO mounted at install time, the Windows installer shows no available disks

---

## Next Steps

- Add Sysmon to windows-victim-01 for process creation (EID 1), network connection (EID 3), and LSASS access (EID 10) telemetry
- Configure custom Wazuh rules for Sysmon events to enable detection of PowerShell encoded commands (T1059.001) and scheduled task creation (T1053.005)
- Deploy Splunk on the dedicated Dell 7480 and configure Universal Forwarders on both victim machines — Stage 2 of the build plan

---

## Status: COMPLETE
