# Mini-SOC in a Box — Week 1 Engineering Journal

**Environment:** VMware Workstation · Ubuntu Server 24.04 LTS · Windows endpoint · Wazuh 4.14.7 · Docker · Sysmon 15.22  
**Scope:** Recorded implementation and troubleshooting carried out during Week 1. These entries are engineering case notes, not invented daily timestamps.

## Entry 1 — Building the Virtual Lab Infrastructure

**Objective:** Provision three VMware virtual machines with enough capacity for a single-node SIEM and two monitored endpoints.

**Challenge:** The Ubuntu installer initially presented an LVM layout that did not make the intended virtual-disk capacity available in the root filesystem.

**Action:** Reviewed the partitioning summary, changed to a simpler non-LVM full-disk layout, and verified the resulting system configuration. Configured `siem-host` (16 GB RAM, 6 vCPU, 250 GB disk), `ubuntu-web` (2 GB, 2 vCPU, 40 GB) and `win-client` (4 GB, 2 vCPU, 80 GB).

**Outcome / lesson:** Infrastructure was provisioned with explicit resource allocations. Virtual-disk size and filesystem allocation must be checked separately.

## Entry 2 — Resolving Host-Only Networking and Addressing Issues

**Objective:** Connect the three lab machines within an isolated network while retaining installation-time internet access where necessary.

**Challenge:** A previous VMware lab network overlapped with the planned network settings. After switching Ubuntu from NAT to host-only, automatic DHCP configuration was not appropriate because DHCP was disabled for the lab network.

**Action:** Established a dedicated `VMnet9` host-only subnet `10.10.10.0/24` and configured static addresses: `10.10.10.10` (`siem-host`), `10.10.10.20` (`ubuntu-web`) and `10.10.10.30` (`win-client`). Verified connectivity and kept NAT only where needed during installation.

**Outcome / lesson:** Predictable IP addressing simplified Wazuh agent onboarding and testing. Host-only networking is not equivalent to full isolation until additional adapters and default routes are inspected.

## Entry 3 — Correcting Timestamp Differences and Preparing Docker

**Category:** Time Synchronization / Log Integrity / Platform Readiness

**Objective:** Make timestamps trustworthy for cross-host event correlation and prepare the SIEM host to run Wazuh.

**Challenge:** During setup, system time and displayed log timestamps needed attention. The SIEM host initially reported that its system clock was not synchronized. Differences between the VM's timezone and the timezone/display settings used when viewing events could make an event appear outside the selected Wazuh search window.

**Investigation and actions:**
1. Checked the Ubuntu time and synchronization status with `timedatectl`.
2. Configured the SIEM host for the `Asia/Baku` timezone and enabled NTP synchronization; subsequently confirmed that time synchronization was active.
3. Interpreted event timestamps in the context of system time, timezone and the Wazuh Dashboard date-range filter rather than assuming a missing result meant data loss.
4. Installed Docker Engine and the Docker Compose plugin from Docker's Ubuntu repository.
5. Persistently set the indexer-required kernel parameter `vm.max_map_count=262144` in `/etc/sysctl.d/99-wazuh.conf`.

**Outcome:** The SIEM host had synchronized time and the Docker/kernel prerequisites required for the Wazuh stack. The exact offset observed between every VM and the Dashboard was not separately recorded, so no numerical time correction is claimed.

**SOC lesson:** In a multi-endpoint SOC, apparent gaps in a timeline can be caused by NTP drift, timezone conversion or an incorrect search range. Verify the clock, the timezone and the query window before concluding that events were dropped.

## Entry 4 — Deploying Wazuh and Restoring Remote Access

**Objective:** Deploy the Wazuh 4.14.7 Manager, Indexer and Dashboard as a single-node Docker Compose stack.

**Challenge:** Initial SSH attempts to `siem-host` returned `Connection refused`. A working network connection did not mean that `sshd` was active and accepting connections.

**Action:** Started/enabled SSH, checked reachability, generated Wazuh indexer certificates, launched the three containers, and signed in to the Dashboard over HTTPS. Changed the initial Wazuh administrator password without documenting the actual secret.

**Outcome / lesson:** All three Wazuh components became accessible. Troubleshooting should separate network reachability from application/service availability.

## Entry 5 — Distinguishing Event Collection from Alert Generation

**Category:** SOC Telemetry Validation

**Objective:** Ensure that Windows Sysmon process-creation activity travelled all the way from the endpoint into Wazuh.

**Challenge:** Sysmon Event ID `1` appeared locally on Windows, but ordinary launches of benign processes such as Notepad did not necessarily trigger a default Wazuh security alert.

**Investigation and actions:**
1. Confirmed Sysmon was installed and producing Process Creation events.
2. Configured the Windows Wazuh agent to read `Microsoft-Windows-Sysmon/Operational` using the `eventchannel` log format.
3. Verified that events could be collected through the Wazuh telemetry pipeline.
4. Differentiated **log collection** from **rule matching**: not every collected record generates an alert visible in the alert-oriented Dashboard view.

**Outcome:** The agent-to-manager telemetry path was verified, which provided the basis for custom detection engineering.

