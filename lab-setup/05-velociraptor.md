# 05 — Velociraptor

## Objective

Install Velociraptor, the open-source endpoint artifact collection and incident response platform, for local collection use within this forensic lab.

## Prerequisites

- Operational Ubuntu 24.04 LTS VM (see [01-vm-provisioning.md](01-vm-provisioning.md))
- Internet access from the VM

## 1. Context and Scope of Use

Velociraptor operates in two primary modes:

| Mode | Description | Availability in This Lab |
|------|-------------|--------------------------|
| Client/Server | Agent deployment on endpoints, centralized collection, real-time hunts | ❌ Not available — requires live endpoint access |
| Local Collection | Direct execution on a system to collect artifacts or query offline files | ✅ Available |

In a self-hosted lab working with pre-collected evidence (triage, memory dump), only local collection mode is relevant. Real-time hunts are unavailable without active client/server infrastructure.

## 2. Installation

Velociraptor is distributed as a standalone static binary with no system dependencies. Installation reduces to download and executable permission.

### 2.1 Download Latest Stable Version

The binary filename includes the version number, which changes each release. Rather than hardcoding the version, the URL is resolved dynamically via the GitHub Releases API:

```bash
cd ~
URL=$(curl -s https://api.github.com/repos/Velocidex/velociraptor/releases/latest \
  | grep browser_download_url \
  | grep 'linux-amd64"' \
  | grep -v musl \
  | head -1 \
  | cut -d '"' -f 4)

wget "$URL" -O velociraptor
chmod +x velociraptor
```

> The `grep -v musl` filter excludes the musl-libc build, intended for Alpine/BusyBox distributions. Ubuntu uses glibc — the standard `linux-amd64` build is correct.

### 2.2 Verification

```bash
./velociraptor version
```

Expected output:

```
name: velociraptor
version: 0.77.x
compiler: go1.25.x
system: linux
architecture: amd64
```

## 3. Local Collection Mode Usage

Velociraptor can run without a server to execute VQL (Velociraptor Query Language) artifacts directly on the system or on collected artifact files.

### List Available Artifacts

```bash
./velociraptor artifacts list
```

### Collect a Specific Artifact

```bash
./velociraptor artifacts collect <ArtifactName> --output ~/output/
```

### Query Offline Files

```bash
./velociraptor query "SELECT * FROM parse_evtx(filename='path/to/file.evtx')"
```

> Practical application of these commands on case files will be performed during corresponding analysis phases of the course.

## 4. Issues Encountered

No significant issues encountered during installation.

The dynamic URL resolution via GitHub API (§2.1) is intentional: a hardcoded version number becomes obsolete at each new release and silently yields a 404 error if the binary is not found. Dynamic resolution guarantees the downloaded binary is always the latest stable release.

## ✅ Expected Final Configuration State

- [x] `velociraptor` binary present in `~/velociraptor`
- [x] Execute permissions granted
- [x] `./velociraptor version` returns expected version and architecture