# 03 — Splunk Enterprise Installation and Configuration

## Objective

Deploy Splunk Enterprise in free license mode (trial license, 500 MB/day ingestion limit) for analysis of Windows event logs exported from the investigation case.

## Prerequisites

- Operational Ubuntu 24.04 LTS VM (see [01-vm-provisioning.md](01-vm-provisioning.md))
- Splunk account created at <https://www.splunk.com> (required for download)
- Available disk space: ~4 GB for installation
- VM RAM: 8 GB — Splunk and Timesketch must not run simultaneously on an 8 GB VM (see Resource Considerations)

## 1. Why Splunk Free License

Splunk Enterprise offers a free license with no time limit, capped at 500 MB of data ingestion per day. This ceiling is more than sufficient for analyzing log exports from a single forensic case.

The free version differs from the paid version in two non-critical aspects for this lab: no scheduled alerts and no multi-user access control. These features are unnecessary in a single-user investigation context.

## 2. Download

The installation package is not available via apt or any official third-party repository. Download is manual from the Splunk website after authentication.

1. Open Firefox in the VM
2. Navigate to <https://www.splunk.com/en_us/download/splunk-enterprise.html>
3. Sign in with Splunk account (free registration if needed)
4. Select: **Linux → .deb → AMD64**
5. Download the file (~1.5 GB)

The downloaded file lands in `~/Téléchargements/` (or `~/Downloads/` depending on system locale).

## 3. Installation

### 3.1 Install the Package

```bash
cd ~/Téléchargements
sudo dpkg -i splunk-*.deb
```

### 3.2 Create Dedicated System User

Recent versions of Splunk Enterprise refuse to start as root. This is a deliberate Splunk decision to limit attack surface in production. The solution is to create a dedicated system user and transfer ownership of Splunk files:

```bash
sudo useradd -m -r -s /bin/bash splunk
sudo chown -R splunk:splunk /opt/splunk
```

### 3.3 First Start

```bash
sudo -u splunk /opt/splunk/bin/splunk start --accept-license
```

On first start, Splunk prompts to set an admin username and password. These credentials are independent of the system `splunk` user — they are used only for web interface authentication.

Full startup takes 1–2 minutes. Expected completion message:

```
Waiting for web server at http://127.0.0.1:8000 to be available... Done
```

### 3.4 Enable Auto-Start (Optional)

**Not enabled in this lab.**

Splunk can be configured to start automatically with the VM:

```bash
sudo /opt/splunk/bin/splunk enable boot-start -user splunk
```

This was not enabled here: Splunk and Timesketch must not run simultaneously on an 8 GB VM. Manual start allows precise control over which services are active based on current analysis needs.

## 4. Web Interface Access

Open Firefox in the VM:

```
http://localhost:8000
```

Sign in with credentials set during first start (§3.3).

## 5. Daily Management Commands

### Start Splunk

```bash
sudo -u splunk /opt/splunk/bin/splunk start
```

### Stop Splunk

```bash
sudo -u splunk /opt/splunk/bin/splunk stop
```

### Check Status

```bash
sudo -u splunk /opt/splunk/bin/splunk status
```

## 6. Case Log Import (Future Reference)

Data import will be performed during the analysis phase per course instructions. Documented here for reference.

### Manual Import via Web Interface

1. **Settings → Add Data → Upload**
2. Select `Splunk_logs_export.csv`
3. Follow wizard: define sourcetype, target index, and timestamp parameters

### Command-Line Import

```bash
sudo -u splunk /opt/splunk/bin/splunk add oneshot \
  ~/lab_case/Splunk_logs_export.csv \
  -index main \
  -sourcetype csv
```

> ⚠️ Do not import data before the corresponding analysis phase of the course requests it — imported data persists between sessions and could bias subsequent investigations if imported out of order.

## 7. Resource Considerations

Splunk consumes 1–1.5 GB RAM in normal operation, plus OS and active analysis tools overhead.

**Cohabitation rules on 8 GB VM:**

| Scenario | Feasible |
|----------|----------|
| Splunk alone | ✅ Comfortable |
| Timesketch alone | ✅ Comfortable |
| Splunk + Timesketch simultaneously | ⚠️ Limit — avoid if possible |
| Splunk + Timesketch + Volatility3 | ❌ Likely memory saturation |

### Recommended Session Start Sequence

```bash
# If session requires Splunk only
cd ~/timesketch && sudo docker compose stop
sudo -u splunk /opt/splunk/bin/splunk start

# If session requires Timesketch only
sudo -u splunk /opt/splunk/bin/splunk stop
cd ~/timesketch && sudo docker compose up -d
```

## 8. Issues Encountered and Resolutions

### Splunk Refuses to Start as Root

**Symptom**:

```
Running Splunk Enterprise as root is deprecated and will be removed
in a future release.
```

Command returns without starting Splunk.

**Cause**: Since Splunk Enterprise 9.x, starting as root is explicitly blocked. This is not a bug but an intentional security decision by Splunk.

**Resolution**: Create dedicated system user and transfer file ownership before any start (see §3.2).

## ✅ Expected Final Configuration State

- [x] Splunk Enterprise installed in `/opt/splunk/`
- [x] System user "splunk" created, owner of `/opt/splunk/`
- [x] Web interface accessible at `http://localhost:8000`
- [x] Admin credentials set and functional
- [x] Manual start only (controlled cohabitation with Timesketch)