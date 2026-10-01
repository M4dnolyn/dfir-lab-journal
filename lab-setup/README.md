# Lab Setup — DFIR Analysis Workstation

**Course**: DFIR Foundations and Techniques — Blue Cape Security
**Last updated**: October 2026
**Status**: ✅ Operational

## Overview

This directory documents the end-to-end provisioning of a self-hosted digital forensics analysis environment, built as part of the Blue Cape Security DFIR Foundations and Techniques course.

The stack runs entirely on a single Linux VM (Ubuntu 24.04 LTS), keeping the footprint minimal while covering the full evidence pipeline: network traffic, memory, disk artifacts, event logs, and timeline reconstruction.

> **Note**: This repository contains no case files, course content, or investigation results. Evidence files are excluded via `.gitignore` in accordance with the Blue Cape Security license.

## Architecture

```text
Host Machine (Linux)
└── VM — Ubuntu 24.04 LTS [VirtualBox 7.2.8]
        RAM    : 8 GB
        Disk   : 80 GB (dynamically allocated)
        vCPUs  : 4
        Network: NAT  |  SSH port-forward  2222 → 22
        Role   : Primary analysis workstation (all tools)
```

A Windows VM was initially considered for native EZ Tools execution but was dropped in favor of the .NET 9 cross-platform builds, which run cleanly under Linux.

## Toolstack

| Tool | Version | Purpose |
|------|---------|---------|
| Wireshark | 4.2.2 | PCAP / network traffic analysis |
| Volatility3 | 2.28.2 | Memory image analysis |
| Plaso / log2timeline | 20260720 | Super-timeline generation |
| Timesketch | 20260630 | Timeline visualization and analysis |
| Splunk Enterprise | 10.4.3 | Log ingestion / SIEM-style querying |
| Eric Zimmerman's Tools | net9 — MFTECmd 2026.5.0 | Windows artifact parsing |
| Velociraptor | 0.77.2 | Endpoint artifact collection |
| .NET Runtime | 8.0.131 + 9.0.20 | Required for EZ Tools cross-platform builds |
| Docker + Compose | 29.8.1 / v5.5.1 | Timesketch containerization |

## Repository Structure

```text
lab-setup/
├── README.md                    ← this file
├── 01-vm-provisioning.md        ← VM creation, Guest Additions, shared folder
├── 02-docker-timesketch.md      ← Timesketch deployment via Docker
├── 03-splunk.md                 ← Splunk install and initial configuration
├── 04-eztools-dotnet9.md        ← EZ Tools on Linux (.NET 9 builds)
├── 05-velociraptor.md           ← Velociraptor binary setup
└── troubleshooting-log.md       ← Real issues encountered and how they were resolved
```

Each file is self-contained: it covers prerequisites, steps, verification commands, and observed pitfalls for that specific component.

## Starting the Lab

Commands to bring the stack up before an analysis session.

### Timesketch

```bash
cd ~/timesketch
sudo docker compose up -d
# Web UI → http://localhost
```

### Splunk

```bash
sudo -u splunk /opt/splunk/bin/splunk start
# Web UI → http://localhost:8000
```

### Volatility3

```bash
source ~/volenv/bin/activate
```

### Clean Shutdown

```bash
cd ~/timesketch && sudo docker compose stop
sudo -u splunk /opt/splunk/bin/splunk stop
deactivate
```

> **RAM note**: Timesketch (OpenSearch) and Splunk together push an 8 GB VM hard. Only start what the current task requires.

## Operational Notes

### Host Disk Space

The VM disk is dynamically allocated and grows as evidence files are extracted and timelines are generated. Monitor the host filesystem regularly:

```bash
df -h /
```

Letting the host disk fill completely will freeze the VM mid-session — see [troubleshooting-log.md](troubleshooting-log.md) for the full incident and resolution.

### Swap

A 4 GB swapfile was added to absorb OpenSearch memory spikes:

```bash
free -h   # Swap line should show 4.0 Gi
```

### `vm.max_map_count`

OpenSearch requires `vm.max_map_count ≥ 262144`. Configured permanently via `/etc/sysctl.d/99-opensearch.conf`. Verify after any reboot:

```bash
sysctl vm.max_map_count   # expected: 262144
```

### Evidence Files

Evidence is staged locally under `~/lab_case/` inside the VM and transferred from the host via a VirtualBox shared folder.

```text
~/lab_case/
├── Triage.7z              # disk triage collection
├── memdump.mem            # memory image
├── pagefile.sys           # page file (memory complement)
├── traffic.pcapng         # network capture
└── Splunk_logs_export.csv # Windows event logs (Splunk re-import format)
```

None of these files are committed to this repository.

## References

| Resource | Link |
|----------|------|
| Blue Cape Security — Course | <https://bluecapesecurity.com> |
| Timesketch | <https://timesketch.org> |
| Volatility3 | <https://volatility3.readthedocs.io> |
| Eric Zimmerman's Tools | <https://ericzimmerman.github.io> |
| Velociraptor | <https://docs.velociraptor.app> |
| Docker — Official Install | <https://docs.docker.com/engine/install/ubuntu> |
| .NET Install Script | <https://dot.net/v1/dotnet-install.sh>