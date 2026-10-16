# Week 1 — VM Inventory and IP Plan

VMnet9 (Wazuh): Host-only `10.10.10.0/24`, DHCP disabled.

| VM | Operating system | IP (host-only) | RAM | vCPU | Disk | Function |
|---|---|---|---:|---:|---:|---|
| siem-host | Ubuntu Server 24.04 LTS | 10.10.10.10 | 16 GB | 6 | 250 GB | Wazuh single-node Docker stack; additional NAT |
| ubuntu-web | Ubuntu Server 24.04 LTS | 10.10.10.20 | 2 GB | 2 | 40 GB | Apache and Linux Wazuh agent; host-only only |
| win-client | Windows 10 | 10.10.10.30 | 4 GB | 2 | 80 GB | Windows Wazuh agent and Sysmon; host-only only |

The physical VMware host can access `https://10.10.10.10` and `http://10.10.10.20`. The endpoint VMs have no default internet route after removing their temporary NAT adapters.
