# Mini-SOC in a Box — Week 1

**Blue Team Capstone | LabShop security monitoring lab**

## Project overview
LabShop is a fictional online shop with one web server and one staff endpoint. This project builds an isolated, reproducible mini Security Operations Center on VMware Workstation. Week 1 focuses on infrastructure, Wazuh single-node deployment, Linux and Windows agent telemetry, and validation of five test events.

## Architecture
See [network diagram](architecture/network-diagram.md) and [address table](architecture/address-table.md).

| VM | Role | Host-only IP | RAM | vCPU | Disk |
|---|---|---|---|---|---|
| siem-host | Ubuntu Server 24.04 LTS; Wazuh Docker single-node | 10.10.10.10 | 16 GB | 6 | 250 GB |
| ubuntu-web | Ubuntu Server 24.04 LTS; Apache; Wazuh agent | 10.10.10.20 | 2 GB | 2 | 40 GB |
| win-client | Windows 10/11; Wazuh agent; Sysmon | 10.10.10.30 | 4 GB | 2 | 80 GB |

VMnet9 is a dedicated host-only 10.10.10.0/24 network with DHCP disabled. siem-host also uses NAT for downloading images and updates. Endpoint NAT access was temporary for installation and subsequently removed. The physical VMware host reaches the Wazuh Dashboard at https://10.10.10.10 and the Apache page at http://10.10.10.20.

## Tools and versions
- VMware Workstation; Ubuntu Server 24.04 LTS; Windows 10/11 (verify exact edition).
- Wazuh Manager, Indexer, Dashboard and Linux/Windows agents: **4.14.7**.
- Docker Engine and Docker Compose plugin installed from Docker's official Ubuntu repository.
- Sysmon **15.22**, with SwiftOnSecurity community configuration.
- Apache2 on ubuntu-web.
- Host kernel parameter `vm.max_map_count=262144`, saved in `/etc/sysctl.d/99-wazuh.conf`.

## Start and stop procedure

1. In VMware, start `siem-host`; confirm NAT and VMnet9 connectivity.
2. If the Wazuh stack does not start automatically, SSH to `socadmin@10.10.10.10`, then run:

   ```bash
   cd ~/wazuh-docker/single-node
   sudo docker compose up -d
   sudo docker compose ps
   ```

3. Allow the Wazuh indexer and dashboard to initialize. Open `https://10.10.10.10` from the physical host. The self-signed lab certificate produces an expected browser trust warning; verify that the address is the lab server before continuing.
4. Start `ubuntu-web` and `win-client` and confirm both are **Active** in Wazuh Dashboard → Agents.
5. Verify the web page at `http://10.10.10.20`.

To stop cleanly, shut down the endpoint VMs first, then on `siem-host`:

```bash
cd ~/wazuh-docker/single-node
sudo docker compose stop
sudo shutdown -h now
```

**Never use `docker compose down -v` to perform a routine shutdown**: it deletes persistent volumes.

## Wazuh event pipeline (V2)
1. A Linux operating-system log or Windows Sysmon/Windows Event Log event is created on an endpoint.
2. Its Wazuh agent collects the event and forwards it over the host-only lab network.
3. Wazuh Manager decodes the event, evaluates rules, and creates an alert if a rule matches.
4. Wazuh Indexer stores searchable alert documents.
5. Wazuh Dashboard queries the indexed data for investigation.

**Important:** Not all collected raw events generate alerts. Normal Notepad execution required a custom alert rule in this lab.

## Verified test alerts (V1)

| Test | Rule ID | Rule description | Level | Evidence |
|---|---:|---|---:|---|
| Failed Linux authentication (verify SSH source) | 5503 | PAM: User login failed. | 5 | `evidence/05-failed-login.png` |
| Successful sudo command | 5402 | Successful sudo to ROOT executed. | 3 | `evidence/06-sudo-command.png` |
| New Linux user | 5902 | New user added to the system. | 8 | `evidence/07-new-linux-user.png` |
| Windows process creation via custom Sysmon rule | 100100 | Mini-SOC: Notepad process execution detected | 5 | `evidence/08-windows-process-creation.png` |
| Failed Windows logon (Event ID 4625) | 60122 | Logon Failure - Unknown user or bad password | 5 | `evidence/09-failed-windows-logon.png` |

The matching screenshots have been bundled under `evidence/`. The first alert confirms PAM authentication failure; inspect the original event/log source to establish the SSH-specific requirement conclusively.

## Custom Windows process alert
A custom rule with ID `100100` and level 5 was tested to trigger on Sysmon Event ID 1 for `notepad.exe`. The initial XML failed validation and was corrected before a fresh alert was found in the dashboard. Do **not** copy the full Wazuh operational Docker configuration into this public repository: it can contain credentials and private keys.

## Docker container memory measurements (W5)
One-time idle/low-activity snapshot with `sudo docker stats --no-stream`:

| Container | Memory | Memory % | CPU % |
|---|---:|---:|---:|
| wazuh.dashboard | 246.1 MiB | 1.54% | 0.13% |
| wazuh.manager | 1.366 GiB | 8.77% | 0.51% |
| wazuh.indexer | 1.724 GiB | 11.07% | 1.04% |

Total approximately 3.33 GiB at measurement time. Readings change with workload.

## Data persistence (W4)
The stack was stopped with `docker compose stop` and restarted with `docker compose start`; the existing custom-rule alert was still searchable. The host was rebooted and the three containers plus both Active agents were confirmed running again. Capture and retain a post-reboot old-alert verification screenshot for complete evidence. Persistent Docker volumes were not deleted.

## Isolation and security (S1–S2)
The `ubuntu-web` and `win-client` temporary NAT adapters were removed after package installation. Their route tables showed no default internet route; they remain connected to VMnet9. `siem-host` retains NAT and host-only adapters. Administrator password was changed from the default; the new value is not stored here.

Before publishing any new file or image:
- Review screenshots manually for visible passwords, hashes, keys, access tokens, email addresses and other private details.
- Run a local secret scan (for example `gitleaks detect --source .`) and review `git status` / `git diff --cached`. A `.gitignore` alone does not prove the absence of secrets.
- Never upload Wazuh certificates, private key files, passwords, operational volumes, or the live Docker Compose configuration containing credentials.

## Snapshots and remaining final checks
All three VMs have `w1-clean-install` and `w1-done` snapshots, verified through VMware Snapshot Manager screenshots: [win-client](evidence/18-win-client-w1-done-snapshot.png), [ubuntu-web](evidence/19-ubuntu-web-w1-done-snapshot.png), and [siem-host](evidence/20-siem-host-w1-done-snapshot.png). Still to complete: verify clocks on all three VMs; confirm the SSH origin of the failed PAM login; and capture an old-alert lookup specifically after the full host reboot. Verify the Windows evaluation expiry date if applicable.

## Repository contents
- `README.md` — lab build, operation and findings.
- `architecture/` — logical topology and IP/resource table.
- `journal.md` — troubleshooting and work journal.
- `evidence/` — actual captured verification screenshots and [evidence index](evidence/README.md).
- `.gitignore` — basic exclusions; manual review and a secret scan are also required.

## Further reading
- [Wazuh Docker deployment](https://documentation.wazuh.com/current/deployment-options/docker/index.html)
- [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Microsoft Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [SwiftOnSecurity Sysmon configuration](https://github.com/SwiftOnSecurity/sysmon-config)