**SOC lesson:** When an expected alert is missing, check each stage separately: endpoint event → agent collection → manager decoding → rule match → alert index → Dashboard filter.

**Evidence:** [`03-sysmon-process-creation.png`](evidence/03-sysmon-process-creation.png).

## Entry 6 — Engineering and Debugging a Custom Sysmon Detection Rule

**Category:** Detection Engineering / XML Validation

**Objective:** Create a testable alert for Windows Notepad process execution using Sysmon Event ID `1`.

**Challenge:** The first version of the Wazuh local rule failed XML validation because a `group` element had been placed incorrectly. After fixing the syntax, a fresh event was needed to demonstrate an actual rule match.

**Investigation and actions:**
1. Worked with the manager's persistent `local_rules.xml` rather than relying on temporary edits inside the container.
2. Corrected the XML structure and validated the Wazuh rule configuration.
3. Restarted the manager after successful validation and launched Notepad on `win-client`.
4. Located the matching alert in Wazuh Threat Hunting.

**Confirmed detection:** Rule ID **`100100`**, level **`5`**, description **`Mini-SOC: Notepad process execution detected`**.

**Outcome / lesson:** A complete custom detection was demonstrated, from Sysmon event creation to a searchable Wazuh alert. Validate XML and trigger a new event after rule changes; historical raw events are not necessarily reprocessed automatically.

**Evidence:** [`08-windows-process-creation.png`](evidence/08-windows-process-creation.png).

## Entry 7 — Validating SIEM Persistence and Recovery

**Category:** Service Recovery / Data Persistence

**Objective:** Verify that restarting containers or rebooting `siem-host` would not erase the Wazuh deployment or agent registrations.

**Test performed:**
1. Ran `sudo docker compose stop` and confirmed that Manager, Indexer and Dashboard were stopped.
2. Used `sudo docker compose start` and checked that all three containers returned to the `Up` state.
3. Searched for the previously generated `rule.id: 100100` alert; the earlier alert was still retrievable after container restart.
4. Rebooted `siem-host` and checked `sudo docker compose ps`; all three containers were running.
5. Verified that the Dashboard displayed **two Active agents**, one Linux and one Windows, with no disconnected agents.

**Outcome:** Container stop/start preserved the existing alert data. The host reboot also preserved the deployment and agent registrations. The supplied evidence does not yet independently show the old alert being queried *after* the full host reboot, so that narrower check remains open.

**SOC lesson:** Container status, persistent stored alerts and live agent connectivity are three different recovery signals; each should be verified. Routine shutdown must not use `docker compose down -v`, because it removes named volumes.

**Evidence:** [`11-wazuh-containers-stopped.png`](evidence/11-wazuh-containers-stopped.png), [`12-wazuh-containers-restarted.png`](evidence/12-wazuh-containers-restarted.png), [`13-alert-persistence-test.png`](evidence/13-alert-persistence-test.png), [`14-wazuh-after-reboot.png`](evidence/14-wazuh-after-reboot.png), [`15-agents-after-reboot.png`](evidence/15-agents-after-reboot.png).

## Entry 8 — Enforcing Network Isolation and Reducing Attack Surface

**Category:** Network Hardening / Segmentation

**Objective:** Ensure that only `siem-host` retained its intended NAT access, while monitored endpoints stayed inside the isolated VMnet9 subnet.

**Challenge:** After installing software, `ubuntu-web` and `win-client` still had temporary NAT adapters. Route-table checks exposed default routes via the VMware NAT gateway `192.168.209.2`.

**Investigation and actions:**
1. Inspected Linux routes with `ip route` and Windows IPv4 routes with `route print -4`.
2. Removed the temporary NAT adapters from both endpoint VMs in VMware settings.
3. Rechecked routing: Linux showed only its connected `10.10.10.0/24` network; Windows had connected/loopback routes but no default route.
4. Confirmed that both Wazuh agents remained **Active** after isolation.

**Outcome:** The monitored endpoints no longer had an ordinary route to the internet. The SIEM remained reachable on `10.10.10.10` via the host-only lab network.

**SOC lesson:** Network isolation is a property of actual interfaces and routing, not merely a VMware label. Verify the resulting routes and application connectivity after every adapter change.

**Evidence:** [`16-windows-network-isolation.png`](evidence/16-windows-network-isolation.png), [`17-linux-network-isolation.png`](evidence/17-linux-network-isolation.png).

## Remaining Validation and Documentation

- Verify that the Linux failed-login evidence corresponds specifically to an **SSH** failure, not only a general PAM failure.
- Save an old-alert query screenshot taken **after** a full `siem-host` reboot.
- Verify clock/NTP status on all three VMs, not just the SIEM host.
- **Completed:** Created VMware final snapshots named `w1-done` for each VM. The three Snapshot Manager screenshots show both `w1-clean-install` and `w1-done`: [win-client](evidence/18-win-client-w1-done-snapshot.png), [ubuntu-web](evidence/19-ubuntu-web-w1-done-snapshot.png), and [siem-host](evidence/20-siem-host-w1-done-snapshot.png).
- Manually review every public screenshot for secrets and complete a Git secret check.
- Record Windows evaluation expiration, if the installed edition is an Evaluation build.
