# Week 1 — Evidence Screenshots

These screenshots were captured from the actual Mini-SOC lab during 15 October 2026 troubleshooting and verification. Each link opens a **real screenshot included in this folder**. The screenshots use consecutive, zero-padded filenames (01–17) ordered by the verification workflow, not by capture time. Screenshots are not simulated or recreated.

**Important privacy check before publishing:** Screenshots show local lab IP addresses, hostnames, usernames and dashboard details. No password or private key was intentionally copied into the repo, but visually review **every image** before making a GitHub commit. Image text is not protected by `.gitignore`.

## Installed and verified evidence

| Screenshot | What it proves |
|---|---|
| [View image](01-wazuh-docker-containers.png) — `01-wazuh-docker-containers.png` | Wazuh Manager, Indexer, and Dashboard containers running. |
| [View image](02-two-active-agents.png) — `02-two-active-agents.png` | Both Windows and Linux agents in Active state. |
| [View image](03-sysmon-process-creation.png) — `03-sysmon-process-creation.png` | Sysmon event documentation received and opened in Wazuh. |
| [View image](04-apache-web-server.png) — `04-apache-web-server.png` | Apache2 default webpage reachable in host browser. |
| [View image](05-failed-login.png) — `05-failed-login.png` | Wazuh PAM login failed alert (rule 5503). Confirm whether specifically SSH. |
| [View image](06-sudo-command.png) — `06-sudo-command.png` | Wazuh successful sudo to ROOT alert (rule 5402). |
| [View image](07-new-linux-user.png) — `07-new-linux-user.png` | Wazuh Linux new user alert (rule 5902). |
| [View image](08-windows-process-creation.png) — `08-windows-process-creation.png` | Custom Notepad alert (rule 100100) in Wazuh. |
| [View image](09-failed-windows-logon.png) — `09-failed-windows-logon.png` | Windows logon failure alert (rule 60122). |
| [View image](10-wazuh-memory-usage.png) — `10-wazuh-memory-usage.png` | Docker stats showing idle memory for three Wazuh components. |
| [View image](11-wazuh-containers-stopped.png) — `11-wazuh-containers-stopped.png` | Docker compose ps -a after stop; all services exited. |
| [View image](12-wazuh-containers-restarted.png) — `12-wazuh-containers-restarted.png` | Docker compose ps after restarting three services. |
| [View image](13-alert-persistence-test.png) — `13-alert-persistence-test.png` | Rule 100100 remained searchable after Docker stop/start. |
| [View image](14-wazuh-after-reboot.png) — `14-wazuh-after-reboot.png` | All Docker services Up after SIEM host reboot. |
| [View image](15-agents-after-reboot.png) — `15-agents-after-reboot.png` | Both agents Active after SIEM host reboot. |
| [View image](16-windows-network-isolation.png) — `16-windows-network-isolation.png` | Windows routes show no default gateway after NAT removal. |
| [View image](17-linux-network-isolation.png) — `17-linux-network-isolation.png` | Linux endpoint routes only over isolated 10.10.10.0/24. |

## Detection and telemetry findings

| Scenario | Wazuh rule | Severity | Image |
|---|---|---:|---|
| Linux failed authentication (PAM; SSH-specific verification remains open) | `5503` | 5 | [Failed login](05-failed-login.png) |
| Linux successful sudo to ROOT | `5402` | 3 | [Sudo command](06-sudo-command.png) |
| Linux new user creation | `5902` | 8 | [New Linux user](07-new-linux-user.png) |
| Windows Sysmon process execution — Notepad custom rule | `100100` | 5 | [Process creation](08-windows-process-creation.png) |
| Windows failed logon | `60122` | 5 | [Failed Windows logon](09-failed-windows-logon.png) |

## Persistence tests

- [Containers stopped](11-wazuh-containers-stopped.png) → [containers restarted](12-wazuh-containers-restarted.png) → [existing alert found](13-alert-persistence-test.png): demonstrates data preservation across a **Docker stop/start**.
- [Containers after host reboot](14-wazuh-after-reboot.png) and [agents after host reboot](15-agents-after-reboot.png): demonstrates service recovery and retained agent registration. **A separate screenshot showing an old alert after the full VM reboot is still recommended** for strict evidence of stored alert persistence across reboot.

## Network isolation tests

- [Windows route table](16-windows-network-isolation.png): no default NAT route.
- [Linux route table](17-linux-network-isolation.png): only the isolated subnet route.

## Evidence still to collect

- Screenshot confirming a failed **SSH** authentication event specifically, including its source/service or event fields (the PAM alert alone does not unambiguously prove SSH).
- Final VM snapshot names `w1-done` for all three VMs; also retain screenshots of initial `w1-clean-install` snapshots if not yet in this folder.
- Windows evaluation expiry screenshot/date.
- Clocks and NTP verification on all three VMs.
- Git secret scan/check screenshot before publication.

## How to view screenshots on GitHub

Open this file on GitHub and click **View image** in the table. GitHub will display the linked PNG directly. Every linked PNG is packaged alongside this README.
